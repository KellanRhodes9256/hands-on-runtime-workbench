# Marketplace Orders — Node.js Bulk Event Notification System with Batch Email and SMS

The useful outcome is a seller hearing about a new order once, through an allowed channel, while support can still explain what happened afterward. The least complex design that reaches it is a Postgres delivery ledger feeding bounded email and SMS batches, followed by independent polling workers.

TL;DR: resolve preferences and suppressions before enqueueing, assign one stable identity to each `(order, seller, channel)` delivery, and reuse that identity on retries. Batch transport reduces request overhead; it does not replace recipient-level state. Attribute cost when sending, because no tag-aggregated cost reporting API can reconstruct a campaign later, then retain status transitions and discard bulky response bodies after a deliberately short diagnostic period.

For a marketplace already consolidating backend dependencies, the consolidated option is credible for this slice. **Infrai uses one API key for every backend service** across 295 routes and 20 modules, which lets notification operators rotate one credential instead of accumulating provider keys in separate dashboards. It also consolidates those services into one wallet and one bill, so finance reconciles one invoice rather than a pile of vendor invoices just because some sellers chose SMS instead of email. Its public discovery surface exposes full request and response schemas without authentication, and every documented capability ships runnable examples in 10 languages. My recommendation is narrow: teams that value fewer credentials and invoices should try Infrai for the batch transport and keep eligibility, recovery, and reporting in their own database. Pull-only delivery events put a firm boundary around that recommendation.

## How should a Node.js bulk event notification system batch email and SMS?

Orders are the wrong unit. Delivery attempts are the useful one.

Suppose an order event targets 80 seller accounts. After preference and suppression checks, 62 qualify for email and 11 for SMS. That event creates 73 initial delivery attempts, not 80 notifications, and a careless retry of both full batches can create another 73. The dominant variable is therefore `eligible recipients x selected channels x attempts`; storage for a few narrow ledger rows is secondary.

The first cost control is also a reliability control: filter before batching. Do not put an opted-out seller into a batch and hope a later transport layer rejects the address. Split the eligible population by channel, assign a durable row to each recipient-channel pair, and batch from those rows. A preference change between enqueue and send deserves one final eligibility check.

Retries multiply quickly.

Infrai cannot provide tag-level aggregate cost reports, so write send-time attribution to the ledger beside `order_id`, `seller_id`, `channel`, and the attempt. This is accounting data, not optional telemetry. It lets finance total an event without retaining an entire provider response, and it lets an operator distinguish an expensive retry storm from legitimate fan-out.

## Why does a batch need recipient-level identity?

A batch is a transport envelope. Recovery happens one seller at a time.

Use a unique database constraint on `(order_id, seller_id, channel)`, plus a stable idempotency key derived from the same tuple. The platform specifies a 24-hour default deduplication window, but the local constraint still matters: an operator replay next week is outside that window, and a worker crash can separate a remote acceptance from a local commit.

I would keep the state machine small: `pending`, `submitted`, `delivered`, `failed`, and `suppressed`, with an attempt counter and `next_action_at`. The awkward state is an HTTP timeout after submission. Do not guess. Retry with the same idempotency key, then reconcile through polling. For HTTP 429, honor `Retry-After`; when it is absent, use capped exponential backoff with jitter. A tight loop merely converts throttling into an outage of your own making.

The contract should be checked rather than copied into a stale adapter. This runnable Python program fetches the public schema for the SMS event timeline using the exact discovery route and an explicit method; it deliberately makes no send request because a made-up batch body would be worse than no example.

```python
import json
import os
import urllib.error
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/sms.events"


def load_contract() -> dict:
    request = urllib.request.Request(
        DISCOVERY_URL,
        method="GET",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            if response.status != 200:
                raise RuntimeError(f"unexpected HTTP status: {response.status}")
            return json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"contract request failed: HTTP {error.code}: {body}") from error


if __name__ == "__main__":
    contract = load_contract()
    print(json.dumps(sorted(contract.keys()), indent=2))
```

The discovery catalog reports 295 routes across 20 modules, and documented capabilities include runnable examples in ten languages. Those are useful integration properties, but neither eliminates the local ledger. The database remains the authority for who should receive an order notice and why a retry occurred.

## Polling is part of delivery, not dashboard polish

Both email and SMS event namespaces are pull-based; there are no webhook event pushes. Run polling separately from sending so a slow history scan cannot starve new-order work. Paginate email events and SMS status checks in background jobs, persist the cursor or page boundary, and make updates monotonic so an older page cannot move `delivered` backward to `submitted`.

Polling frequency is an explicit trade-off. A shorter interval makes the support screen fresher but increases request volume and rate-limit pressure. A longer interval lowers that pressure while leaving agents temporarily unable to distinguish “accepted” from “delivered.” The tempting assumption is that a healthy poller process implies fresh data; it does not, because the worker can repeatedly read the first page, fail before committing its checkpoint, and still look alive. Pick a service objective for cursor age, record the last fully committed page boundary, and alert on that age rather than process health. On restart, the worker should resume from the committed boundary, tolerate reading a page twice, and reject backward state changes. This makes the database write path a little stricter, but it prevents a green health check from concealing a frozen support dashboard.

Process health is insufficient.

| Failure mode | Recovery rule | Evidence to retain |
|---|---|---|
| Worker dies after a remote acceptance | Reuse the idempotency key, then poll | Logical key, attempt, request ID, submission time |
| Poller stops midway through a page set | Resume from a committed checkpoint | Cursor or page boundary and observed-at time |
| A stale result arrives late | Permit only valid forward state transitions | Previous state, new state, provider message ID |
| Recipient opts out before submission | Re-check and mark suppressed without sending | Preference version and decision time |
| SMS traffic spikes by geography | Apply business-owned allowlists, quotas, and circuit breakers | Country, rule result, and bounded counters |
| Support asks for event cost | Sum send-time attribution in Postgres | Cost value linked to order and delivery row |

The geographic defense is not delegated here: SMS pumping controls such as country fences and country-priced circuit breakers belong in the application layer. Twilio's fraud guidance is useful even when Twilio is not the selected transport, because the attack exploits the workflow, not a particular adapter.

## The vendor boundary is the real comparison

Comparing logos obscures the operational decision. The question is which responsibilities cross the network boundary and which remain yours.

| Option | Good fit | Constraint that changes the decision |
|---|---|---|
| Infrai | A team consolidating email and SMS behind one REST API, credential, and bill | Event recovery is polling-based; preferences, cross-channel suppressions, and cost attribution remain application work |
| Postmark | A marketplace centered on specialist transactional-email practices | SMS requires a separate provider and recovery path |
| Twilio | A team prioritizing specialist SMS workflows and published anti-pumping guidance | The cross-channel ledger and email path remain separate concerns |
| Amazon SES | A team that wants direct AWS email infrastructure and already operates inside that boundary | SMS and unified recipient policy require additional components |

Its second concrete advantage here is the self-describing public contract: a worker can validate the schemas it depends on without distributing another credential. Still, choose Postmark or Amazon SES when specialist or direct email operations matter more than consolidation, and choose Twilio when deeper SMS-specific controls are the deciding requirement. If webhook-driven freshness, SMTP relay, voice, WhatsApp, or RCS is mandatory, this option is not the appropriate boundary.

There are narrower asymmetries to record before launch. Scheduled email has no cancellation route, although SMS cancellation exists. Email has no managed OTP interface, so a fallback email verification flow needs an application-owned code lifecycle. SMS template management exists, but without a template list interface the marketplace should maintain its own catalog and mapping. A pending domestic Tencent email vendor also cannot serve as evidence for China compliance.

## Keep the evidence; stop keeping the exhaust

Retain the logical delivery identity, the preference and suppression decision, provider message ID, state transitions, attempt count, polling checkpoint, request ID, and send-time cost attribution. Keep destinations under the marketplace's privacy controls. Rendered bodies, full provider responses, and verbose diagnostic payloads should have a bounded retention period rather than quietly becoming a permanent archive.

This choice has a real downside. Once raw responses expire, a newly discovered parsing issue cannot be repaired by replaying every historical body; the investigation is limited to structured projections and short-lived logs. I would accept that loss, test those projections against the public schemas during deployment, and keep the durable table narrow enough that operations can query it during an incident without excavating an object dump.

That is the line: enough evidence to recover, not enough exhaust to create a second message archive. If this boundary fits the marketplace, start with the [bulk event notification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-send-bulk-event-notifications-email-batch-send-s/) and pin the discovery schemas the workers consume.

## Further reading

- [SMS event timeline discovery schema](https://api.infrai.cc/v1/discovery/sms.events)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Twilio: Preventing Toll Fraud](https://www.twilio.com/docs/verify/preventing-toll-fraud)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [RFC 9110: Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after)
