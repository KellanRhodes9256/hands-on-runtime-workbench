# Tenant-Isolated Private Browser Uploads: Signed URLs, CORS, and European Limits

Short answer: for per-tenant customer-support backups, keep the bucket private, issue a short-lived upload capability from the application backend, bind every object key to the authenticated tenant and an immutable snapshot ID, and treat the browser's successful upload as provisional until the backend verifies the object. Supabase Storage is often the shortest integration when application authorization already lives in Postgres row-level security; Amazon S3 or Cloudflare R2 presents a separate, S3-compatible storage boundary. None of those paths removes CORS, restore authorization, or European residency work.

The operational constraint changes the choice: a support agent must be able to restore one selected snapshot without gaining a way to enumerate another tenant's files. Upload convenience matters, but isolation is the decision axis. A direct browser upload only removes application servers from the byte path; it doesn't remove them from the authority path.

Decision status: accepted as an architecture pattern, with provider selection left to the deployment team.

## What should private React browser uploads and Next.js restores guarantee?

The backend should mint capabilities, while object storage should carry bytes. For each backup, the backend creates an immutable `snapshot_id`, derives an object key under the authenticated tenant's namespace, and returns a narrowly scoped upload instruction. The browser may upload only to that key. A separate completion request causes the backend to inspect metadata and record the snapshot as restorable; a client-supplied `tenant_id`, bucket name, or arbitrary key never becomes authoritative.

This decision has four invariants:

1. A tenant identity comes from the authenticated server-side session, not a form field.
2. An upload capability names one object key, one method, and a short expiration; it is not a reusable storage credential.
3. Restore authorization is evaluated again when a signed download URL is issued. Possession of an old application URL is not proof of current access.
4. A snapshot becomes selectable only after server-side verification ties the stored object to the pending database record.

The third invariant is easy to miss. Signed URLs are bearer capabilities, so anyone who obtains one can generally use it until it expires within the signature's conditions. An offboarded support agent must not be able to ask the application for a fresh restore URL. Amazon documents that a presigned URL uses the permissions of its creator and can be reused until expiration; Supabase describes signed URLs as fixed-time access to a private asset; R2 describes the same bearer-token property for its S3-compatible presigned URLs.

CORS is a browser admission rule, not tenant authorization. Configure the exact production origins, upload methods, and required headers, then keep authorization in the signature and application records. A permissive CORS response does not make a private object public by itself, but it broadens which browser origins may attempt signed operations and makes accidental capability leakage more useful.

## Rollout ownership starts with the snapshot record

React does not need storage credentials, and a Next.js route should not proxy the backup body merely to prove it participated. The UI first asks the application for an upload capability. The application authenticates the user, resolves the tenant from its own session, creates the pending snapshot, and returns the signed target. After the browser uploads directly, it sends only the opaque snapshot ID back for completion. The server reconstructs the expected object key from its database rather than trusting a key echoed by the browser.

Keep those steps separate.

For a customer-support export, a key such as `tenants/{tenant_uuid}/snapshots/{snapshot_uuid}/archive.bin` makes the isolation boundary reviewable, but the prefix is organization rather than authorization. Bucket policy, row-level policy, or the signing service must still prevent cross-prefix access. Random UUIDs reduce accidental collisions; they don't grant access control. The restore table should store the tenant, snapshot state, expected size or checksum when the workflow supplies one, object key, creation time, and retention deadline. Human-readable customer names belong in application metadata, not keys that may appear in logs.

The easiest integration depends on where authority already lives. Supabase Storage integrates access control with Postgres row-level security, and its documentation states that uploads require the relevant `INSERT` policy on `storage.objects`; that can avoid a second policy language in an application already centered on Supabase Auth and Postgres. S3 presigned requests keep authorization in an AWS identity and bucket-policy model. R2 exposes an S3-compatible API and presigned URL flow, which fits teams already operating S3 tooling. These are integration differences, not proof that one service provides stronger tenant isolation by default.

| Option | Where upload authority is expressed | Browser-upload fit | Application-owned boundary |
|---|---|---|---|
| Supabase Storage | Storage policies backed by Postgres row-level security, or a server-issued signed upload URL | Direct fit when tenant membership uses the same authorization model | Policy coverage, snapshot records, restore authorization, and CORS |
| Amazon S3 | IAM credentials sign a time-limited request; bucket policy can add constraints | Presigned request path with region-specific endpoints | Tenant-to-key mapping, signer permissions, completion checks, lifecycle, and CORS |
| Cloudflare R2 | S3-compatible credentials sign a time-limited request | S3-compatible presigned path for browser uploads | Tenant-to-key mapping, signer scope, completion checks, lifecycle, and CORS |
| Google Cloud Storage | IAM-backed signed URLs or resumable upload sessions | Separate signed-upload boundary in a Google Cloud estate | Application authorization, snapshot state, retention, and CORS |

I'm not sure which option is easiest for a particular team without seeing its identity source, compliance contract, upload-size distribution, and on-call ownership. Those facts resolve the question; a feature checklist doesn't.

## Code: a capability handoff in Python

The following Python is deliberately provider-neutral. `ObjectStore` is the narrow adapter boundary; implementations may use a provider SDK internally, but the application workflow never accepts a caller-selected key. The important part is the state transition and repeated tenant check, not a particular signing library.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol
from uuid import UUID, uuid4


@dataclass(frozen=True)
class UploadCapability:
    snapshot_id: UUID
    url: str
    method: str
    required_headers: dict[str, str]
    expires_at: datetime


@dataclass(frozen=True)
class StoredObject:
    size: int
    checksum: str | None


class ObjectStore(Protocol):
    def sign_put(
        self, *, key: str, content_type: str, expires_in: timedelta
    ) -> tuple[str, dict[str, str]]: ...

    def inspect(self, *, key: str) -> StoredObject: ...

    def sign_get(self, *, key: str, expires_in: timedelta) -> str: ...


def snapshot_key(tenant_id: UUID, snapshot_id: UUID) -> str:
    return f"tenants/{tenant_id}/snapshots/{snapshot_id}/archive.bin"


def begin_upload(
    *, tenant_id: UUID, content_type: str, store: ObjectStore, snapshots
) -> UploadCapability:
    snapshot_id = uuid4()
    key = snapshot_key(tenant_id, snapshot_id)
    snapshots.insert_pending(
        tenant_id=tenant_id, snapshot_id=snapshot_id, object_key=key
    )
    ttl = timedelta(minutes=10)
    url, headers = store.sign_put(
        key=key, content_type=content_type, expires_in=ttl
    )
    return UploadCapability(
        snapshot_id=snapshot_id,
        url=url,
        method="PUT",
        required_headers=headers,
        expires_at=datetime.now(timezone.utc) + ttl,
    )


def complete_upload(
    *, tenant_id: UUID, snapshot_id: UUID, store: ObjectStore, snapshots
) -> None:
    pending = snapshots.get_pending(
        tenant_id=tenant_id, snapshot_id=snapshot_id
    )
    stored = store.inspect(key=pending.object_key)
    if stored.size <= 0:
        raise ValueError("backup object is empty")
    snapshots.mark_ready(
        tenant_id=tenant_id,
        snapshot_id=snapshot_id,
        size=stored.size,
        checksum=stored.checksum,
    )


def create_restore_url(
    *, tenant_id: UUID, snapshot_id: UUID, store: ObjectStore, snapshots
) -> str:
    ready = snapshots.get_ready(tenant_id=tenant_id, snapshot_id=snapshot_id)
    return store.sign_get(key=ready.object_key, expires_in=timedelta(minutes=5))
```

A real adapter must sign the same method and headers the browser will send. If `Content-Type` participates in the signature, the browser must preserve that value; adding or changing signed headers later can invalidate the request. The UI should distinguish three states: capability creation failed, object transfer failed, and completion verification failed. Only the last two can leave a pending snapshot, and a scheduled reconciler can inspect or expire pending records without making them restorable. Don't mark a backup ready from the browser's `200` alone.

For larger archives, multipart or resumable upload changes the transfer protocol but not the authority model. The backend should initiate a bounded upload, associate its opaque upload identifier with the pending snapshot, authorize individual parts without disclosing long-lived credentials, and finalize only the known upload. Limits and expiry behavior differ by provider and client path, so test the chosen mechanism with the actual archive-size distribution rather than assuming a single `PUT` is adequate.

## Test isolation, residency, and restorability

The most dangerous failure is a restore that selects by `snapshot_id` without also constraining by `tenant_id`. A UUID looks unguessable, yet authorization cannot depend on guessing resistance. Make the database lookup composite, exercise it with two tenants in every integration suite, and assert that neither upload completion nor restore signing crosses the boundary. Also test a user who loses tenant membership between upload and restore; the later restore request must use current membership.

Then test the browser boundary in a deployed preview that uses the real origin and headers. Local same-origin mocks hide preflight behavior. Cover an allowed origin, a denied origin, an expired capability, a changed content type, an interrupted upload, a duplicate completion call, an empty object, and a retention deletion racing with restore selection. Log tenant ID, snapshot ID, object key, operation, signing expiry, and provider request ID, while excluding signed query strings because logs containing a live signed URL become an access path.

Europe requires a written claim narrower than "EU hosted." Record the selected storage region or jurisdiction control, replication behavior, backup and log locations, subprocessors, and the legal terms the organization relies on. S3 exposes region selection and lifecycle configuration; R2 documents location hints and jurisdiction restrictions; Supabase projects select a region; Google Cloud Storage documents bucket locations. Those controls are not interchangeable, and a UI label does not establish the full data path. Your mileage may vary with enterprise contracts, so compliance owners should approve the documented path rather than a developer inferring residency from latency.

Durability deserves the same skepticism. A successful upload response is evidence of provider acceptance under that service's contract, not proof that the application can restore the right tenant's snapshot months later. Run periodic restore drills: select an immutable snapshot through the normal authorization path, download it, verify its recorded checksum when available, parse the archive, and record the result. Lifecycle rules should expire abandoned pending objects and enforce the retention policy, but deletion schedules must be tested against legal holds and restore windows. AWS documents that lifecycle actions are asynchronous, which is a good reason not to present a scheduled expiration timestamp as an exact deletion instant.

Fast feedback matters.

One cross-tenant denial test and one restore drill reveal more about this design than a long list of storage features.

## Failure modes of byte proxying

We rejected routing every backup byte through the Next.js application server for this workload. It couples transfer duration, body-size limits, memory pressure, and deployment-region bandwidth to an authorization service that only needs to grant a bounded capability; it also adds another place where a partial upload can be mistaken for a completed snapshot. The catch is that direct upload increases the number of states the UI and backend must reconcile, and browser CORS configuration becomes part of deployment.

The proxy remains valid when files must be transformed, scanned synchronously before storage accepts them, or inspected under a network policy that forbids direct browser access to the object endpoint. Stick with a server-mediated stream when those controls outweigh the extra data hop. Some organizations place tenant membership and object policy under one database team; others assign signing identities and buckets to an infrastructure team. Those ownership facts change the adapter boundary without changing the isolation invariants.

No provider choice repairs a weak tenant key.

The accepted design is the capability pattern plus a composite application authorization check; the storage service remains an adapter selected against documented organizational constraints.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://supabase.com/docs/guides/storage/security/access-control
- https://supabase.com/docs/guides/storage/serving/downloads
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://developers.cloudflare.com/r2/buckets/data-location/
- https://cloud.google.com/storage/docs/access-control/signed-urls
- https://cloud.google.com/storage/docs/locations
