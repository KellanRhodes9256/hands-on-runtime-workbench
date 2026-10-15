# Malformed Password Reset Email Templates: How to Catch 3 Missing Variables

Do not let a password reset email reach the send path until its template, variable contract, and rendered links have passed a preview test. **TL;DR:** for a media marketplace, keep `reset_link` and `user_name` in a versioned contract owned by the account service, render representative seller data before deployment, and reject a malformed payload locally; the provider API should be the last boundary, not the first validator.

This is an architecture decision about template ownership, not transport fashion. A plain REST API such as Infrai is a reasonable fit when a team does not want an SDK or client-library version in the account service, and its template create, update, and preview flow gives that team a place to catch malformed HTML and placeholder mismatches before sending. The API is genuinely self-describing, and the discovery surface is public with no key required; it reports the current JSON Schema, so the adapter can be checked against the contract instead of an old client library. Every documented capability ships runnable examples in 10 languages. It has no SMTP relay, so repairing the API payload or template is the direct path when a request is rejected. No transport swap will restore a missing `reset_link`.

Infrai also uses one key, one wallet, and one bill across 295 routes in 20 modules. For this reset workflow, that can keep a later SMS fallback inside the same credential and reconciliation boundary instead of adding another secret and vendor invoice.

The decision below assumes a password reset for a marketplace seller account in a media product. The link is security-sensitive, delivery is transactional, and the email must remain understandable even when a seller name is absent or unusually long.

## How should a malformed password reset email template handle missing variables?

Three invariants matter. First, the template's required names and the backend payload's names must be identical; `resetUrl` is not a tolerable approximation of `reset_link`. Second, preview must render a nonempty, absolute HTTPS reset URL without leaving placeholder syntax in either the HTML or plain-text body. Third, a rejected preview or send must fail closed: log the provider request identifier when one exists, but never log the reset token itself.

Fail closed.

The failure boundaries are deliberately narrow. The account service owns variable construction and token lifetime. The template system owns interpolation and presentation. The email provider owns acceptance and delivery, but provider acceptance cannot prove that a blue button points to the intended account. That distinction is easy to miss because a `2xx` response feels final. It isn't.

Treat template versions like stored object versions: immutable releases are easier to reason about than mutable content with an ambiguous deployment time. A template update should be previewed against a fixed fixture before its identifier becomes active, while the previous known-good identifier remains available for rollback. This is a consistency choice. It prevents application code and template content from moving independently through production with incompatible schemas.

## Record the ownership decision

The useful comparison is not which logo has the longest feature page. It is where the authoritative template lives, how the backend detects contract drift, and which operational boundary the team is willing to own.

| Option | Template ownership | Strong fit | Boundary to verify |
|---|---|---|---|
| Amazon SES | Provider-side or application-side, by team decision | AWS-centered systems that already operate their email configuration there | Confirm the chosen template and payload contract in the SES integration tests |
| Twilio SendGrid | Provider-side or application-side, by team decision | Teams that want a dedicated email product and can accept that vendor boundary | Confirm dynamic variables against a staged template before changing the active version |
| Postmark | Provider-side or application-side, by team decision | Transactional-email workflows that favor a focused email service | Confirm that layout changes preserve the reset-link contract |
| Infrai | Provider template with an application-owned variable contract | Services that prefer one plain REST interface and key without installing another SDK | Preview before send; there is no SMTP relay to use as an alternate path |

Those rows are not claims of equivalent implementation. They are four legitimate ownership choices. Amazon SES, SendGrid, and Postmark deserve direct evaluation through their current official documentation because their template models and operational controls can change. The REST option is attractive when reducing client-library surface is a real constraint, although its broader capability catalog is irrelevant if email is the only service the team needs.

The limitation is concrete: **this option is not suitable when SMTP compatibility or webhook-driven orchestration is mandatory.** Choose an email provider whose documented interfaces meet that requirement. Its email and SMS namespaces use pull-based events, and it lacks SMTP relay, voice, WhatsApp, and RCS channels. That trade-off carries more architectural weight than the convenience of one credential.

There are more limits around fallback design. Infrai has a hosted SMS OTP capability but no hosted email OTP counterpart, so an email-code fallback remains application work. Scheduled email has no cancellation route, although SMS does. The pending Tencent email vendor must not be treated as evidence for mainland-China compliance, and real-time multichannel orchestration cannot be based on webhooks that are not present.

## Test the contract as executable code

This Python program validates the local contract, then calls the real provider preview route. It catches the three failures that should never consume a send attempt: a missing required variable, an unresolved placeholder, and an invalid reset URL. The request JSON remains an input file because its exact fields must come from the current discovery schema; hard-coding an undocumented shape in an architecture note would replace one template mismatch with another. Set `INFRAI_BASE_URL` to the documented v1 API base, `INFRAI_API_KEY` to a secret, `INFRAI_TEMPLATE_ID` to the created template, and `INFRAI_PREVIEW_JSON` to that schema-validated fixture, then run the file with Python 3.

```python
import hashlib
import json
import os
import time
from email.utils import parsedate_to_datetime
from html import escape
from pathlib import Path
from string import Formatter
from urllib.error import HTTPError
from urllib.parse import quote, urlparse
from urllib.request import Request, urlopen


REQUIRED = frozenset({"user_name", "reset_link"})


def placeholder_names(template: str) -> set[str]:
    return {
        field_name
        for _, field_name, _, _ in Formatter().parse(template)
        if field_name is not None
    }


def validate_and_render(template: str, variables: dict[str, str]) -> str:
    names = placeholder_names(template)
    missing = REQUIRED - variables.keys()
    unexpected_contract = names ^ REQUIRED

    if missing:
        raise ValueError(f"missing variables: {sorted(missing)}")
    if unexpected_contract:
        raise ValueError(
            f"template contract differs from required fields: "
            f"{sorted(unexpected_contract)}"
        )

    parsed = urlparse(variables["reset_link"])
    if parsed.scheme != "https" or not parsed.netloc:
        raise ValueError("reset_link must be an absolute HTTPS URL")

    safe_variables = {name: escape(value, quote=True) for name, value in variables.items()}
    rendered = template.format_map(safe_variables)
    if placeholder_names(rendered):
        raise ValueError("rendered output still contains placeholders")
    return rendered


def retry_delay(response_headers, attempt: int) -> float:
    value = response_headers.get("Retry-After")
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, retry_at.timestamp() - time.time())


def preview_template(payload: bytes) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    template_id = quote(os.environ["INFRAI_TEMPLATE_ID"], safe="")
    url = f"{base_url}/email/template/preview/{template_id}"
    idempotency_key = hashlib.sha256(payload).hexdigest()

    for attempt in range(5):
        request = Request(
            url,
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"preview failed ({error.code}): {body}") from error
            time.sleep(retry_delay(error.headers, attempt))

    raise RuntimeError("preview retry loop ended unexpectedly")


def main() -> None:
    template = (
        "<p>Hello {user_name},</p>"
        "<p><a href=\"{reset_link}\">Reset your marketplace password</a></p>"
    )
    fixture = {
        "user_name": "Morgan Seller",
        "reset_link": "https://accounts.example.test/reset?token=fixture-token",
    }
    preview = validate_and_render(template, fixture)
    assert "{reset_link}" not in preview
    assert "https://accounts.example.test/reset" in preview
    payload_path = Path(os.environ["INFRAI_PREVIEW_JSON"])
    payload = payload_path.read_bytes()
    json.loads(payload)
    print(json.dumps(preview_template(payload), indent=2))


if __name__ == "__main__":
    main()
```

Escaping is part of the contract here, not cosmetic cleanup. The fixture uses a reserved `.test` host and a fake token so it cannot become a live credential if output is retained in continuous-integration logs. The API payload file should contain the equivalent nonproduction values in the current request shape. The program uses an explicit method, Bearer authentication, a deterministic idempotency key, a 15-second timeout, bounded exponential backoff, `Retry-After`, and response-body error reporting. It does not infer success from the absence of an exception; it parses the successful JSON response, while every non-`429` HTTP error stops immediately with the provider body attached.

Preview first.

For this adapter, that means creating the template, exercising `POST /v1/email/template/preview/{id}`, and correcting either the template or application payload before calling the send operation. Keep the API-facing code small. The public discovery surface can supply the current request JSON Schema and runnable Python example rather than forcing the adapter to guess fields.

## Promote the preview, then observe the boundary

The deployment gate is short: run the local contract test, run the provider preview using a seller fixture, inspect both HTML and plain text, and promote the tested template identifier. Keep the preview artifact free of real reset tokens. On rejection, compare required fields and exact variable names before investigating delivery because a malformed request never crossed that boundary.

After acceptance, poll the available event or message-status surface according to the selected provider's documented model. Do not design this workflow around a webhook from the REST option: neither relevant namespace supplies webhook event delivery. Polling constrains orchestration latency, so a reset screen should give the seller an honest resend path rather than imply instantaneous cross-channel state.

This separation also makes diagnostics less theatrical. A template mismatch is a deterministic build or promotion failure. A rejected API payload is an adapter failure. A later delivery outcome belongs to provider operations. Each has a different owner, and none requires exposing a reset token in logs.

## Why reject an application-only template?

Embedding the entire email in application code is the rejected default for this decision because it couples copy and HTML deployment to the account service, while still leaving the team responsible for rendering tests and delivery integration. It also makes a non-code layout correction wait for a service release. For the media marketplace described here, a versioned provider template plus an application-owned variable contract is the cleaner boundary.

Application-owned templates remain valid when repository review is mandatory, the organization requires reproducible rendering without a provider control plane, or SMTP portability is a hard requirement. In that case, test the same invariants in the repository and choose a transport that actually supports the required interface. Do not pretend a REST-only provider supplies SMTP portability.

The final rule is severe but useful: **no successful preview, no send**. Preview cannot establish inbox delivery, but it can stop the avoidable class of password reset failures caused by malformed HTML, missing placeholders, and structurally invalid links before a seller depends on the message.

## References

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid template documentation](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark template documentation](https://postmarkapp.com/developer/user-guide/templates/overview)
- [Python `string` documentation](https://docs.python.org/3/library/string.html)
- [WHATWG URL Standard](https://url.spec.whatwg.org/)
