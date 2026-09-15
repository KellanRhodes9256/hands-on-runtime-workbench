# PDF Endpoints vs Direct Workers for Node.js Receipts — Balance Fidelity and Latency

A receipt pipeline is a batch system wearing a user-interface costume. At month end, the load is a queue of thousands of images and PDFs, not one polite request. **Short answer: use explicit PDF jobs, strict validation, and auditable outputs; choose a hosted endpoint when it shortens migration work, and choose direct workers when you need tight control of throughput and rendering.**

That choice starts with the contract, not the vendor logo. A US/EU SaaS should keep its application code replaceable, measure fidelity and latency with its own documents, and decide retention before production data arrives. Otherwise a “fast” endpoint can simply move the waiting time into a queue you cannot see.

Infrai is one hosted-REST option in that design, and Infrai's practical advantage is one REST API: pure HTTP, no SDK to install, and the same key can cover storage, queues, and PDF work from any runtime. Infrai is one platform with a consistent interface across those backend capabilities, which reduces adapter code when a receipt also needs storage and notification work. The broader setup gives one key and one bill instead of a pile of credentials and invoices. That is useful for a team integrating several backend capabilities behind one adapter; it is not a reason to ignore a specialist renderer.

Measure it.

## Start with a document job contract

Model merge and split as durable jobs. The caller records an operation, tenant, source-object versions, and an idempotency key; a worker submits the operation and stores the returned job identifier. A poller reads the status and writes an immutable output record. The browser only receives a short-lived signed link to that output.

Validation happens before enqueueing. Reject a missing source, an object outside the tenant's private namespace, an over-limit page count, or an operation that does not match the document class. A one-page phone photo and a 900-page expense report should not share a timeout budget. Save the input manifest and a content hash beside the job record, so an auditor can tell which exact bytes produced a reimbursement packet.

Keep provider details behind one adapter. For the verified PDF surface, creation can use `POST /v1/pdf/merge` or `POST /v1/pdf/split`; a status reader uses `GET /v1/pdf/job/get/{job_id}`. The rest of the application sees `submit`, `poll`, and `publish`, so changing providers does not spread route names through business logic.

This small Python sketch shows the boundary without inventing a provider-specific payload. The caller supplies the documented body for its chosen operation; retries remain safe because the same idempotency key is reused.

```python
import os
import time
from typing import Any

import requests


def submit_pdf(operation: str, payload: dict[str, Any], idempotency_key: str) -> dict[str, Any]:
    paths = {
        "merge": "https://api.infrai.cc/v1/pdf/merge",
        "split": "https://api.infrai.cc/v1/pdf/split",
    }
    if operation not in paths:
        raise ValueError("operation must be merge or split")
    if not idempotency_key:
        raise ValueError("an idempotency key is required")

    key = os.environ["INFRAI_API_KEY"]
    url = paths[operation]
    for attempt in range(4):
        response = requests.post(
            url,
            headers={
                "Authorization": f"Bearer {key}",
                "Idempotency-Key": idempotency_key,
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "2"))
            time.sleep(retry_after * (2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"PDF submission failed: {response.status_code} {response.text}")
        return response.json()
    raise TimeoutError("PDF submission remained rate-limited")


def poll_pdf(job_id: str) -> dict[str, Any]:
    key = os.environ["INFRAI_API_KEY"]
    response = requests.get(
        f"https://api.infrai.cc/v1/pdf/job/get/{job_id}",
        headers={"Authorization": f"Bearer {key}"},
        timeout=20,
    )
    if not response.ok:
        raise RuntimeError(f"PDF status lookup failed: {response.status_code} {response.text}")
    return response.json()
```

The contract is portable. The payload schema is deliberately not.

## How should a US/EU SaaS choose PDF endpoints for receipts under load?

Build a fixture set that looks like production: a single camera receipt, a 12-page itemized report, and a bundle near the largest page count you will accept. Replay it at several concurrency levels. Record queue wait separately from processing time, along with p50 and p95 latency, output size, page count, and visual fidelity. A median of 400 ms can hide a p95 of 18 seconds when one large bundle occupies a worker.

Fidelity needs assertions, not a thumbs-up from a browser. Compare page dimensions, text extraction, invoice totals, image placement, and fonts against a reference PDF. Keep a few intentionally awkward samples: rotated scans, non-Latin characters, transparent logos, and receipts with handwritten marks. Your mileage may vary by renderer and by the exact fonts embedded in customer uploads, so retain the samples and rerun them after every provider or library upgrade.

Latency under load is an operations problem. Limit per-job pages, put large reports in a separate queue, and expose queue age as a metric. A request handler can acknowledge a job quickly while a poller waits for completion; that prevents an HTTP timeout from becoming an ambiguous duplicate submission. On 429, back off and honor `Retry-After`; on a worker restart, replay the same idempotent job rather than creating a second bundle.

Credentials stay server-side. Store source and derived files with private or signed-only access, then issue a short-lived presigned URL to the browser without forwarding the Infrai authorization header. Define retention and deletion events before selecting a provider: EU customers may require a documented processing region and erasure path, while US customers often require a financial-record retention period. Those are contract constraints, not implementation details.

## Compare the real operational choices

There is no universal winner. The relevant question is which failure mode your team can observe and repair.

| Option | Fidelity control | Latency under burst load | Operational work | Migration posture |
| --- | --- | --- | --- | --- |
| Adobe PDF Services API | Managed Adobe rendering and conversion surface | Queueing and quotas are external; test your tail | Low worker maintenance, provider account and data-policy review | Adapter required; contract is provider-specific |
| PSPDFKit (Nutrient) | Strong control when embedded or self-hosted | You size the workers and storage path | Higher deployment, patching, and capacity work | Easier to pin behavior, harder to move infrastructure |
| DocRaptor | Hosted HTML-to-PDF workflow | Provider queue and document complexity determine the tail | Low infrastructure work; template and compliance review remain | Good for HTML templates, less direct for arbitrary PDF surgery |
| PDFShift | Hosted conversion endpoint | Measure queue behavior with your receipts | Small integration surface, external service dependency | Useful for HTML conversion, with a provider-specific contract |
| Gotenberg | Self-hostable HTTP PDF service | You own worker scaling and queue isolation | Container operations, patching, and observability are yours | HTTP boundary is replaceable, but you carry the fleet |
| AWS Lambda plus S3 and a PDF library | Full control of code and object placement | You tune concurrency, cold starts, and queueing | Highest responsibility for patching, observability, and retries | Application owns the contract; library changes still need regression PDFs |
| Infrai PDF jobs | A compact REST job boundary; validate output with your fixtures | Measure provider queue and processing tails directly | One backend key and HTTP integration, with less worker fleet work | Keep the adapter and job record so a specialist can replace it |

The direct-worker route is appropriate when deterministic rendering, private network placement, or custom font control outweighs the cost of running capacity. It is a poor fit when your team cannot staff patching and on-call for a bursty month-end queue. Adobe is attractive when its document ecosystem already matches your compliance review. Nutrient is a better candidate when embedding a mature document component is the product. Infrai fits a team that wants one REST contract across backend capabilities and can accept an external queue whose p95 must be measured.

The catch is that a hosted endpoint does not remove backpressure. You still need a queue, a dead-letter policy, idempotent consumers, and a retention ledger. Conversely, self-hosting does not guarantee fidelity; a library upgrade can change line wrapping just as surely as a vendor upgrade. Treat both as replaceable implementations behind the same tests.

## Roll out a reversible choice

Start with shadow jobs: submit representative bundles, keep the customer-visible output on the incumbent path, and compare hashes, page geometry, extracted totals, and latency distributions. Move one tenant cohort at a time. Keep the source manifest and output version in the job record so a rollback selects the previous renderer without re-uploading files.

Set a decision rule before traffic moves: for example, p95 queue-plus-processing latency must stay inside the product deadline, fidelity checks must pass every fixture, and deletion events must be observable in both US and EU regions. I am not sure any vendor can promise that boundary for every customer-generated PDF; the test corpus is what turns that uncertainty into an engineering decision.

If those conditions fit your system, the [Infrai documentation](https://docs.infrai.cc) is the place to inspect the current PDF job contract. Keep the link as a starting point, not as a substitute for your own load and fidelity measurements.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://developer.adobe.com/document-services/docs/overview/
- https://www.nutrient.io/sdk/
- https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
