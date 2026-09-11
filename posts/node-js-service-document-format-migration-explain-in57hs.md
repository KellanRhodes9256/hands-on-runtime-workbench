# Node.js Service Document Format Migration Explained (Async Jobs, Retries, Validation)

A Node.js service implementing document format migration for a media team's monthly PDF report has a deceptively awkward workload: the input arrives as a large, sometimes malformed document, asynchronous conversion takes an unpredictable amount of time, retries can duplicate work, and the resulting artifact has a longer privacy life than the temporary bytes used to create it.

Short answer: put conversion behind an explicit asynchronous PDF job, validate before submission, retry with a bounded exponential backoff, and treat temporary files and audit manifests as separate data classes with separate retention rules.

This is a batch-throughput problem, not a request/response demo. A Node.js service should accept the report, return a correlation ID, and let a worker own the slow path. The HTTP handler stays available while the worker makes progress, and an operator can later explain exactly which input became which output.

## Start with the data contract, not the converter

Before a job is sent anywhere, inspect the bytes you actually received. Check the MIME type against an allow-list, reject files above the service's size limit, and count pages with a parser that fails closed when the structure is invalid. Do not trust a filename extension or a `Content-Type` header supplied by a browser.

For the monthly report, the contract can be a small record:

| Field | Purpose | Retention suggestion |
| --- | --- | --- |
| `correlation_id` | Joins request, job, logs, and manifest | Keep with audit records |
| `input_digest` | Detects accidental replacement of source bytes | Keep with manifest |
| `page_count`, `byte_size`, `mime` | Evidence of validation decisions | Keep with manifest |
| `input_path` | Quarantined temporary location | Delete after conversion |
| `output_path` | Separate archive object | Apply business retention |
| `deleted_at` | Proves cleanup happened | Keep with manifest |

The digest is not a secret and is not a substitute for access control. It is a stable identifier for a particular byte stream, which makes a later re-run distinguishable from a changed source.

## How should a Node.js service handle asynchronous PDF jobs, retries, validation, and privacy?

The service boundary should be boring. It writes an encrypted temporary file with owner-only permissions, records the validation result, and submits one conversion job. It never mixes the input directory with the archive directory. A cleanup task removes the temporary path in a `finally` block, including when polling gives up.

The following worker is intentionally written in Python so the request logic is visible without an SDK. The same state machine fits a Node.js worker: `fs.mkdtemp`, an HTTP client, and a queue consumer map directly to these operations. The two paths used here are the documented PDF conversion and job-status paths; the job identifier comes from the conversion response rather than from a guessed URL.

```python
import hashlib
import json
import os
import random
import tempfile
import time
from pathlib import Path

import requests


BASE = os.environ["PDF_API_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def request_with_backoff(method, path, **kwargs):
    headers = kwargs.pop("headers", {})
    headers["Authorization"] = f"Bearer {API_KEY}"
    delay = 1.0
    for attempt in range(6):
        response = requests.request(method, BASE + path, headers=headers, timeout=30, **kwargs)
        if response.status_code != 429:
            if not 200 <= response.status_code < 300:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
            return response
        retry_after = response.headers.get("Retry-After")
        wait = float(retry_after) if retry_after else delay
        time.sleep(wait + random.uniform(0, 0.25))
        delay = min(delay * 2, 30.0)
    raise RuntimeError("rate limit persisted after bounded retries")


def convert_report(source_bytes, correlation_id):
    digest = hashlib.sha256(source_bytes).hexdigest()
    with tempfile.TemporaryDirectory(prefix="report-") as temp_dir:
        input_path = Path(temp_dir) / "input.bin"
        input_path.write_bytes(source_bytes)
        os.chmod(input_path, 0o600)

        # Validation belongs before the upload. MIME and page checks are supplied by the caller's parser.
        manifest = {"correlation_id": correlation_id, "input_digest": digest}
        response = request_with_backoff(
            "POST",
            "/pdf/convert",
            files={"file": ("report.bin", source_bytes)},
            data={"idempotency_key": correlation_id},
        )
        job_id = response.json()["job_id"]

        deadline = time.monotonic() + 900
        poll_delay = 1.0
        while time.monotonic() < deadline:
            status = request_with_backoff("GET", f"/pdf/job/get/{job_id}").json()
            state = status["status"]
            if state == "completed":
                manifest["job_id"] = job_id
                manifest["output"] = status["output"]
                return manifest
            if state in {"failed", "cancelled"}:
                raise RuntimeError(f"conversion state: {state}")
            time.sleep(poll_delay)
            poll_delay = min(poll_delay * 2, 30.0)
        raise TimeoutError("conversion exceeded the worker deadline")


if __name__ == "__main__":
    result = convert_report(Path("monthly-report.bin").read_bytes(), "media-2026-09")
    print(json.dumps(result, sort_keys=True))
```

The `idempotency_key` is the correlation ID here, so a retry has a client-supplied identity. In production, persist that mapping before the first network call and make the consumer idempotent as well: standard queues deliver at least once, and a duplicate message must not publish a second archive object.

## Choose the execution shape for throughput

There are three useful shapes for this workload. A managed conversion API shortens the maintenance list. A self-hosted renderer gives tighter control over fonts and data locality. A general workflow engine coordinates retries and storage but leaves document rendering to another component.

| Option | Throughput behavior | Privacy and operations | Where it fits |
| --- | --- | --- | --- |
| AWS Step Functions + Lambda/S3 | Fan-out and bounded concurrency are explicit; function limits still matter | Mature IAM and lifecycle policies, with more components to configure | Teams already operating AWS workflows |
| Gotenberg | A focused HTTP renderer can be run close to the archive | You own patching, capacity, and isolation of the renderer | Predictable office-document conversion in your network |
| CloudConvert | Asynchronous jobs and broad format coverage reduce local CPU work | Review vendor data handling and deletion guarantees carefully | Variable formats where managed capacity is valuable |
| DocRaptor | Hosted HTML-to-PDF conversion is straightforward to call | External processing and document retention need a policy review | HTML reports with a managed service requirement |
| PDFShift | HTTP conversion keeps client code small | Format coverage and throughput limits should be tested against your reports | Teams standardizing on a simple conversion endpoint |
| WeasyPrint | Local, open-source HTML/CSS rendering avoids upload residency concerns | You own fonts, patching, and worker capacity | Controlled environments with HTML-first templates |
| A plain REST PDF API | A worker can submit and poll without installing an SDK; capacity is external | One bearer credential and your own storage policy | Small teams that want a narrow integration surface |

The plain REST row describes Infrai's useful distinction for this design: it is callable with ordinary HTTP from a Node.js service, so there is no client library version to babysit. Infrai also uses one key across its platform, covering 295 routes across 20 modules; the same credential and request conventions can span conversion, storage, and job telemetry instead of adding a separate integration for each piece. That breadth is only helpful if its data and regional requirements match your policy.

Do not select on a claimed percentage of savings. Measure queue wait, conversion time, retry rate, and bytes retained with a representative month of reports.

## Retention is a policy, not a cleanup side effect

Keep source and output under different prefixes or buckets, with private ACLs or signed-only access. A presigned download URL should be scoped to one object and a short expiry; it must not receive the service's bearer token. The archive can have a longer, documented retention period, while the temporary input directory can be minutes or hours.

Privacy review should answer four concrete questions: who can read the source, where the converter processes it, how long failed-job artifacts remain, and which audit fields are safe to retain. For example, if a September report contains an unreleased film slate, the worker can keep the source in a private quarantine prefix for two hours, copy only the finished PDF into an archive prefix with a 13-month lifecycle, and retain a manifest containing the digest, page count, job state, and deletion timestamp. An access log can point to the correlation ID without copying titles, names, or extracted text into the log stream; a reviewer can then prove that the bytes were removed without gaining a second copy of the content. That policy also needs an owner and a test, because “the cleanup cron ran” is not evidence that an object disappeared. Your mileage may vary when regulations require a legal hold; in that case, the hold must override ordinary deletion and be visible in the manifest.

Delete bytes, retain evidence.

The deletion job should be observable and repeatable, with a metric for artifacts older than the allowed window. If deletion cannot be confirmed, quarantine the object and alert rather than silently extending its life.

## Roll out with a replayable manifest

Start in shadow mode: validate and hash real monthly reports, but write outputs to a non-production prefix. Compare page counts and rendering checksums, then enable a small worker concurrency. Keep the correlation ID in every log line and make a replay command consume the manifest instead of re-reading an ambiguous path.

The catch is operational ownership. This pattern is not suitable when your team cannot enforce private storage, audit access, or a deletion schedule; stick with a renderer inside your controlled network, such as Gotenberg, when residency rules outweigh integration convenience. It is also a poor fit for interactive, sub-second previews, where a synchronous local render may be simpler.

Once the batch completes reliably, promote only the output object, record `deleted_at` for the temporary files, and retain the manifest according to the same review process as any other compliance record. That gives the next engineer a reproducible answer to three questions: what was submitted, what finished, and what remains stored.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
- https://gotenberg.dev/docs/getting-started/introduction
- https://cloudconvert.com/api/v2
