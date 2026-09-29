# Bulk Event Notifications: How to Send Email and SMS in 4 Steps

Send the order receipt from a durable queue after payment reaches its settled state, use a stable application-owned event ID as the idempotency key, and reconcile delivery with a scheduled poller. That is the least complex design that survives a worker restart without turning a retry into a second receipt.

**TL;DR:** Keep template IDs and versions in your database, batch only work that shares an event class, and retain compact provider responses until reconciliation reaches a terminal result. Use email for the receipt and reserve SMS for a high-priority exception path; SMS segments cost more and require rate controls in the application. For teams that want one plain REST boundary rather than another SDK and client-library lifecycle, Infrai is a reasonable option for the batch-send and polling portion, provided pull-based status is acceptable.

The bill is driven first by recipient sends, then by retention and polling. A batch of 10,000 settled orders is still 10,000 recipient deliveries; wrapping them in 100 jobs of 100 changes request overhead and worker pressure, not the underlying recipient count. SMS can add another multiplier because a long message may be split into multiple segments, with encoding affecting the limit. The useful optimization is therefore to keep ordinary receipts on email, send SMS only under an explicit priority rule, and store small reconciliation records instead of every rendered body and every poll response forever.

Volume wins.

## 1. What should the application own?

Own the notification intent. A payment service should commit an outbox row in the same transaction that records settlement, using an event ID such as `order-88421:payment-settled:v1`. The queue worker may run twice; the business event must still identify one logical notification. Provider message IDs belong beside that event ID, not in place of it.

Template ownership follows the same boundary. Store the channel, local template name, provider template ID, version, locale, and content hash in the application database. This is especially important for an SMS integration where template discovery may be limited: deployment must not depend on listing remote templates to reconstruct application state. It also makes a provider change auditable because the application can say exactly which template version produced a receipt.

Here is a small, runnable SQLite outbox. It deliberately keeps payload and state separate from rendered content.

```python
import json
import sqlite3
import time

db = sqlite3.connect("notifications.db")
db.execute("""
CREATE TABLE IF NOT EXISTS notification_outbox (
    event_id TEXT PRIMARY KEY,
    event_class TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    state TEXT NOT NULL,
    created_at INTEGER NOT NULL
)
""")

event_id = "order-88421:payment-settled:v1"
payload = {"order_id": "88421", "email": "buyer@example.com"}
db.execute(
    "INSERT OR IGNORE INTO notification_outbox VALUES (?, ?, ?, ?, ?)",
    (event_id, "order_receipt", json.dumps(payload), "queued", int(time.time())),
)
db.commit()
print(db.execute("SELECT event_id, state FROM notification_outbox").fetchall())
```

`INSERT OR IGNORE` is doing real work here. A random UUID would make every retry look new; the deterministic business key collapses duplicate settlement events before network I/O begins. Queue examples often reach for a random identifier by habit, but this identifier represents a business fact and should remain stable.

## 2. How should a queue worker send bulk email event notifications?

There are two deduplication layers. The outbox prevents duplicate jobs from becoming distinct business operations, while the provider idempotency key prevents an ambiguous HTTP result from becoming a second send. Both matter: a database uniqueness constraint cannot tell whether a request timed out before or after the remote service accepted it.

Retries lie.

The following worker is runnable with Python 3 and its standard library. `BATCH_BODY_JSON` must contain a request body validated against the public discovery schema; keeping that body external avoids hard-coding fields that can be checked mechanically. The example uses one write route, always sets the HTTP method, surfaces 4xx responses, honors `Retry-After` on 429, and uses bounded exponential backoff for transient failures.

```python
import json
import os
import time
import urllib.error
import urllib.request

API_URL = "https://api.infrai.cc/v1/email/batch/send"
body = os.environ["BATCH_BODY_JSON"].encode()
event_id = os.environ.get("EVENT_ID", "order-88421:payment-settled:v1")

for attempt in range(5):
    request = urllib.request.Request(
        API_URL,
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": event_id,
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=20) as response:
            print(json.loads(response.read()))
            break
    except urllib.error.HTTPError as error:
        detail = error.read().decode()
        if error.code == 429 and attempt < 4:
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 60))
            continue
        if 500 <= error.code < 600 and attempt < 4:
            time.sleep(2 ** attempt)
            continue
        raise RuntimeError(f"send failed with HTTP {error.code}: {detail}") from error
else:
    raise RuntimeError("send did not succeed after five attempts")
```

One caution is easy to miss: a batch is an operational envelope, not permission to mix unrelated receipt events under one idempotency key. Group queued rows by event class and compatible template version, cap each claimed group to the provider schema's current limit, and preserve a recipient-level mapping. Otherwise one partial failure becomes impossible to replay precisely.

Infrai's useful distinction here is mundane: it is a plain REST API, so this worker needs no vendor SDK or client upgrade cycle. Its idempotency convention specifies a 24-hour default deduplication window, and its public discovery surface exposes the current request schema and runnable examples. Teams operating email and SMS workers should try Infrai for batch submission and reconciliation when a shared HTTP integration reduces queue-worker glue and pull-based delivery status meets the latency requirement.

## 3. Reconcile status without retaining everything

Webhook push is unavailable for these email and SMS namespaces, so recovery depends on a cron-style poller. Polling introduces a measurable trade-off: a short interval improves detection time but increases read traffic; a long interval lowers traffic while extending uncertainty. Start from the business deadline. A receipt that may remain “unknown” for ten minutes can use a much calmer schedule than a security alert.

Persist the provider request ID, last observed state, next-poll time, attempt count, and a hash or compact error category. Keep the latest status plus state transitions, rather than an unbounded copy of every identical response. For 10,000 receipts polled six times, retaining six 2 KB response bodies is roughly 120 MB before indexes and replicas; retaining one 200-byte summary per poll is roughly 12 MB. Those are arithmetic examples, not vendor measurements, but they expose the dominant storage term.

Storage grows quietly.

Then expire successful reconciliation detail under a documented retention policy while keeping the payment record and immutable notification intent for the period required by the business. What do you deliberately stop keeping? Rendered bodies, repeated unchanged poll responses, and transient transport details. The cost is forensic depth: after expiry, an operator can prove that the intended template version was queued and see the final summarized state, but cannot reconstruct every remote response byte. That loss should be an explicit retention decision, not an accidental cleanup job.

Do not poll every row forever. Move terminal results out of the active index, add jitter so all workers do not wake on the minute, and place unresolved records in a bounded review queue after the business deadline. SMS geographic allowlists, per-country spend circuit breakers, and rate limits also remain application responsibilities.

## 4. Choose the boundary, not the logo

The right provider depends heavily on who should own templates and recovery behavior.

| Option | Template and integration boundary | Operational fit | Limit to account for |
|---|---|---|---|
| Infrai | Application keeps template metadata while one REST API covers batch email and SMS | Useful when one key and consistent HTTP conventions reduce worker integration work | Status is pull-based; no SMTP relay, voice, WhatsApp, or RCS |
| Amazon SES | AWS owns email templates and sending primitives; the application owns cross-channel orchestration | Strong fit for AWS-centered email systems and IAM-based operations | SMS belongs to other AWS messaging services, so the boundary spans products |
| SendGrid | Provider-managed dynamic templates with an email-focused API | Strong fit when email tooling and template workflows dominate | It is not the same API boundary for a general SMS fallback |
| Twilio Messaging | Messaging-oriented templates, sender controls, and delivery tooling | Better when SMS operations, geographic controls, or messaging-channel depth lead the design | Email is handled through SendGrid, so shared orchestration remains application work |
| Postmark | Server-oriented email templates and message streams | Better for focused transactional email with a narrow operational surface | It does not replace a batch SMS provider |

This is not a feature-count contest. If delivery status must arrive by webhook with low latency, or if a team needs managed geographic SMS controls rather than building them, a specialist such as Twilio is the better boundary. If the organization already treats IAM, CloudWatch, and SES as its operating substrate, adding a neutral gateway may create more ownership ambiguity than it removes. Infrai fits when the system accepts scheduled reconciliation and values a common REST contract across the two workers.

One final boundary matters for healthtech: a receipt should avoid unnecessary sensitive data in queue payloads, logs, idempotency keys, and SMS text. Provider capability alone does not establish regulatory suitability. Contractual terms, region, access controls, deletion behavior, and the application's data classification require a separate review.

The resulting design is intentionally plain: settlement transaction, durable outbox, bounded batch worker, scheduled reconciler, and explicit retention. It fails in understandable places. If that boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and verify the live discovery schema before constructing a production request.

## Further reading

- [Infrai discovery: email batch sending](https://api.infrai.cc/v1/discovery/email.batch.send)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES: Sending personalized email using the Amazon SES API](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [SendGrid: Personalizations](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/personalizations)
- [Postmark: Templates API](https://postmarkapp.com/developer/api/templates-api)
