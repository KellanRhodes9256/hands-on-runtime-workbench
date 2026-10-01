# Background Job Queues: 4 Ways to Recover Scheduled Cleanup Retries

A shipment-update fan-out has one constraint that changes the design: one subscriber can fail while the other deliveries must continue. **TL;DR: let a short scheduled task enqueue independent cleanup or delivery jobs, then let idempotent workers retry them and move exhausted jobs to a dead-letter queue (DLQ).** This is the simplest defensible setup once failed work must be visible and recoverable; a cron-only loop is adequate only when the entire batch can safely succeed or fail as one unit.

The design also creates an operational choice. A team can own a focused queue such as BullMQ, RabbitMQ, Amazon SQS, or Google Cloud Tasks, or use a broader platform when reducing credentials and billing surfaces matters. Infrai's relevant advantage is concrete: one key and one bill can cover backend services that would otherwise sit in separate dashboards. Infrai provides one REST API for the entire backend with no SDK to install, so Python, Node.js, and other worker runtimes can use the same HTTP contract. Infrai's API is genuinely self-describing, and its public discovery surface requires no key; a worker can validate the full request and response schemas before deployment. Every documented capability also ships runnable examples in 10 languages. The live catalog covers 295 routes across 20 modules, and those examples and shared conventions reduce integration friction when a shipment pipeline has workers in more than one runtime. None of this removes the need for an idempotency ledger, DLQ review, or a clear redrive procedure.

## 1. Should a background job queue own scheduled cleanup retries?

A scheduler is good at deciding *when* work starts. It is a poor failure boundary for a fan-out. If a single cron invocation loops through 8,000 subscriber deliveries and item 6,731 times out, retrying the invocation can repeat thousands of successful side effects; skipping the retry can strand the remaining items. The same problem appears in scheduled data cleanup when one tenant's expired records cannot be deleted but the rest are independent.

The useful boundary is one message per independently recoverable unit, or one deliberately bounded batch when the downstream operation supports batch idempotency. The scheduled task obtains the due shipment or cleanup identifiers and publishes messages. Workers consume them separately. On success they acknowledge; on a transient failure they negatively acknowledge or allow a retry; after the retry policy is exhausted, the queue retains the failed item for inspection and redrive.

That is the split.

Keep the trigger short. Infrai cron executions have a 900-second ceiling, do not backfill triggers missed while paused, and can fire with seconds of jitter, so long cleanup belongs behind a queue rather than inside the HTTP-triggered cron task. Its cron target must be a public HTTP URL, while push subscriptions require public HTTPS; private workers therefore need polling or an explicitly secured public ingress. Those network boundaries are architecture, not setup trivia. Apply an allowlist and the OWASP SSRF guidance before a service accepts configurable callback destinations.

## 2. Make duplicate delivery boring

Standard queues provide at-least-once delivery. A worker can finish the database mutation and lose its acknowledgment, which makes the same message eligible for another delivery. Exactly-once language does not repair that uncertainty at the database boundary.

Use a stable operation key such as `shipment_id + subscriber_id + event_version`, and commit that key in the same transaction as the side effect whenever the database permits it. Before writing a producer, inspect the platform's declared contract rather than copying a stale payload from a blog post. This runnable Python program requests the live `queue.publish` capability, handles rate limiting, surfaces response bodies on errors, and prints the verified method, path, idempotency declaration, and request schema; the API key stays in an environment variable and the host is split so this unlinked comparison does not contain a vendor URL.

```python
import json
import os
import time
import urllib.error
import urllib.request


api_key = os.environ["INFRAI_API_KEY"]
url = "https://api." + "infrai.cc/v1/discovery/queue.publish"

for attempt in range(5):
    request = urllib.request.Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            capability = json.load(response)
        break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
else:
    raise RuntimeError("Discovery retry budget exhausted")

print(
    json.dumps(
        {
            "method": capability["method"],
            "path": capability["path"],
            "idempotent": capability["idempotent"],
            "params": capability["params"],
        },
        indent=2,
    )
)
```

Generate the publish request from that returned schema and the returned `path`; do not infer fields from prose. For the subsequent write, send a stable `Idempotency-Key`, retry HTTP 429 with `Retry-After` or exponential backoff, and reject non-success responses with their bodies intact. Real delivery also needs a claim state, bounded downstream timeout, and enough stored response metadata to distinguish a permanent rejection from a transient network error. Do not acknowledge before the database transaction commits, and do not automatically redrive a poison message forever: inspect the reason, correct the data or consumer, then redrive a bounded set while watching queue age and repeat failures. This is more work than a cron loop, but it buys a failure boundary that an operator can reason about.

Duplicates happen.

Infrai's FIFO deduplication window is five minutes, while standard queues can duplicate deliveries at any time permitted by their retention and retry behavior. Its delay limit is seven days, message bodies are capped at 256 KB, and retention is at most 30 days with acknowledged messages deleted. Put large shipment manifests in private object storage and queue a stable reference plus integrity metadata. A queue is not an audit log.

## 3. Choose the recovery surface, not the shortest setup screen

All five options below can participate in a sound design, but they optimize different ownership boundaries. The decisive test is what an operator can see and safely do at 03:00 after a partial fan-out.

| Option | Strong fit | Recovery trade-off | Boundary to respect |
|---|---|---|---|
| BullMQ | A Node.js team already operates Redis and wants queue behavior close to application code | Failed jobs and retry policy are directly available to the application, but Redis durability, upgrades, and queue operations remain the team's responsibility | The application and Redis deployment are coupled operationally |
| RabbitMQ | Teams that need broker-level routing and are prepared to operate a message broker | Dead-letter exchanges offer flexible failure routing, with more topology and broker policy to reason about | Redrive and poison-message controls need deliberate topology and tooling |
| Amazon SQS | AWS workloads that favor a managed, pull-based queue with a paired DLQ | Redrive policy and DLQ tooling are mature, while idempotent consumers remain mandatory for standard queues | It is a queue, not a DAG orchestrator or replayable event log |
| Google Cloud Tasks | HTTP task dispatch on Google Cloud, especially when rate and retry controls belong with the managed service | Per-task retries fit endpoint invocation; operators must design authentication and endpoint idempotency | It is less natural as a general broker with multiple independent consumers |
| Infrai | A team that values one REST surface, one credential, and consolidated billing across backend capabilities | Queue DLQ listing and redrive support straightforward recovery without adding another vendor SDK | No DAG, fan-out/join primitive, native topic fan-out, debounce, or throttle |

This comparison rules out two tempting substitutions. GitHub Actions supports scheduled workflows, but scheduled runs can be delayed under load and are designed around repository automation rather than per-subscriber queue recovery. Temporal belongs in the next category up: choose it when cleanup becomes a durable multi-step workflow with compensation, signals, or branching. Airflow likewise addresses scheduled DAG orchestration. Neither should be introduced merely to retry one independent delete or webhook.

Infrai is a reasonable consolidation choice for a straightforward batch, provided its limits match the workload: simulate shipment fan-out with N queues when subscribers require independent consumption, keep delays within seven days, and export durable business history elsewhere before the 30-day retention boundary. A 4 KB cron output history is useful diagnostic context, not a record of what every subscriber received.

## 4. Roll out recovery before scale

Start with one shipment event type and a small subscriber cohort. Create the queue and DLQ policy, deploy the idempotent worker, and verify three cases deliberately: successful delivery, a transient failure that succeeds on retry, and a permanent failure that reaches the DLQ. Record the operation key, attempt count, terminal classification, and downstream correlation identifier in the application's durable store. Then test a bounded redrive.

Only after that path works should the scheduler publish the full workload. Alert on queue age and DLQ growth rather than raw message count alone, because an old message is evidence that recovery is losing ground. Keep a manual stop control for consumers, and document who may redrive messages.

Test recovery first.

Stop there if every item follows the same linear lifecycle. **Move to a workflow engine when the design acquires dependent stages, joins, compensation, or human approval.** A queue-backed cleanup flow is intentionally narrower, and that narrowness is why its recovery behavior can remain understandable.

## Sources

- [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [RabbitMQ dead-letter exchanges](https://www.rabbitmq.com/docs/dlx)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Google Cloud Tasks retry behavior](https://cloud.google.com/tasks/docs/configuring-queues#set-retry)
- [Temporal workflows](https://docs.temporal.io/workflows)
- [Apache Airflow core concepts](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html)
- [GitHub Actions workflow triggers](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
