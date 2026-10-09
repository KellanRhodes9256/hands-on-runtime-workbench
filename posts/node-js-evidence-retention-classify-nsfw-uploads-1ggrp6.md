# Node.js Evidence Retention: Classify NSFW Uploads with Multimodal JSON

Short answer: for Node.js image upload moderation in a B2B SaaS code-review service, use multimodal chat to classify NSFW content, violence, and hate symbols, validate its JSON against a schema, and retain the uploaded image only long enough to obtain a small safety record. Make that record, rather than the image or a provider-specific response, the durable input to policy. The least complex design is a quarantine object, a provider-neutral classifier adapter, strict JSON Schema validation, and a short retention clock. Its bill is dominated by retained image bytes and repeated classification work, so the first useful calculation is `uploads x average bytes x retention days`, followed by the retry and manual-review rates. Do that before debating models.

The change that moves the dominant storage term is deletion, not compression: keep the original while classification or an authorized appeal can still need it, then delete it on a documented schedule. I would deliberately stop keeping indefinite originals, raw multimodal responses, and duplicate thumbnails. The cost is real. After deletion, an operator can explain a decision from the normalized record but cannot reclassify the exact bytes, inspect a newly discovered visual cue, or prove that a parser discarded no useful detail. A retention policy is therefore a durability decision, not housekeeping.

## How should Node.js classify an image upload for NSFW moderation?

An upload attached to a proposed code change can contain a UI screenshot, architecture diagram, generated test artifact, or unrelated material. The safety gate needs to classify NSFW content, violence, and hate symbols without pretending that three booleans settle every case. Context matters: a hate symbol in a security report and the same symbol used as praise may demand different handling, while a screenshot of violent content may be evidence of a defect. The classifier should report observations and uncertainty; a separate policy layer should decide allow, block, or review.

Keep the durable record narrow:

- a random asset identifier and tenant identifier;
- a content digest, byte count, media type, and capture time;
- one status per category: `present`, `absent`, or `uncertain`;
- bounded confidence values and a short evidence note;
- schema version, policy version, adapter name, and adapter revision;
- timestamps for classification and source deletion.

Do not use the object key as evidence. Do not retain the image as an implicit audit log either. An object can be deleted while a database row survives, and that split is useful only if the row records the transition explicitly. Otherwise, a missing object is indistinguishable from a lifecycle action, an operator mistake, or a failed write.

The record should also avoid embeddings unless a separate, reviewed use case requires similarity search. pgvector adds exact and approximate vector search to Postgres, but that capability does not make a vector necessary for a three-category policy gate. Likewise, tiktoken is a BPE tokenizer for OpenAI models; token accounting can matter for text inputs, yet it does not measure retained image bytes. Those are separate cost surfaces.

## Put a stable contract between intake and classification

Provider portability fails when raw response shapes leak into queues, database columns, dashboards, and appeal tooling. A thin adapter is not enough if every adapter is permitted to reinterpret the categories. Define one internal contract first, test it with fixtures, and make provider-specific code translate into it.

The focused example below is Python because the classifier boundary should be language-neutral even when the upload service runs on Node.js. It validates a decoded response and deliberately converts malformed or out-of-range output into a review result. It does not guess missing labels.

```python
from dataclasses import dataclass
from typing import Any, Literal

Label = Literal["present", "absent", "uncertain"]
CATEGORIES = ("nsfw", "violence", "hate_symbols")


@dataclass(frozen=True)
class Finding:
    label: Label
    confidence: float
    evidence: str


@dataclass(frozen=True)
class ModerationRecord:
    schema_version: str
    findings: dict[str, Finding]
    disposition: Literal["allow", "block", "review"]


def normalize(payload: Any) -> ModerationRecord:
    if not isinstance(payload, dict) or payload.get("schema_version") != "1":
        return review_record("invalid envelope")

    raw_findings = payload.get("findings")
    if not isinstance(raw_findings, dict) or set(raw_findings) != set(CATEGORIES):
        return review_record("missing or unknown category")

    findings: dict[str, Finding] = {}
    for category in CATEGORIES:
        item = raw_findings.get(category)
        if not isinstance(item, dict):
            return review_record(f"invalid {category} finding")

        label = item.get("label")
        confidence = item.get("confidence")
        evidence = item.get("evidence")
        if label not in {"present", "absent", "uncertain"}:
            return review_record(f"invalid {category} label")
        if not isinstance(confidence, (int, float)) or not 0 <= confidence <= 1:
            return review_record(f"invalid {category} confidence")
        if not isinstance(evidence, str) or len(evidence) > 240:
            return review_record(f"invalid {category} evidence")
        findings[category] = Finding(label, float(confidence), evidence)

    disposition = apply_policy(findings)
    return ModerationRecord("1", findings, disposition)


def apply_policy(findings: dict[str, Finding]) -> Literal["allow", "block", "review"]:
    if any(item.label == "uncertain" for item in findings.values()):
        return "review"
    if any(item.label == "present" for item in findings.values()):
        return "block"
    return "allow"


def review_record(reason: str) -> ModerationRecord:
    findings = {
        category: Finding("uncertain", 0.0, reason) for category in CATEGORIES
    }
    return ModerationRecord("1", findings, "review")
```

In production, JSON Schema should constrain required keys, reject additional properties, bound strings and numbers, and enumerate labels before this domain mapping runs. If a multimodal chat endpoint offers schema-constrained output, use it as the first parsing path. The fallback is still ordinary JSON decoding followed by the same validator. A fallback must never scrape prose for reassuring words; that turns punctuation changes into policy changes.

Short code, strict boundary.

The adapter also needs a small error vocabulary: timeout, unavailable, rejected media, invalid output, and internal failure. Those errors belong in operational telemetry, while the user-facing workflow receives `review` or a retryable intake state according to policy. Never translate a network timeout into `absent`.

## Retention changes the failure model

The safest order of operations is intentionally boring. Accept bytes into a quarantined namespace, verify the declared type against the decoded format, enforce byte and dimension limits, compute a digest, and create a pending row. Then classify from the quarantined object, validate the normalized result, commit the decision, and schedule object deletion. Publication or attachment to the code review happens only after policy allows it.

Retries complicate that sequence. Use the asset identifier plus classifier revision as an idempotency key, and let one database state transition win. A retry may reuse the same quarantined object; it must not create another permanent copy. If classification succeeds but the database commit fails, the job can run again. If deletion fails, the record stays in a deletion-pending state and an independent sweeper retries it.

| Failure mode | Default state | Bytes retained | Recovery |
|---|---|---:|---|
| Classifier timeout | Pending or review | Quarantine object | Bounded retry, then human review |
| Invalid JSON | Review | Quarantine object until deadline | Preserve validation error, not raw prose |
| Database commit failure | Pending | Quarantine object | Retry idempotently |
| Deletion failure | Deletion pending | Quarantine object | Sweep and alert on age |
| Duplicate delivery | Existing state | No new durable copy | Return the recorded result |

The table exposes the uncomfortable trade-off: deleting earlier reduces exposure and storage, but shrinks the window for appeal and forensic replay. Keeping everything makes later replay easy, but creates a growing collection of sensitive user content. Pick a window from legal, abuse-response, and customer-contract requirements; no universal number follows from the classification task itself.

## Test the boundary, the policy, and the clock

Golden images alone are a weak test suite because expected labels can be disputed and model revisions can shift scores. Keep a consented, access-controlled evaluation set with category, context, and expected policy outcome. Add parser fixtures that contain missing keys, extra keys, wrong types, `NaN`-like values, excessive evidence text, and unknown labels. Run the same fixtures against every adapter.

Then test time. A fake clock should prove that pending objects remain available during a permitted retry, allowed and blocked objects enter deletion-pending state at the correct deadline, and expired objects disappear while their normalized records remain. Restore drills matter too: a database restore must not resurrect references that imply deleted bytes still exist.

Deployment should use shadow classification before a provider or model revision controls policy. Compare category disagreements, uncertain rates, latency distributions, and invalid-output counts, segmented by media type and tenant only where privacy rules permit it. Do not log image bytes, data URLs, raw prompts, or full raw responses. Metrics need counts and durations; diagnosis needs asset identifiers with tightly controlled lookup.

Capacity planning follows the state machine. Track quarantined bytes by age, deletion backlog, retry count, review queue age, and classification calls per terminal decision. A low request error rate can hide a stalled deleter, while a healthy deleter can hide a classifier that sends everything to human review. Both are operational failures.

## Choose portability by running an exit test

Claims of portability are cheap. An exit test is measurable: implement a second adapter against recorded, consented fixtures; require the same normalized schema; run it in shadow; and verify that queues, database migrations, policy code, dashboards, and review tools do not change. Only the adapter and its configuration should move.

Evaluate candidates against the boundary rather than a feature checklist:

| Decision axis | Question that exposes coupling |
|---|---|
| Structured output | Can invalid output be detected without provider-specific parsing? |
| Image transport | Can the adapter switch between signed-object access and byte upload? |
| Error handling | Can provider errors map into the internal error vocabulary without losing cause? |
| Revision control | Can every decision name the exact adapter and model revision? |
| Evaluation | Can the same consented fixture set run without rewriting labels? |
| Data handling | Can retention and processing constraints be enforced and audited? |

This test also clarifies what to retain. Keep normalized findings, policy decisions, revisions, digests, and lifecycle timestamps for the approved audit period. Keep raw output only when a documented investigation requirement justifies it, with its own shorter deletion deadline. Stop keeping source images after the appeal and retry window. When a future category is introduced, old decisions remain explainable but cannot be recomputed from deleted pixels; the schema version makes that limitation visible instead of concealing it.

That is the honest bargain: fewer sensitive bytes and less provider coupling, purchased with a finite replay window. For a code-review SaaS, the durable artifact should be a versioned safety decision whose provenance can be audited, not a warehouse of customer screenshots.

## Further reading

- [openai/tiktoken: a BPE tokenizer library](https://github.com/openai/tiktoken)
- [pgvector: vector similarity search for Postgres](https://github.com/pgvector/pgvector)
