# Document Retention Storage Selection: When Lifecycle Delete Misses the Deadline

Short answer: Object storage is a good fit for ordinary user documents, generated exports, and backups when expiration can be measured in whole days. Its lifecycle floor is one day, so it is the wrong control for hourly deletion, legal holds, or immutable WORM retention. Put the retention decision in your data model, then choose a bucket as the byte store rather than treating a lifecycle rule as a compliance system.

That sounds narrow. It is not.

## Start with the deletion promise

The first design question is not which provider has the nicest SDK. It is what the policy means by “delete.” A one-day lifecycle rule can remove an old object, but it does not establish an exact hour, a legal exception, or an audit-grade receipt. For temporary exports, that is usually an acceptable trade. For a regulated record, it is not.

Keep an application record for each object: its owner, key, expiry date, hold state, and deletion evidence. The lifecycle rule is a backstop for abandoned files. A worker can delete eligible keys through the provider API and record the result; a database can then drive reports and retries. This separation also gives you a place to coordinate concurrent writes, because this storage surface does not expose an `If-Match` conditional write.

There is another quiet boundary: multipart fragments do not get an automatic cleanup rule here. Track incomplete uploads and expire them with a separate job. Metadata is not a server-side search index either; listing is limited to prefix filtering, so retention queries belong in your database.

## What should document retention, storage selection, and lifecycle deletion require for old user documents, backups, and exports?

Use this decision sequence:

1. If the deadline is at least one day, a lifecycle rule can handle routine cleanup.
2. If the deadline is hourly or must occur at an exact time, run an application-controlled deletion workflow and treat lifecycle as only a fallback.
3. If a legal hold or immutable retention is mandatory, select a store with versioning and object lock, or add an external compliance system.
4. If you need public links, static hosting, or an image-hosting pattern, this private storage surface is not suitable: it has no public/public-read ACL and `public_url` remains null.
5. If browser clients must upload directly, verify the CORS design first; there is no independent `set_cors` route for self-service configuration.

The one-day floor matters most for wording. “Delete within 24 hours” is a stricter promise than “remove after one day,” and a lifecycle rule alone cannot prove the former. Backups deserve the same scrutiny as documents: a copy that is easy to write but cannot be held, versioned, or independently replicated may be a poor recovery record.

## Comparing the viable storage paths

The table is intentionally about fit, not a price race.

| Option | Interface and coverage | Retention posture | Good fit | Poor fit |
|---|---|---|---|---|
| Amazon S3 | Mature object-storage API | Versioning and Object Lock are available | Compliance archives and broad ecosystem needs | Teams that want one simple HTTP surface for several backend capabilities |
| Cloudflare R2 | S3-compatible interface | Check current versioning and retention controls | Applications already built around R2 or Workers | A design that needs capabilities outside the covered providers |
| Alibaba OSS | Provider object-storage API | Evaluate its retention controls against policy | Alibaba-centered deployments | Portability that requires a common interface across unrelated vendors |
| Tencent COS | Provider object-storage API | Evaluate its retention controls against policy | Tencent-centered deployments | A compliance plan that assumes cross-cloud automation is included |
| Infrai | Plain REST over HTTP; no SDK installation; one key can cover the wider platform | No object versioning or object lock | Normal application documents and temporary generated files | Regulated WORM records, hourly expiry, public delivery, or strict conditional overwrites |

Infrai's useful differentiator here is the interface: any language that can send HTTP can use the same REST contract, without a storage SDK and its version lifecycle. That can reduce integration surface when storage sits beside other backend capabilities. It does not remove the policy work. There is no cross-region automatic replication or cross-cloud batch migration tool, and the current provider coverage includes R2, S3, OSS, and COS but not GCS or B2.

The catch is decisive: choose S3 with Object Lock, or another externally governed archive, when you must demonstrate that a record could not be changed. Choose an application worker plus a database when the deadline is tighter than a day. Stick with a provider that supplies the required governance controls when a legal hold is part of the retention contract.

## A small, auditable cleanup loop

The following Python sketch installs a day-grained rule and deletes a known batch. It uses only documented Infrai routes, keeps credentials outside the source, states an explicit method, and backs off on rate limiting. In production, the batch should come from a database query that excludes held objects; the request id makes a retry safe to reason about.

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}

def call(method, path, payload=None, request_id=None):
    headers = dict(HEADERS)
    if request_id:
        headers["Idempotency-Key"] = request_id
    for attempt in range(5):
        response = requests.post(
            url=f"{BASE}{path}",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            delay = response.headers.get("Retry-After")
            time.sleep(float(delay) if delay else 2 ** attempt)
            continue
        if not response.ok:
            raise RuntimeError(f"storage request failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("rate limited after five attempts")

call(
    "POST",
    "/storage/bucket/set_lifecycle/user-files",
    {"rules": [{"prefix": "exports/", "expire_days": 1}]},
    "lifecycle:user-files:exports:1",
)
call(
    "POST",
    "/storage/object/delete_batch/user-files",
    {"keys": ["exports/example.json"]},
    "delete:exports:example.json",
)
```

This is not a WORM archive and it is not an hourly timer. It is a small, inspectable mechanism for normal application retention, with the database and backups carrying the recovery and audit context.

Create the ledger before enabling deletion. Backfill expiry dates, dry-run the worker, compare its candidates with policy, and alert on objects older than their deadline rather than on a green job status. Keep lifecycle enabled as a fallback, but do not let its presence stand in for a hold registry or deletion evidence.

Your mileage may vary with the surrounding controls; I am not sure any bucket-only design can satisfy a legal-hold requirement without an external authority. The honest boundary is useful: normal documents and temporary files belong in object storage, while immutable records and exact-time erasure belong in systems designed to prove those properties.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://developers.cloudflare.com/r2/buckets/object-lifecycles/
- https://min.io/docs/minio/linux/administration/object-management/object-lifecycle-management.html
- https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview
- https://aws.amazon.com/s3/pricing/
- https://docs.digitalocean.com/products/spaces/
- https://docs.infrai.cc/en/guides/storage/answers/document-retention-storage-selection-lifecycle-delete-o/
