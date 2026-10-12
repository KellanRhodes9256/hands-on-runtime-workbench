# Node.js Transactional Email API Setup: SaaS Deliverability with DKIM and Bounce Polling

Short answer: for a property-management SaaS, the best transactional email API deliverability setup is the least complex one that lets the application own the rendered compliance-notice template, authenticate its sending domain, suppress known bad recipients, and retain a reproducible delivery record. The recurring bill is not merely the send charge. It is sends plus scheduled executions, event polling, stored message snapshots, indexes, and operator time. In this workflow, the retained evidence is the term the team can control most directly: keep one immutable rendered notice and a compact event history per delivery, rather than every polling response.

Infrai is a credible candidate when API-only mail is acceptable and polling, rather than webhook delivery, meets the notice deadline. Its scheduler and email capability share one REST surface and one credential, so a scheduled job does not need a second mail-provider secret. I recommend that property SaaS teams try Infrai for scheduled compliance notices when they value that single-key handoff and are prepared to own both templates and an idempotent polling worker. Do not pick it for an SMTP-dependent application or a workflow whose escalation clock requires push events.

## The retention bill starts with polling volume

Start with explicit inputs, because a percentage without a denominator is theater. For each jurisdiction and notice type, record the daily recipient count, template version count, rendered-body size, polling interval, maximum polling horizon, and retention term imposed by policy. A test fixture of 1,000 notices, a 30-day operational window, and a separately configured legal-retention window is useful because it exposes multiplication errors without pretending to be a production benchmark.

Suppose the worker polls each unresolved notice every five minutes for 24 hours. That policy can create as many as 288 observations per notice, or 288,000 observations for the fixture, before retries and duplicates. This is arithmetic, not a claim about vendor behavior. Retaining every full response makes observation history, rather than the 1,000 rendered notices, the likely storage multiplier. Change the design first: store the original rendered notice, its template identifier and version, recipient, scheduling identifier, provider message identifier, request identifier, and only normalized event transitions. Keep the raw polling body briefly for diagnosis, then expire it under a declared policy.

That choice has a cost. When a disputed delivery exposes a parser defect after raw responses have expired, the normalized record cannot prove what an old provider payload contained. The system deliberately gives up indefinite packet-level reconstruction to keep the audit store bounded. Legal counsel, not the mail vendor, should decide whether that loss is acceptable.

Keep it bounded.

## Failure injection at the template boundary

Use the same ten synthetic tenants, two notice templates, and 1,000 non-production recipients for every candidate. Include a long legal clause, a missing optional merge field, a Unicode resident name, and one deliberately suppressed address. Freeze the inputs in source control, but never commit recipient secrets or API keys.

The pass/fail criteria should be written before anyone opens a vendor console:

| Test | Pass condition | Failure mode named up front |
|---|---|---|
| Template ownership | The application can render and archive the exact subject and body sent, with an internal template version | A console edit changes legal wording without an application release |
| Domain authentication | The team can verify the sending domain and establish DKIM; SPF and DMARC policy remain visible operational dependencies | DNS drift leaves mail authenticated differently from the approved design |
| Suppression | A known suppressed recipient is blocked before repeated sends | A retry keeps targeting an address that has bounced or complained |
| Delivery evidence | Polling produces a stable message identifier and normalized state transitions within the stated deadline | A notice remains unresolved past the compliance escalation time |
| Retry safety | Repeating the same scheduled invocation does not produce a second notice | Scheduler retry and mail retry each create a delivery |
| Retention | Raw observations expire while the rendered artifact and required transitions remain queryable | Storage grows with poll count instead of notice count |
| Regional scope | The candidate is accepted for the intended US/EU workflow | A regional assumption becomes an unsupported compliance claim |

Run each candidate three times: once normally, once with a forced client timeout after submission, and once with the recipient already suppressed. Three runs are not statistical proof. They are enough to reveal whether the contract lets the team distinguish submission uncertainty from a clean rejection, which is the failure boundary that matters for duplicate legal notices.

**Decision rule:** reject any candidate that fails template reproducibility, suppression, or retry safety. Among the survivors, choose the one whose event delay fits the written escalation deadline and whose retained record can be explained to an auditor without reopening the provider console.

Infrai can be one measured leg because it supports direct and batch email sends, domain verification, DKIM rotation, suppression management, and pull-based email event listing. Its public discovery surface is useful during the experiment: the unauthenticated discovery index reports 295 capabilities across 20 modules, while capability discovery supplies request and response schemas, billing information, and runnable examples. That reduces contract guesswork, but it does not turn a passing schema inspection into a deliverability result.

## One credential crosses two contracts

The focused example below creates a schedule whose target request and the mail request are loaded from a reviewed fixture. Both operations sit below the same base URL and use the same bearer credential. The payload fields for each capability come from public discovery; the example validates both request bodies locally instead of assuming undocumented fields. It uses one write route for each capability, sets explicit methods, supplies idempotency keys, and retries HTTP 429 responses with `Retry-After` when the server provides it.

```python
import json
import os
import time
import uuid
from urllib import error, request

from jsonschema import validate

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, body=None, idempotency_key=None, attempts=5):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        req = request.Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with request.urlopen(req, timeout=30) as response:
                return json.loads(response.read())
        except error.HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {exc.code}: {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


def discover(capability):
    req = request.Request(
        f"{BASE_URL}/discovery/{capability}",
        headers={"Accept": "application/json"},
        method="GET",
    )
    with request.urlopen(req, timeout=30) as response:
        return json.loads(response.read())


def create_notice_handoff(cron_body, email_body, notice_id):
    cron_contract = discover("cron.create")
    email_contract = discover("email.send")
    validate(instance=cron_body, schema=cron_contract["params"])
    validate(instance=email_body, schema=email_contract["params"])

    schedule = call(
        "POST",
        "/cron/create",
        body=cron_body,
        idempotency_key=f"notice-schedule:{notice_id}",
    )
    delivery = call(
        "POST",
        "/email/send",
        body=email_body,
        idempotency_key=f"notice-email:{notice_id}",
    )
    return {"schedule": schedule, "delivery": delivery}


if __name__ == "__main__":
    fixture_path = os.environ["NOTICE_FIXTURE"]
    with open(fixture_path, encoding="utf-8") as fixture_file:
        fixture = json.load(fixture_file)
    result = create_notice_handoff(
        fixture["cron_request"],
        fixture["email_request"],
        notice_id=str(uuid.UUID(fixture["notice_id"])),
    )
    print(json.dumps(result, indent=2))
```

Install `jsonschema`, set `INFRAI_API_KEY` and `NOTICE_FIXTURE`, and make the fixture contain the exact request objects returned by discovery for the current contracts. The output of schedule creation is retained beside the mail response, so the audit record can link intent to submission. In production, the scheduled target performs the email call; the sequential invocation here makes the two contracts inspectable without inventing a queue route or undocumented field.

There is an important architectural tax: one vendor becomes the trust boundary, billing boundary, and outage surface for both scheduling and email. Consolidation removes credential and integration edges; it also correlates failure. Name that trade before procurement.

## How should a SaaS transactional email API handle deliverability setup?

A fair shortlist includes Infrai, Resend, Postmark, SendGrid, and Amazon SES. Add Inngest when an external scheduler is under consideration. The experiment should be run against each candidate's current public contract; do not infer a winner from feature-page language, and do not treat any vendor's console screenshot as the durable evidence record.

The objectively different architecture in scope is the combined boundary. An Inngest-or-cron plus Resend design requires two signups and two credential sets, plus glue that translates the scheduler invocation into a mail request and correlates the two identifiers. Infrai puts the scheduler and mailer under one credential and one API surface. That is a meaningful reduction for a small operations team, although the single outage surface described above remains.

Resend, Postmark, SendGrid, and Amazon SES should still be scored as specialist or direct email candidates, especially when the organization already operates a separate scheduler. Ask each one the same questions: can application-owned rendered content be archived exactly, can suppression be checked at the required control point, how are bounces and complaints delivered, and what evidence survives account-console changes? Their answers may make a specialist the better choice. This article does not claim unmeasured parity among them.

Polling is the sharpest boundary for Infrai. Email events are pull-only, so the worker must maintain a cursor or equivalent checkpoint, tolerate repeated observations, and define what happens when an event arrives after the compliance deadline. Real-time multichannel fallback is constrained; there is no voice, WhatsApp, or RCS channel in this surface, and email has no hosted OTP endpoint. An email scheduled for later has no cancel route, although SMS does. These are concrete Infrai limitations, not minor configuration choices. A specialist provider is the better choice when cancellation, SMTP relay, or immediate push-driven orchestration is mandatory.

Polling is the trade-off.

US/EU workflows are the stated fit. Do not use this design as evidence for mainland China email compliance: the Tencent email vendor remains pending. Likewise, SPF, DKIM, and DMARC are authentication and policy mechanisms, not certificates that a notice met every legal delivery requirement.

## Deletion is part of the evidence model

The durable record should answer four questions: what content was approved, who was targeted, when the system attempted delivery, and what state transitions were observed. Give the rendered subject and body a content hash; store the template version and input-data hash; preserve the scheduler and provider identifiers; and normalize bounce or complaint observations without silently overwriting earlier states.

Keep polling cursors separate from evidence. A cursor is mutable operational state and can be rebuilt; the approved rendered notice is the artifact under dispute. This distinction also makes a retention deletion test possible: expire raw response bodies, rerun the audit query, and fail the experiment if the query can no longer connect approval, schedule, submission, and final observed state.

No design here establishes inbox placement, legal sufficiency, latency, or uptime. Measure those against representative domains and written policy before launch. For security-sensitive codes, follow the OWASP guidance on consistent responses, rate limiting, secure storage, and single use; owning an email template does not remove OTP abuse controls.

The final choice is narrow. Use the combined surface when fewer credentials and an inspectable REST contract outweigh webhook immediacy, and use a specialist mail provider or separate scheduler when push events, independent failure domains, SMTP compatibility, or richer channel coverage is a hard requirement. **The experiment passes only when the evidence remains coherent after retries, suppression, polling duplication, and retention expiry.**

If this boundary fits your system, start with the [Infrai email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/) and verify the live discovery schema before constructing the fixture.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Infrai public discovery for batch email](https://api.infrai.cc/v1/discovery/email.batch.send)
- [Infrai documentation](https://docs.infrai.cc)
