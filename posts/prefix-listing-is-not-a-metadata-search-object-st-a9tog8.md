# Prefix Listing Is Not a Metadata Search: Object Storage Keys for Tenant Image Exports

The constraint that decides this design tends to arrive late, usually the first time a customer on a developer-tools platform asks for their data back: a tenant-scoped export has to enumerate one tenant's product images and nothing else, and object storage gives you exactly one server-side selector for that job — the key prefix. Content type, custom metadata and tags ride along in the list response; they don't narrow it. Bottom line: encode tenancy in the key, keep the facts you will filter on (owner, MIME type, dimensions, upload time) in a database you can index, and treat every list call as a paid range scan rather than a search.

The split is unglamorous. It also survives a provider migration, which is more than most clever schemes manage.

## The invariants a tenant export has to hold

Write these down before arguing about layout, because they are what the layout is for. No object belonging to another tenant may appear in the archive, and no credential minted during the export may be replayable against a neighbour's key. The export must be explainable after the fact: which objects, at what version, with what declared type. Enumeration cost has to scale with the tenant's object count, not the bucket's — a 40-tenant bucket and a 40,000-tenant bucket should cost the same per export.

Four failure modes follow directly from those invariants, and I have yet to see a design that dodges all four for free.

The first is the full-bucket scan, where somebody implements "find this tenant's images" as a listing over the whole namespace with a client-side filter on content type, and the job's runtime and transaction bill grow with total objects stored. The second is the torn snapshot: a paginated walk that takes minutes is not a point-in-time view, so an image uploaded or deleted mid-walk may or may not land in the archive, and neither outcome is wrong from the storage layer's perspective. The third is cross-tenant enumeration, which only bites when keys are guessable and served raw — sequential SKUs are a gift to anyone with a for-loop. The fourth is trusting the stored content type, and it's the one most teams wave off.

RFC 9110 is explicit that Content-Type is the sender's declaration about the representation it is sending; a recipient that receives no type may assume `application/octet-stream` or examine the data itself. Object storage stores what your upload endpoint handed it. So "search avatars and product images by content type" over stored headers means searching a value that a client asserted at upload time, not a verified property of the bytes — validate on ingest, store the validated type in the row, and let the header be a delivery detail.

## Can you search object storage by content type or tags, or only list by prefix?

The list operation takes a prefix, an optional delimiter, a continuation token and a page cap — S3's ListObjectsV2 returns at most 1,000 keys per response, and other implementations page similarly. A prefix is a lexicographic range over the key namespace. There's no field in that request that says "content type equals image/webp", and there is no index behind the call that could answer it.

Tags exist, and they are genuinely useful — just not for this. S3 allows up to 10 tags per object, and they drive lifecycle rules, replication filters and IAM conditions; the list API still doesn't filter on them. Azure Blob Storage is the one mainstream exception worth naming: blob index tags support a separate find-by-tags query over up to 10 index tags per blob, with the index maintained asynchronously, so a tag written a second ago may not be queryable yet. Google Cloud Storage added glob matching to its object listing, which makes key-shaped questions easier to express and still answers nothing about metadata.

The portable conclusion is unchanged across all of them.

Anything you need to filter, sort, paginate or join lives in a database; the bucket answers exact-key reads and prefix-bounded walks. Scheduled inventory reports — daily or weekly manifests of a bucket's contents — are a real third option for reconciliation, but their latency makes them a background input, not an interactive query. And the walks are not free: list calls are billed transactions on most providers, which is easy to miss until an hourly reconciliation job over a large bucket shows up as a line item.

## Key layout is an access-control decision, not a naming convention

Here is where access control and delivery simplicity actually collide, and the axis is sharper than the usual naming-convention debate.

| Layout | Server-side selector | Access control | Delivery | Where it breaks |
| --- | --- | --- | --- | --- |
| `t/{tenant}/product/{sku}/{image_id}.webp` | Prefix walk per tenant | Policy conditions on the prefix; one boundary to reason about | Per-object signed URLs, cache keys fragmented per tenant | Re-parenting an image means copy plus delete, which is not atomic |
| `blob/{sha256[0:2]}/{sha256}` | None that carries tenancy | Entirely in the application layer | Immutable, dedupable, trivially cacheable | Export and deletion both require the index; shared bytes need refcounting |
| One bucket per tenant | Bucket-level everything | Hardest isolation available | Same as prefix layout, plus N configurations to keep aligned | Bucket count is an account quota, and config drifts per bucket |
| Flat keys plus object tags | Tag query, on backends that offer one | Tag-based conditions, where supported | Unaffected | Asynchronous indexing and a tag-count ceiling; not portable |

The decision rule I would put in the record: if the images are private, per-tenant, and you owe customers export and deletion on request, tenancy belongs in the prefix, because a bounded walk is the only cheap proof you can enumerate exactly what you promised. If the images are public, hot, and identical across tenants — icons, badges, a component gallery — content addressing wins, and delivery becomes a CDN problem instead of a signing problem.

Both layouts still need the database. That part isn't negotiable.

## The critical path, driven by a manifest rather than a walk

The export reads rows, then reads objects by exact key. The prefix walk appears exactly once in this system, and not here.

```python
import hashlib
import io
import os
import tarfile

import boto3

S3 = boto3.client("s3", endpoint_url=os.environ["OBJECT_STORE_ENDPOINT"])  # any S3-compatible endpoint
BUCKET = os.environ["IMAGE_BUCKET"]


def export_tenant_images(db, tenant_id, out):
    """Stream one tenant's product images into a tar, ordered and checksummed."""
    rows = db.execute(
        "SELECT object_key, sku, content_type, sha256 FROM product_image "
        "WHERE tenant_id = %s AND deleted_at IS NULL ORDER BY sku, object_key",
        (tenant_id,),
    )
    manifest = []
    with tarfile.open(fileobj=out, mode="w|") as tar:
        for key, sku, content_type, digest in rows:
            if not key.startswith("t/%s/" % tenant_id):
                raise ValueError("row escapes its tenant prefix: %s" % key)   # isolation, checked twice
            body = S3.get_object(Bucket=BUCKET, Key=key)["Body"].read()
            if hashlib.sha256(body).hexdigest() != digest:
                raise ValueError("stored bytes differ from the recorded digest: %s" % key)
            entry = tarfile.TarInfo(name="%s/%s" % (sku, key.rsplit("/", 1)[-1]))
            entry.size = len(body)
            tar.addfile(entry, io.BytesIO(body))
            manifest.append({"key": key, "sku": sku, "content_type": content_type})
    return {"tenant_id": tenant_id, "objects": len(manifest), "manifest": manifest}
```

Two things in there are deliberate. The prefix assertion is redundant with the query, and I keep it anyway, because a redundant check on the isolation invariant costs nothing and a bad join costs a disclosure. Reading each object fully into memory is fine for product imagery in the low hundreds of kilobytes and wrong for anything larger — swap in a streaming copy once your uploads include source files.

Now the walk, which earns its place in reconciliation:

```python
def orphaned_keys(db, tenant_id):
    """Bytes with no row: the upload committed, the transaction didn't."""
    known = {row[0] for row in db.execute(
        "SELECT object_key FROM product_image WHERE tenant_id = %s", (tenant_id,))}
    orphans, token = [], None
    while True:
        page = S3.list_objects_v2(
            Bucket=BUCKET,
            Prefix="t/%s/" % tenant_id,
            MaxKeys=1000,
            **({"ContinuationToken": token} if token else {}),
        )
        for obj in page.get("Contents", []):
            if obj["Key"] not in known:
                orphans.append(obj["Key"])
        if not page.get("IsTruncated"):
            return orphans
        token = page["NextContinuationToken"]
```

Run it on a schedule, alert on a non-empty result, and resist the temptation to auto-delete: an orphan is as likely to be an in-flight upload as a leak. Observability here is two counters and one gauge — objects exported per run, orphans found, and the age of the oldest orphan — and that is enough to notice a broken upload path within a day. Testing is easier than it looks, too, since the same code runs against a local S3-compatible container in CI with a seeded fixture bucket; the export is deterministic because the row order is, which is the main reason the query carries an explicit `ORDER BY`.

## The option I rejected, and when it is still the right call

A bucket per tenant was the tempting alternative, and I rejected it for this platform. It makes the export trivial and the isolation argument almost free, then charges you for it everywhere else: bucket counts are an account quota rather than an unlimited namespace, and every lifecycle rule, CORS configuration, encryption setting and policy has to be applied N times and kept aligned forever. At a few thousand self-serve tenants that stops being an architecture and starts being a fleet.

Stick with per-tenant buckets when tenant count is small and the boundary is contractual — a dozen enterprise customers with data-residency clauses or customer-managed encryption keys, where the bucket boundary is the compliance boundary you will be audited against. That is a real case, and prefix isolation is a weaker answer to it.

The recommended layout has its own catch, and it is not suitable everywhere. Tenant-prefixed keys are a poor fit when images are served publicly at high volume, because per-tenant paths fragment cache keys and force signing work into the hot path; content addressing is the better tool there. They are also a poor fit when objects legitimately move between tenants, since renaming a key is a copy followed by a delete, with a window in between that no amount of care removes. And if your read path has no database in it at all — a static publishing pipeline, say — then a metadata index in the request path is a dependency you were trying to avoid.

Your mileage may vary on how much reconciliation you actually need; that depends on whether your upload path can commit the row and the object in the same failure domain, which mine cannot. I am not sure there is a clean answer to tenant merges either. Everything else here I would defend in a review.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging.html
- https://learn.microsoft.com/en-us/azure/storage/blobs/storage-manage-find-blobs
- https://cloud.google.com/storage/docs/listing-objects
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html
- https://www.backblaze.com/cloud-storage/pricing
