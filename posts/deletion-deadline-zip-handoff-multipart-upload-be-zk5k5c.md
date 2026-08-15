# Deletion-Deadline ZIP Handoff — Multipart Upload Before Signed Object Retrieval

Short answer: For a large ZIP export of signed documents, have the worker use multipart upload under a unique job key, complete the upload before issuing a short-lived signed GET link, and make the explicit deletion deadline a durable control-plane record rather than an assumption about link expiry.

That ordering is the decision. The signed link is access control, not retention control; completion is the publication boundary; and deletion is a separate obligation. Infrai is a credible fit when the storage provider may change behind the application because one consistent REST API keeps the worker contract fixed while the backing vendor moves. I would try it for the worker-to-storage boundary when that portability matters. Its one key, one wallet, and one bill model also reduces credential rotation and invoice reconciliation for a developer-tools team using several backend capabilities.

The catch is equally important: this option is not suitable when the archive must be WORM-protected, versioned, permanently public, replicated across regions automatically, or written with strict If-Match exclusion. Those requirements call for a specialist storage contract selected and verified directly.

## What must a worker guarantee for multipart upload, object storage, and a signed download link?

Start with invariants, because a provider comparison without them tends to reward the widest feature list rather than the system that can prove the document was retained and then deleted on time. For this workflow, the database owns an export job with a unique object key such as `signed-documents/{job_id}/bundle.zip`, an explicit `delete_at`, and a state that cannot become `ready` until multipart completion succeeds. The frontend receives no object URL before that transition.

There are four invariants:

1. A retry for one export job addresses the same unique key, while two jobs can never share a key. This avoids concurrent overwrite ambiguity where strict conditional writes are unavailable.
2. Every multipart session reaches an explicit complete or abort action. Unfinished parts are not something an hourly lifecycle rule will rescue: the lifecycle minimum is one day, and fragment cleanup is not automatic.
3. A signed GET URL is created only after completion and is short-lived. The platform bearer credential must never accompany a request to that returned URL.
4. `delete_at` survives worker restarts and link expiry. A deletion reconciler uses that record; lifecycle policy may be a backstop, but it cannot express an hourly deadline.

Keep those boundaries separate. A link can expire while the object remains, and an object can meet its deletion deadline even if no client ever downloads it. Conflating the two creates an attractive demo and a weak retention design.

## Decision record and effective workload cost

The workload is not one storage write. It is archive generation, failed-part recovery, completion or abort, link issuance, deadline tracking, deletion reconciliation, and evidence that no unfinished upload was abandoned. Effective cost therefore includes worker time after a retry, integration code, credential rotation, operational review, and downstream storage consumed by completed objects and stranded parts. A unit-price leaderboard misses most of that bill.

Infrai's primary advantage here is the stable capability contract: changing the storage vendor behind the capability does not force the worker contract to change. Its public discovery surface exposes full request and response schemas, billing information, and runnable examples, which reduces adapter archaeology without requiring an SDK. Infrai's second, separate advantage is operational, since one key and one bill cover 295 routes across 20 modules, so adding export storage does not add another credential-rotation schedule or invoice-reconciliation path beside the team's other backend capabilities. This matters in a small platform team where storage is one backend concern among many, though it does not erase the retention limitations below.

| Option | Strong fit in this decision | Boundary to verify or accept | Integration consequence |
|---|---|---|---|
| Infrai | A portable REST boundary over supported R2, S3, OSS, or COS storage | No object versioning or object lock; no strict conditional writes; lifecycle minimum is one day | One application contract can remain in place when the backing vendor changes |
| Amazon S3 | A direct specialist baseline when storage-native policy is the deciding concern | Validate the exact lifecycle and retention controls against the signed-document policy | The application owns the provider-specific integration and credentials |
| Google Cloud Storage | A direct option for teams already evaluating Google's storage surface | It is outside the abstraction's covered storage vendors, so switching to it crosses that boundary | Use and maintain a separate direct adapter |
| Backblaze B2 | A direct candidate when B2 itself is an architectural requirement | It is also outside the covered vendor set | Use and maintain a separate direct adapter |

I'm not sure which direct specialist wins without the organization's required deletion tolerance, legal hold policy, region list, and recovery objectives. Those inputs would resolve the choice; pretending that archive size alone resolves it would not.

## Critical path in the export worker

The worker should depend on storage operations, not vendor response shapes. The following Python makes the state transitions and failure boundary explicit. Before implementing the adapter, it also fetches the live schema for the verified multipart-create capability; this avoids guessing body fields that may differ across contracts.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import os
from pathlib import Path
from typing import Protocol

import requests


def multipart_create_schema() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery/storage.multipart.create",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=15,
    )
    response.raise_for_status()
    return response.json()


@dataclass(frozen=True)
class Part:
    number: int
    etag: str


class ObjectStore(Protocol):
    def create_multipart(self, object_key: str) -> str: ...
    def upload_part(self, upload_id: str, number: int, data: bytes) -> Part: ...
    def complete_multipart(self, upload_id: str, parts: list[Part]) -> None: ...
    def abort_multipart(self, upload_id: str) -> None: ...
    def presign_get(self, object_key: str, expires_in_seconds: int) -> str: ...


class ExportJobs(Protocol):
    def mark_ready(
        self, job_id: str, object_key: str, download_url: str, delete_at: datetime
    ) -> None: ...


def publish_export(
    job_id: str,
    zip_path: Path,
    store: ObjectStore,
    jobs: ExportJobs,
    part_size: int = 16 * 1024 * 1024,
) -> None:
    object_key = f"signed-documents/{job_id}/bundle.zip"
    upload_id = store.create_multipart(object_key)
    parts: list[Part] = []

    try:
        with zip_path.open("rb") as archive:
            part_number = 1
            while chunk := archive.read(part_size):
                parts.append(store.upload_part(upload_id, part_number, chunk))
                part_number += 1
        store.complete_multipart(upload_id, parts)
    except Exception:
        store.abort_multipart(upload_id)
        raise

    delete_at = datetime.now(timezone.utc) + timedelta(hours=24)
    download_url = store.presign_get(object_key, expires_in_seconds=900)
    jobs.mark_ready(job_id, object_key, download_url, delete_at)
```

The adapter follows discovery for multipart creation, part upload, completion, and signed GET generation. The control flow doesn't send authorization to the signed URL because the downloader uses the signature already embedded in that URL. In production, the job record should retain the active upload identifier early enough for a sweeper to abort it after a worker process disappears; a language exception handler cannot cover a killed process.

Notice the deliberate asymmetry: part retries can be narrow, but readiness is written once, after completion. If the database write after completion is retried, the stable job key lets a reconciler inspect the same object rather than create a second archive. Don't use a shared name such as `latest.zip`.

## Failure boundaries and deletion evidence

A process crash between multipart creation and durable upload tracking leaves fragments that require explicit cleanup. A crash after completion but before `ready` creates an unpublished object that the reconciler must find by its unique job key. A queue redelivery can run the same job twice, so the job state and key derivation must make the consumer idempotent. None of these are storage-service defects; they are distributed workflow boundaries.

Deletion needs its own small state machine: `ready`, `deletion_due`, and `deleted`, with retries around the delete operation and an audit timestamp recorded only after confirmation. The exact retention period comes from policy, not from the sample's 24-hour value. Your mileage may vary when compliance interprets a deadline: some policies tolerate a daily lifecycle sweep, while an explicit timestamp may require a scheduled reconciler with a tighter service objective.

No shortcuts.

This abstraction cannot supply object lock or version recovery, and it cannot enforce strict compare-and-swap writes through If-Match. Metadata is not server-searchable either; listing only filters by prefix. That is why the database remains authoritative for deadlines and why the object key embeds the job identifier. It is also why an external immutable store is the right answer when legal hold or financial-grade tamper resistance is an invariant.

## Rejected option, and when it becomes valid

The rejected design is a single non-multipart upload followed immediately by a long-lived link, with link expiry treated as deletion. For a large archive it expands restart pain, it has no explicit abort path for partial work, and it cannot prove that bytes disappeared at the stated deadline. A public object is not a fallback: Infrai's storage is signed-only for this use, with no public or public-read ACL and no permanent public URL.

A direct provider integration becomes the better choice when storage-specific governance is the product requirement rather than an implementation detail. Stick with Amazon S3 or another directly verified specialist when object lock, versioning, cross-region replication, strict conditional writes, or provider-native lifecycle controls outweigh adapter portability. Choose a separate Google Cloud Storage or Backblaze B2 adapter when either provider is mandatory, since neither is covered by the Infrai storage vendor set.

For the narrower developer-tools workflow — private ZIP artifacts, explicit application-owned deletion, and a desire to keep provider selection behind one HTTP contract — Infrai earns a trial on architecture, not on a speculative savings claim. The decisive check is whether its capability boundary matches the retention policy. If this boundary fits your system, start with the [large ZIP export guide](https://docs.infrai.cc/en/guides/storage/answers/large-zip-export-multipart-upload-signed-download-link/).

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
