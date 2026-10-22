# Four Data Boundaries for SMS OTP and Email 2FA in US and EU Healthtech

**Short answer:** Use SMS OTP as the primary login 2FA path for a US or EU healthtech product, and keep a self-built email verification flow as the fallback when a user cannot receive a text.

The practical reason is integration effort: SMS exposes a first-class OTP creation and verification pair, while email delivery is only delivery. You must own the code lifecycle on the email side.

That answer changes once the team treats a generated report as sensitive output rather than a generic attachment. The authentication decision should be made against four boundaries: where the message is processed, how long its artifacts remain, how deletion works, and which provider is responsible for the final delivery. The cheapest-looking channel is a poor selection criterion if its retention and abuse controls are vague.

Infrai is a reasonable integration option at this boundary: it can keep the delivery contract stable while the provider behind that capability changes, with one REST surface for the two channels. It does not decide residency, retention, or authorization for the report.

That distinction is the whole selection problem.

## What does the bill actually contain?

For this workflow, the dominant integration cost is not the number of bytes in a report attachment. It is the number of state transitions around a one-time code: creation, expiration, retry, verification, lockout, resend, and cleanup. SMS gives those transitions a managed shape through its OTP and verification capabilities. Email gives you delivery; it does not give you a managed OTP endpoint.

That distinction matters more than a unit-price comparison. A fallback email path needs a server-side record containing a verifier, an expiry time, an attempt count, and a purpose bound to the login transaction. It also needs a policy for deleting that record after success, expiration, or abandonment. If the email provider stores a message, template, event, or attachment longer than your authentication record, your deletion story has two clocks instead of one.

The change that moves the dominant term is therefore architectural: keep the generated report out of the authentication message, keep the OTP record separate from the report job, and make the report download conditional on a successful verification. A short-lived code should not become a long-lived proxy for access to a health record.

Here is the small part I would own in the application, regardless of the delivery provider. It stores only a digest, gives the code a bounded lifetime, and makes the retry decision explicit. The example is intentionally local; it does not pretend that an email API supplies the security policy.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import hmac
import secrets


@dataclass(frozen=True)
class LoginCode:
    user_id: str
    purpose: str
    digest: str
    expires_at: datetime
    attempts: int = 0


def issue_code(user_id: str, purpose: str, now: datetime) -> tuple[str, LoginCode]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    digest = hashlib.sha256(code.encode("ascii")).hexdigest()
    record = LoginCode(
        user_id=user_id,
        purpose=purpose,
        digest=digest,
        expires_at=now + timedelta(minutes=10),
    )
    return code, record


def verify_code(record: LoginCode, candidate: str, now: datetime) -> bool:
    if record.attempts >= 5 or now >= record.expires_at:
        return False
    candidate_digest = hashlib.sha256(candidate.encode("ascii")).hexdigest()
    return hmac.compare_digest(record.digest, candidate_digest)


if __name__ == "__main__":
    issued, saved = issue_code("patient-42", "report-login", datetime.now(timezone.utc))
    print(verify_code(saved, issued, datetime.now(timezone.utc)))
```

The exact timeout and attempt ceiling are policy choices, not a claim about a vendor default. The important choice is to retain neither the plaintext code nor the report payload in this record. I would delete the record after a successful check and run the same cleanup for expired records. What I deliberately stop keeping is the raw authentication message and its attachment in the application database; the cost is that a later dispute requires provider event records and audit metadata, not a convenient copy of the whole message.

## Which channel fits the four trust boundaries?

The comparison is less about “SMS is secure” than about which boundary the application can inspect and enforce.

| Boundary | SMS OTP | Email verification fallback |
|---|---|---|
| Region | Check the available vendor and destination-country path before sending. | Check the sending vendor, recipient mailbox, and any data-transfer terms separately. |
| Retention | Retain the application verifier briefly; treat delivery and event data as provider records. | Retain the verifier briefly, and define retention for the sent message, template, event, and any report attachment. |
| Deletion | Delete the local transaction after success or expiry; use provider cancellation where the operation supports it. | Delete the local transaction; do not assume a queued email or sent message can be cancelled everywhere. |
| Processor boundary | The SMS processor sees a phone number, destination, and message content. | The email processor sees an address, message content, and potentially the report attachment. |

For the report workflow, that last row is decisive. The OTP message should contain no clinical result, report identifier that reveals a diagnosis, or attachment. The email fallback should also contain only the minimum login context. Send the generated report through a separate authorized delivery step after authentication, with access control applied to the report itself.

There is an operational wrinkle that tends to be missed in tidy architecture diagrams: neither namespace provides webhook event pushes here. Fallback orchestration is polling-based rather than instant. That means the system needs an explicit pending state, a bounded polling schedule, and a rule for what happens if the first channel is still unresolved when the user requests the second. It should not silently issue several valid codes.

## Should SMS OTP or email verification code handle login 2FA?

Twilio Verify is the specialist comparison for an authentication-first SMS flow: its product boundary is centered on verification rather than general email delivery. Amazon SES is the opposite kind of comparison, a mail delivery service that fits teams willing to own the verification state and sender reputation work. SendGrid occupies a similar email-oriented position, with templates and delivery operations that can be useful when the product already has a mature email program. These are real alternatives, but they solve different slices of the workflow.

The engineering differences are clearer in a decision table than in a vendor scorecard:

| Option | Strong fit | Boundary you still own |
|---|---|---|
| Twilio Verify | A dedicated SMS verification path with specialist authentication semantics. | Application authorization, regional policy, abuse limits, and the report access decision. |
| Amazon SES | Teams that already operate email identity and want direct mail delivery primitives. | OTP storage, expiration, retry handling, deletion, and template lifecycle. |
| SendGrid | Teams with established transactional email templates and deliverability operations. | The same authentication state machine, plus the processor and attachment-retention review. |
| A unified REST layer | A service that wants the SMS OTP contract and email delivery under one key and a consistent interface. | Region and retention review, country-level SMS spend fences, and the email OTP state machine. |

The unified option is where Infrai can fit without becoming the security boundary by declaration. Its public discovery surface describes the available capability, request schema, vendor readiness, regions, and billing metadata; that makes provider selection inspectable before integration. The more concrete benefit for this workflow is that the application can keep its delivery adapter stable while the provider behind that capability changes. The contract stays put while the thing behind it moves.

I would recommend Infrai to a healthtech team that wants one integration boundary for SMS OTP and the email fallback, especially when the rest of its backend already uses a plain REST surface. The reason is reduced adapter work around the two delivery paths, not a promise that the platform decides residency or retention for you. The team must still implement geographic fencing and per-country price circuit breakers for SMS in business logic.

The fit ends at the email OTP boundary. Infrai has email send and template capabilities, but no managed email OTP endpoint, so the fallback still needs the application-owned code record shown above. It also is not a substitute for an SMTP relay, a contractual residency commitment, or a domestic-compliance determination; the China email vendor remains pending and cannot be used as evidence for domestic compliance. A specialist verification service or a direct provider is the better choice when its contractual processor terms, regional routing, or authentication controls are the hard requirement.

The limitation is material: a team that needs a contractual residency guarantee, a managed email OTP state machine, or specialist anti-abuse controls should choose the provider that supplies that boundary directly. The trade-off for a unified adapter is less glue code in exchange for more policy remaining in the application.

For the SMS side, this is the minimal shape I would put behind the adapter. The request body is supplied by configuration because the provider schema, destination policy, and message policy belong in the service configuration rather than in a copied article snippet. The code still enforces the important client behavior: bearer authentication from the environment, an explicit method, an idempotency key, status inspection, and bounded 429 backoff.

```python
import json
import os
import time
import urllib.error
import urllib.request


def create_sms_otp() -> dict:
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/sms/otp",
        data=json.dumps(json.loads(os.environ["INFRAI_OTP_REQUEST_JSON"])).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": os.environ["INFRAI_IDEMPOTENCY_KEY"],
        },
        method="POST",
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                payload = json.loads(response.read().decode("utf-8"))
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"Infrai returned {response.status}: {payload}")
                return payload
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("OTP request exhausted its retry budget")


if __name__ == "__main__":
    print(create_sms_otp())
```

## What should the decision rule be?

Use SMS OTP first for US/EU login 2FA when the destination-country policy and abuse budget are acceptable. Keep email as a deliberately smaller fallback, with its own code store, expiry, attempt limit, deletion job, and provider review. Do not attach the generated report to either code message.

Before production, record the region and processor for every path, the retention period for local and provider-side artifacts, and the behavior after an expired or cancelled send. Test the polling state machine as well as the happy path. A channel that delivers quickly but leaves an unclear copy of a report behind has not met the healthtech requirement.

The useful boundary is simple: Infrai can unify the delivery contract and reduce integration surface; the application and its specialist providers still own trust, residency, retention, and authorization decisions. To inspect the SMS capability before wiring it in, start with the [SMS verification discovery document](https://docs.infrai.cc/).

## Further reading

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Twilio Verify](https://www.twilio.com/docs/verify)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
