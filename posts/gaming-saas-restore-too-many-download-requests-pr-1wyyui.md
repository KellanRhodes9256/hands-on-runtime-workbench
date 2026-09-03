# Gaming SaaS Restore: Too Many Download Requests, Presign Rate Limits, Retry Backoff

For a gaming SaaS, put a stable snapshot record between the export page and signed download-link creation, and make retention and deletion authoritative before you optimize request volume. Short answer: cache the signed URL for one snapshot, coordinate its refresh across application instances, and treat a page refresh as a read rather than permission to mint another URL.

This is a control-plane decision. A player-support operator may refresh a tenant's restore page ten times while checking a 40 GB backup; the object key and snapshot identity should remain unchanged, while the authorization to download can expire and be replaced. Retry backoff helps at the boundary. It cannot turn refresh spam into capacity.

## Retention is the restore contract

The system needs four invariants. A completed export has one immutable snapshot ID and one object key. A restore request names that snapshot explicitly. Deletion changes the snapshot's lifecycle state before the download handler issues authorization. Finally, a signed URL is treated as a short-lived bearer credential: it is returned from application state while safe to use, then refreshed under coordination.

The retention clock belongs to the snapshot record, not to the browser. Store `retention_expires_at`, `deletion_requested_at`, and a state such as `available`, `deleting`, or `deleted`. A scheduled deletion worker can remove the object and then record the result. The download path must reject a snapshot that is expired, deleting, or deleted even if an old URL is still present. This prevents a cache from silently overriding the deletion policy.

The failure boundary is equally important. Export generation, authorization refresh, object retrieval, and deletion are different operations with different retry rules. Repeating an export job can create duplicate data; refreshing authorization for the same object should not. A 429 from a presign endpoint is a signal to slow the authorization path, not to create another snapshot or to make the browser retry immediately.

| Decision | Good fit | Cost or limit |
|---|---|---|
| Application record plus reusable signed URL | Completed tenant snapshots opened from a status or restore page | Requires expiry checks, secret handling, and coordinated refresh |
| Per-request presigning | A policy that demands a fresh authorization decision for every download | Browser traffic becomes direct pressure on a rate-limited control plane |
| Public object URL | Non-sensitive assets intended for anonymous delivery | Wrong for tenant backups, retention-controlled exports, and restore data |
| Direct file proxy | Small files where central auditing and response filtering outweigh transfer cost | The application carries bandwidth, timeout, and range-request complexity |

The recommendation is the first row for ordinary gaming backups. The limitation is that a reusable bearer credential cannot satisfy a policy that forbids reuse, requires every request to be freshly evaluated, or demands immutable retention guarantees that the chosen object store does not provide. This approach is not suitable in those cases; keep the per-request authorization model or select a storage system with the required retention controls. Do not weaken deletion rules to make a page faster.

That is the whole point.

## How should download-link requests share a rate limit during refreshes?

The page handler should first read the snapshot row. If the snapshot is available and the cached URL has more than a safety margin left, return it. If the URL is absent or close to expiry, acquire a distributed lease keyed by snapshot ID. The lease holder mints one replacement and stores its expiry; other requests wait briefly and reread the row.

An in-memory cache is only an optimization. It does not coordinate two Node.js processes, survives no deployment, and can return stale authorization after another process records deletion. A database row with a conditional update, or a queue with one active refresher per key, is the durable control point. Keep the URL out of ordinary logs, traces, analytics events, and client-side error reports. You don't want a support screenshot, a trace exporter, and a browser error collector to become three extra places where a bearer credential persists, so the repository should expose a redacted snapshot view rather than handing raw columns to every caller.

In a concrete failure sequence, an operator opens tenant `t-1842`'s snapshot page, the browser retries after a slow response, and a second application instance sees the same nearly expired URL. Without a lease, both instances mint replacements; if the page is embedded in an operations dashboard that polls every five seconds, the same multiplication continues across every open tab. With a snapshot-keyed lease, the first instance owns the refresh briefly, the second rereads the winning value, and a deletion worker can still change the state so neither instance issues a new credential afterward. The important observation is that caching reduces authorization work only when lifecycle state and cache replacement share a clear ordering; an uncoordinated dictionary merely hides the race until a deployment or retention event exposes it.

Here is the critical path in Python, with the storage-specific presign call deliberately isolated behind an adapter. The deletion check happens before the cached credential is returned, and the cache key is the snapshot ID rather than a page URL.

```python
import time


def get_download_link(repo, snapshot_id, mint_url, now=None, safety_seconds=120):
    now = int(time.time()) if now is None else now
    snapshot = repo.get_snapshot(snapshot_id)
    if snapshot is None:
        raise KeyError("unknown snapshot")
    if snapshot["state"] != "available":
        raise PermissionError("snapshot is not available")
    if snapshot["retention_expires_at"] <= now:
        raise PermissionError("snapshot retention has expired")

    cached_url = snapshot["signed_url"]
    cached_expiry = snapshot["signed_url_expires_at"] or 0
    if cached_url and cached_expiry > now + safety_seconds:
        return cached_url

    lease_key = "snapshot-presign:" + snapshot_id
    if repo.try_lease(lease_key, seconds=15):
        refreshed = mint_url(snapshot["object_key"])
        repo.save_signed_url(
            snapshot_id,
            refreshed["url"],
            refreshed["expires_at"],
        )
        repo.release_lease(lease_key)
        return refreshed["url"]

    time.sleep(0.05)
    current = repo.get_snapshot(snapshot_id)
    if current and current["state"] == "available":
        return current["signed_url"]
    raise TimeoutError("authorization refresh is still in progress")
```

The production version needs a `try/finally` around lease release, a bounded wait loop, and a compare-and-update so a late refresher cannot overwrite a newer expiry. Those details are not decoration. They decide what happens when a worker is killed after minting a URL but before saving it, or when deletion races with a refresh. The object key must never be accepted from an untrusted browser parameter; derive it from the server-side snapshot row and validate upload and download handling against the application's file policy, as the OWASP guidance recommends.

## A lease is the boundary around authorization

Use retries only for transient authorization failures, with a small maximum attempt count, exponential delay, and random jitter. Honor `Retry-After` when the endpoint provides it. Do not retry a rejected snapshot, an expired retention window, or a malformed request. Those are decisions, not congestion.

The retry budget belongs to the lease holder, not to every browser request. Ten refreshes should produce one cache miss and at most one bounded mint sequence. A useful test is to send concurrent requests for the same completed snapshot and assert that only the lease winner calls `mint_url`; then repeat the test after the URL is expired, during deletion, and after a simulated worker restart.

Metrics should expose the shape of the pressure: presign attempts per snapshot, cache-hit ratio, lease contention, 429 count, retry delay, link age at return, and deletion-to-denial latency. Alerting on raw request count alone will confuse harmless page reads with authorization work. Also record an audit event for restore initiation without recording the signed URL itself.

Your mileage may vary on the safety margin. Clock skew, download duration, and the largest expected file should set it. A two-minute margin is an example in the code, not a universal security setting.

## How do tests prove deletion wins?

The dangerous tests are race tests, not happy-path download tests. Create a snapshot, complete it, and verify repeated page loads reuse its authorization. Start deletion while a refresh is waiting on its lease, and verify no new URL is issued after the state changes. Advance the retention clock, and verify the handler denies access even when the database still contains an unexpired signed URL. Retry an interrupted delete until the object and the record agree, with an operator-visible audit trail for ambiguous outcomes.

For gaming tenants, restore should be a separate operation from download. The selected snapshot ID is copied into a restore job; the worker validates tenant ownership and snapshot state again before writing into the target environment. A downloaded archive should be treated as untrusted input during unpacking: reject unexpected file types and paths, enforce size limits, and keep extraction outside executable locations. The OWASP file-upload controls are relevant to restore ingestion as well as user uploads.

I would reject a design that lets an export page construct object keys, chooses the latest snapshot implicitly, or interprets a successful URL mint as proof that the backup is restorable. The first two make authorization ambiguous; the third hides corrupt or incomplete backup data. The restore job needs its own validation and observable state.

The final rule is plain: cache authorization only inside the snapshot's valid lifecycle, coordinate replacement, and let deletion win races. Boring control flow is an advantage when the data is a tenant's last recoverable game state.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://aws.amazon.com/efs/
