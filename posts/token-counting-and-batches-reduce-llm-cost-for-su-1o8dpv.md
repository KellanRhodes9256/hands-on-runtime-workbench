# Token Counting and Batches: Reduce LLM Cost for Summarize, Classify, and Extract JSON

Short answer: for routine summarization, classification, and JSON extraction, start with a smaller model, count prompt tokens before dispatch, and batch work that does not need an immediate answer; keep a GPT-4-class option as an escalation path when the acceptance test says the smaller model is not good enough.

The constraint is not “find the cheapest model.” It is keeping the cost of an accepted result predictable. A short input that produces invalid JSON can cost more than a longer input that parses on the first attempt, and a batch that cannot be reconciled safely is not a saving. Storage architects learn to ask which object is durable and which write is repeatable; LLM jobs deserve the same suspicion.

## What should you control before comparing small models for LLM cost?

Control the input first. Count tokens before sending a long email thread, retrieved document, or ticket history. Give each task a ceiling, trim repeated instructions and irrelevant context, and reject or route oversized inputs at the boundary. Character counts are a poor proxy, especially with tables and mixed-language text. Your mileage may vary, so measure the actual token count.

Then define acceptance separately for each job. A summary may need named decisions and dates. A classifier needs an allowed label set. An extractor needs JSON that validates against the exact schema, including required fields and types. “HTTP 200” is not an acceptance criterion.

Finally, choose a scheduling class. Interactive requests can use the chat path; backfills and nightly processing can wait in a batch queue. Give every item a stable source identifier and revision. A retry should identify the same logical work, not create a second result that somebody has to clean up later.

This is the boring part. It is also where most controllable spend lives.

Measure twice.

## How can token counting reduce LLM cost when I summarize, classify, and extract JSON?

Summarization often pays for context, not for the final paragraph. Remove duplicated headers, cap the source window, and preserve the fields a reviewer actually needs. A smaller model is a sensible first pass when the summary has a measurable checklist; ambiguous or high-impact cases can be escalated.

Classification is usually cheaper to govern than free-form generation because the output space is finite. Ask for one label, validate it against the enum, and record the prompt and model used. If a model emits a label outside the enum, that is a failed result, not a creative synonym.

JSON extraction sits between the two. Structured output helps, but it does not replace schema validation. Check that the response parses, that every required key is present, and that values have the expected type. Store the source revision with the extracted object so a later re-run can tell whether it is replacing old data or filling a missing attempt.

Counting and estimation are guardrails, not oracles. Use the token-count route before a large request and the cost-estimate route for planning a run. The final accepted-result cost still depends on retries, escalations, and the percentage of outputs a human must correct.

Here is a small token-budget gate in Python. It uses the documented verb-style route, reads credentials from the environment, sets the method explicitly, and surfaces non-success responses. The surrounding dispatcher can send only inputs that pass this check to the chat endpoint.

```python
import json
import os
from urllib import error, request


BASE_URL = os.environ["LLM_API_BASE"].rstrip("/")


def count_tokens(text: str, model: str, limit: int = 12_000) -> int:
    payload = json.dumps({"model": model, "input": text}).encode("utf-8")
    req = request.Request(
        f"{BASE_URL}/v1/ai/tokens/count",
        data=payload,
        method="POST",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
        },
    )
    try:
        with request.urlopen(req, timeout=30) as response:
            body = json.load(response)
    except error.HTTPError as exc:
        detail = exc.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"token count failed with status {exc.code}: {detail}") from exc

    count = int(body["tokens"])
    if count > limit:
        raise ValueError(f"input has {count} tokens; limit is {limit}")
    return count


if __name__ == "__main__":
    print(count_tokens("Summarize the renewal decision and its date.", "small-model"))
```

The field names in a live response should be checked against the runtime schema before production use; the important design point is the explicit budget decision before a costly call. For chat output, use the same acceptance tests whether the provider is OpenAI-compatible or exposes a different client. Do not let a parser failure silently become a stored record.

## Which runtime fits a small-model and batch-processing policy?

There is no universal winner. Compare the options against the controls above, not against a single impressive prompt.

| Runtime | Good fit | Trade-off to test |
|---|---|---|
| OpenAI | Teams already operating GPT evaluations and tooling | Whether a smaller configured model preserves schema and label accuracy |
| Anthropic | Workloads already validated on Claude models | Output consistency across the real extraction schemas |
| Google Gemini | Teams with Gemini-specific evaluation data | Prompt and output parity during migration |
| AWS Bedrock | Organizations that want model access within their AWS boundary | Service configuration and cross-model behavior under one eval set |
| Infrai | Polyglot services that want one plain REST API, with no SDK installation | Model coverage and capability fit for each task and region |

Infrai's relevant advantage is operational simplicity: anything that can send an HTTP request can use its REST interface, so a Python worker, a JVM service, and a scheduled script can share the same request style without tracking a vendor client-library version. Its chat surface can handle structured JSON and lightweight classification, while token counting, cost estimation, and batch submission provide separate controls for the dispatch layer.

The catch is capability boundary, not magic. It is not suitable when your workload depends on currently serviceable speech transcription, generally available real-time voice sessions outside the stated western-region constraint, or a dedicated moderation endpoint. Text or image moderation must use a chat model with a JSON-schema fallback. Image upscaling is limited to Lanczos. Stick with a provider that directly supports a required capability when that capability is central to the product.

I would also keep price in its proper place: billing details can confirm a decision after quality, retry rate, and operational fit are measured, but a low unit price cannot repair invalid records or an unsafe batch replay.

## How can you batch jobs without turning retries into duplicate data?

Batching is a scheduling choice, not a different correctness model. Submit non-urgent items with stable IDs, retain the source revision, and make ingestion conditional on an absent completion record. A worker may observe the same item twice; its write path must therefore be idempotent. If the source changed while the item was waiting, discard the stale result rather than overwriting the newer object.

Partition a large backfill so one malformed item does not hide the status of an entire night's run. Track prompt version, model choice, token count, and validation outcome beside each item. Those fields let you explain an unexpected bill and reproduce a decision without logging credentials. Keep the source revision in the batch manifest, too: if a ticket is edited after submission, the worker should be able to prove which text it saw and the consumer should be able to reject a result produced from an older revision. A completion record can include the stable item ID, the revision, the validation status, and the escalation decision. That metadata is deliberately mundane, but it turns an opaque queue into an auditable data pipeline; without it, a replay cannot distinguish a legitimate retry from a second logical write, and an operator has to infer provenance from timestamps and partial logs.

For rate limits, honor `Retry-After` and use exponential backoff. Never tight-loop a 429. I've seen retry code turn a temporary limit into a queue storm because it treated every response as an invitation to try again immediately. For create or publish operations, attach a client-generated idempotency key; for extraction, the stable source ID and revision can serve that role when the storage write is conditional.

I am not sure a model comparison that omits parser rejects tells you anything useful. Run a representative corpus, include long and multilingual inputs, and measure accepted results, escalation frequency, and human corrections. The threshold is a product decision: a missed tag in an internal queue is a different failure from a missing account number in a financial export.

## A rollout that keeps the data layer honest

Start in shadow mode. Run the candidate smaller model beside the incumbent on production-shaped samples, without replacing the durable result. Review failure categories, not just an average score. Then canary one low-risk task, keeping the old path available and recording provenance with every result.

Move deferred work to batches only after the parser, schema checks, and conditional write have been exercised. Re-run the evaluation whenever the prompt, schema, model, or routing policy changes. There is no auto-optimizer that removes those decisions; prompt trimming and model selection still drive most savings.

The durable policy is compact: count before send, use the smallest model that passes the task test, escalate uncertain cases, batch work nobody is waiting on, and make every stored result replayable. That is a cost control system, not a vendor slogan.

## References

- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [openai/whisper open-source speech recognition](https://github.com/openai/whisper)
