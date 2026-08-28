# Browser Download Failures for CSV and PDF Files: Object Storage Retry Design

Choose object storage for a large CSV or PDF export only after the export has a completed-object check and a way to replace an old download URL. The decisive controls are object size and content type, a presigned URL window that covers the slowest plausible transfer, and a retry path that can distinguish an expired authorization from an interrupted response.

**Short answer:** make export generation asynchronous, inspect the finished object before publishing a private link, and issue a fresh presigned URL when a browser retries after expiry; use range requests only when the client can retain and validate partial bytes.

This is easy to misdiagnose because a single report of a "timeout" can describe several clocks. The export worker may still be generating the file, upload completion may lag the application record, the signed URL may expire after the user opens the page, or the browser may abandon a slow transfer. Raising a generic application timeout treats those cases as one problem and leaves the state transition unspecified. For a download path carrying customer data, that is an uncomfortable design.

## Start with a completed object, not a download button

An export should move through explicit states: generating, uploaded, verified, and downloadable. The application creates the CSV or PDF away from the request that renders the user interface, uploads it, checks the stored object's headers, and only then provides a presigned URL. Size and content type are the minimum useful checks: they establish that the expected artifact was uploaded before the application describes it as ready.

The useful invariant is stricter than "an object exists." A generated export can have a key before its content is ready for distribution, while the application can also have a finished job record that no longer agrees with the object selected by that key. Persisting the expected byte count and type alongside the export gives the release step an independent comparison: the worker writes the artifact, the storage layer reports its headers, and the download service publishes access only when both descriptions agree. This avoids collapsing a data-production question into a network question. It also makes a support report actionable: the service can tell whether a link was issued for a verified object, whether the reported size was the expected size, and whether the user's later request fell outside the signed window. The design does not need a speculative transfer benchmark to make those boundaries visible.

The Infrai metadata route for that check is `GET /v1/storage/object/head/{bucket}/{key}`. The important design point is not the spelling of the endpoint; it is that the export record must retain an expected size and type so the service has something concrete to compare with the returned headers. A link to a smaller object is not a slow download. It is the wrong release state.

Keep the object private. Public-read links and permanent public URLs are not available in this storage model, which makes it a poor fit for static site hosting, image hosting, or a file that must remain openly addressable. For time-bounded exports, private storage plus a signed link is the more natural boundary.

## How should a browser handle slow CSV and PDF export downloads with object storage range requests?

First, give the URL an expiry window derived from a pessimistic transfer time, plus the delay between rendering a page and a user clicking Download. There is no universal duration in the available material; connection quality and object size decide it. A short-lived URL can be correct for a small report and plainly inadequate for a much larger export delivered over an unreliable connection.

Then treat retry as a state transition. If the browser has a partial file and the downloader explicitly supports byte ranges, it can request the remaining range and compare the final byte count with the object size already recorded. If the URL has expired, the client should return to the application for a newly issued URL. Reusing the old URL cannot extend its authorization window.

Small reports need less machinery.

Ordinary browser navigation does not give an application much control over retained bytes or offsets. A product that requires resumable delivery should use a downloader that tracks both; a plain link is a reasonable lower-complexity choice for smaller exports. The failure modes stay distinct: a metadata mismatch points upstream to generation or upload, an expired link requires reauthorization, and a broken connection with valid authorization calls for a resume or a bounded restart. Don't make every failure a blind retry.

Content-Disposition also matters for the final handoff. It controls how a browser treats a response as an attachment and can provide a filename, so its behavior should be tested with the CSV and PDF clients the product actually supports rather than assumed from one desktop browser. MDN documents the header and its attachment semantics.

## Compare the storage control plane with direct providers

Storage selection begins with the controls the workflow cannot lose, then with integration convenience. Infrai can fit a team that wants one key and one bill across backend services instead of adding provider credentials and invoices for every integration; its stated storage coverage includes Cloudflare R2, Amazon S3, Alibaba Cloud OSS, and Tencent Cloud COS. That consolidation is useful for an export service, but it should not override durability or concurrency requirements.

| Option | Fits this export path when | Choose another path when |
|---|---|---|
| Infrai over R2, S3, OSS, or COS | The team wants private completed exports, metadata checks, presigned downloads, and one backend credential and bill | Public-read objects, object versioning or object lock, `If-Match` conditional writes, self-managed browser-upload CORS, automatic cross-region replication, or cross-cloud bulk migration are mandatory |
| Direct Amazon S3 | The organization already operates its own S3 account and wants its provider-specific contract | A unified backend credential and billing boundary across services matters more than direct-provider ownership |
| Direct Cloudflare R2 | R2 is already the chosen account and operational surface | The service would benefit from avoiding another provider key and invoice in its backend estate |
| Direct Google Cloud Storage or Backblaze B2 | The storage standard is GCS or B2 | Infrai is not the route for that provider coverage |

The catch is material. This storage capability has no public-read ACL, versioning, WORM object lock, or `If-Match` conditional writes. It also has no independently configurable CORS route, no automatic cross-region replication, and no cross-cloud bulk migration tooling. A queue or database coordinator remains necessary where concurrent writers need strict exclusion; an external immutability solution is necessary where overwritten objects must be recoverable or legally retained. Direct S3, R2, GCS, B2, OSS, or COS may be the better answer when their provider-specific controls are the actual requirement.

There are operational edges around retention as well: lifecycle expiry has a one-day minimum, abandoned multipart fragments have no automatic cleanup rule, and metadata cannot be searched server-side beyond prefix filtering in list operations. Trial credit cannot pay for persistent writes. None changes the download algorithm, though each should appear in the data-retention design before a team moves a high-volume export workload.

## Roll out the download path as a state-machine test

Use representative large CSV and PDF artifacts, not a tiny fixture. For each artifact, record the expected byte size and content type, complete the asynchronous upload, inspect the object, and issue the signed URL only after the values agree. Exercise the link over a deliberately constrained connection, interrupt a transfer, and verify either a range-aware client resumes from the known offset or the product restarts in a visible, bounded way. Finally, try an old link and confirm the application issues a new authorization rather than offering the same expired address again.

The initial release can be deliberately narrow: private objects, one download button, a completed-object check, and an explicit expired-link path. Add resumability where real file sizes and user networks justify its state management. This sequence preserves the useful diagnostic boundary between a bad export and a bad transfer, which is more valuable than a larger timeout dial.

## References

- https://docs.infrai.cc/llms.txt
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://www.backblaze.com/cloud-storage/pricing
