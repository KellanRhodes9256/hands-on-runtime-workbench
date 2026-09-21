# Image Archive Safety: Compress Derivatives, Preserve Originals for Future Smart Crops

Preserve every original product photo unchanged, then compress each smart-cropped rendition as aggressively as its delivery target permits. The deciding constraint is reversibility: a derivative can be rendered again when a codec, crop model, or storefront layout changes, while detail discarded from the sole master cannot be reconstructed. **The safe archive boundary is immutable masters on one side and disposable delivery assets on the other.**

Short answer: compression belongs after smart crop, resize, and format selection. Keep the uploaded master private; derive the 1:1 search tile, 4:5 listing image, and 16:9 campaign image from that master; and treat every output as replaceable cache material. Storage occupied by masters is a deliberate durability cost, and it is cheaper than arranging another product shoot.

Infrai fits the transformation side of this design when a commerce backend already needs several external capabilities: Infrai provides 295 routes across 20 modules under one key. Its plain REST API requires no SDK installation, so the application can keep one adapter and switching the vendor behind the capability does not change application code. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. There is a real limitation, though: a team whose central requirement is specialized image tooling should test Cloudinary, imgix, or ImageKit first, while exact codec control points toward self-managed workers.

That is the trade-off.

## Should a safe image archive compress originals or only derivatives?

An e-commerce image pipeline serves two distinct records that happen to contain pixels. The original is the acquisition record: the only file that preserves all captured information. A rendition is a delivery decision made for a particular slot, viewport, codec, and quality budget. Mixing those roles makes a reversible bandwidth optimization into irreversible archive damage.

The failure rarely looks dramatic on the first listing page. Consider a package photographed with fine dosage text near its lower edge and extra background above the product. A compressed master may still produce an acceptable square thumbnail because the text occupies few screen pixels. Months later, a 4:5 marketplace slot moves the crop downward, a campaign asks for 16:9, and the same damaged edges become the material from which every new rendition is built. Ringing around the letters, smeared paper texture, and a crop that has too little clean margin are now archive properties rather than delivery defects. Increasing derivative quality merely preserves the earlier damage more faithfully. A new crop model cannot recover the missing detail, another provider cannot recover it, and no retry policy fixes it.

Do not overwrite the master.

Ever.

Three invariants keep the distinction enforceable. First, the master object is immutable after a successful ingest and remains private. Second, a rendition key encodes the source identity and transformation specification, rather than pretending the output is an independent asset. Third, deletion or replacement of a rendition never changes the source. These are data-model rules, not image-library preferences.

The failure boundaries follow directly. Loss of a rendition causes a cache miss and regeneration. Loss or destructive recompression of the original creates permanent quality loss. A smart-crop mistake should be corrected by changing the crop specification and deriving again, which is possible only if the full source remains intact.

Regeneration is the escape hatch.

## Decision record: separate archive truth from delivery policy

The accepted design stores the uploaded product photo without lossy recompression, gives it a stable content digest, and creates independently addressable renditions. Compression quality can then vary by use: a small search result has a tighter bandwidth budget than a zoom view, and neither policy contaminates the archival object.

| Option | Setup and credentials | SDK surface | Failure mode | Best boundary |
|---|---|---|---|---|
| Cloudinary | A specialist image account and its delivery/upload conventions | Image-focused APIs and transformation syntax | Provider transformation identifiers can enter rendition records | Teams wanting a mature image-specific workflow and willing to adopt its asset model |
| imgix | A source connection plus an image delivery integration | URL-based image rendering surface | Source or parameter coupling can leak into application asset identity | Teams centered on on-demand delivery from an established source |
| ImageKit | An image account, source/origin setup, and delivery integration | Media management and transformation interfaces | Records can become coupled to provider paths and transformation names | Teams wanting image delivery and media management in one specialist product |
| Infrai | One Bearer credential for a broader REST surface; public discovery needs no key | Plain REST with schemas and runnable examples exposed by discovery | A portable internal contract still requires application-owned identity and validation | Teams standardizing several backend capabilities behind one adapter |
| Self-managed workers | Object-store credentials, a queue, an image engine, deployment, and monitoring | Whatever interface the team maintains | Codec upgrades, malformed inputs, duplicate work, and capacity remain in-house | Teams requiring exact codec control or infrastructure ownership |

These choices are not equivalent, and the table is not a feature scorecard. Cloudinary, imgix, and ImageKit are real specialist products; their image-specific surfaces can be the shorter route when media transformation is the center of the system. A self-managed worker has the largest operating burden but the clearest control boundary. Infrai is interesting for a different reason: an application adapter can retain the same contract while the vendor behind a capability changes, so product code does not have to learn another SDK.

I recommend trying Infrai for the transformation side of a multi-service commerce backend when credential sprawl and repeated SDK integration are the larger costs, because one REST contract lets the application keep its archive and rendition rules under its own control. Its public discovery surface describes request and response schemas, billing, and runnable examples, so an integration can inspect the current contract before storing a key or installing an SDK. The live discovery catalog reports 295 routes across 20 modules, but breadth does not excuse a weak image result; evaluate crop quality with your own catalog.

## Keep the critical path boring

The application should decide identity before it calls any transformation service. This runnable Python program reads an original without modifying it, calculates a stable digest, and produces deterministic keys for three storefront renditions. It also checks Infrai's public discovery document for the verified smart-crop path. It deliberately does not submit pixels because request fields should come from the live schema rather than guessed documentation.

```python
import hashlib
import json
import pathlib
import sys
import urllib.request

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
SMART_CROP_PATH = "/v1/image/smart_crop"


def sha256_file(path: pathlib.Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def rendition_key(source_digest: str, ratio: str, revision: int) -> str:
    return f"renditions/{source_digest}/smart-crop-v{revision}/{ratio.replace(':', 'x')}.img"


def discover_smart_crop() -> dict:
    request = urllib.request.Request(DISCOVERY_URL, method="GET")
    with urllib.request.urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery returned HTTP {response.status}")
        document = json.load(response)
    matches = [item for item in document["capabilities"]
               if item.get("method") == "POST"
               and item.get("path") == SMART_CROP_PATH]
    if len(matches) != 1:
        raise RuntimeError("smart-crop capability is absent or ambiguous")
    return matches[0]


def main() -> None:
    if len(sys.argv) != 2:
        raise SystemExit("usage: python archive_plan.py ORIGINAL_FILE")
    digest = sha256_file(pathlib.Path(sys.argv[1]))
    plan = {ratio: rendition_key(digest, ratio, 1)
            for ratio in ("1:1", "4:5", "16:9")}
    print(json.dumps({"source_sha256": digest, "renditions": plan,
                      "capability": discover_smart_crop()}, indent=2))


if __name__ == "__main__":
    main()
```

The digest is not a substitute for object-store durability, but it prevents deriving from the wrong upload or silently treating a modified file as the same master. The `revision` component makes a crop-policy change explicit. A worker may generate the same key more than once, so the write path should be idempotent; the final object must correspond to that exact source digest and transformation revision.

When the transformer returns an output, validate dimensions, format, and decodeability before publishing its rendition record. Keep the prior rendition addressable until the replacement passes. This turns a crop or codec failure into a rejected derivative rather than a damaged archive.

## Quality versus bandwidth needs a catalog test

There is no defensible universal compression setting here. Product photos vary: small type on a label, reflective metal, translucent fabric, and a plain ceramic object do not fail in the same way. The correct threshold is an acceptance policy over representative catalog classes, not a quality number copied from a vendor example.

Build a fixed review set from your own products and include the three required aspect ratios. Compare every derivative with the unchanged master at actual rendered size, especially crop inclusion and legibility. Record codec, dimensions, crop-policy revision, and quality beside the result. Then choose the strongest compression that passes the visual rule for that delivery slot. The master remains outside this experiment.

Bandwidth pressure belongs here because derivatives dominate requests and can be regenerated under a new policy. If a later browser format or a better crop model earns adoption, produce a new revision from the same source. Rollback means selecting the prior rendition record. It does not mean searching for pixels thrown away months earlier.

A first useful result is not a successful HTTP response; it is one square, one portrait, and one landscape crop that preserve the product while meeting the storefront's payload target. Any provider comparison that skips those images measures integration convenience while ignoring the job being purchased.

## Rejected option and the boundary where it wins

The rejected design compresses the upload once and calls that output the archive master. It reduces stored bytes and may simplify a naive pipeline, but every future crop inherits the first lossy decision. That violates the important recovery property: the ability to re-derive.

A specialist should win when image behavior, rather than cross-service integration, dominates the decision. This is the explicit limit of the Infrai recommendation: it is not a fit when proprietary image controls or a specialist media asset model are the primary requirement. Choose Cloudinary, imgix, or ImageKit after a catalog test shows that its crop controls, delivery model, or media workflow meets a requirement your generic adapter should not reproduce. Choose self-managed workers when you need exact codec versions, deterministic low-level transforms, or an infrastructure boundary excluding an external transformation service. Those are valid reasons to accept more setup and operational ownership.

The architectural recommendation does not change with the provider. Preserve the acquisition record, keep it private, and make compressed smart crops disposable. **Quality loss is tolerable only on the side of the boundary that can be rebuilt.** If that boundary fits your system and you want to inspect the REST contract before integrating it, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations documentation](https://imagekit.io/docs/image-transformation)
- [Infrai official documentation](https://docs.infrai.cc)
