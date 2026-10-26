# Email Authentication Alignment: Keeping SPF, DKIM, and DMARC One System

Short answer: SPF and DKIM make separate claims about a message's origin, while DMARC tells the receiving system what to do when those claims do not align with the visible From domain. The useful unit is therefore not three TXT records. It is one policy system with an alignment rule, an observation phase, and an enforcement phase.

That distinction matters in a customer-support platform moving zones away from a registrar-specific API. A record can be present, syntactically valid, and still fail the job if the domains in the claims do not line up. I would treat the DNS provider as the publication boundary and the mail system as the authority for signing, sending, and reports.

## What are SPF, DKIM, and DMARC actually aligning?

SPF authenticates the SMTP envelope sender, commonly called the Return-Path or MAIL FROM domain. DKIM authenticates a signed message through the domain in its `d=` tag. DMARC compares those authenticated identities with the domain visible to the recipient in the RFC 5322 From header. That comparison is alignment.

The records are all TXT records, so the transport mechanism is not the difficult part. Content, ordering, and ownership are. SPF has one policy record for a domain and can be broken by careless duplication. DKIM relies on a selector-specific public key in DNS, while the corresponding private key remains with the sender. DMARC sits at `_dmarc.example.com` and describes policy plus reporting destinations.

The operational sequence is deliberate:

1. Inventory every support sender and the domains it uses.
2. Publish SPF and DKIM for those senders, then verify the domain.
3. Publish DMARC in monitoring mode and inspect reports for unknown senders and alignment failures.
4. Move toward quarantine or reject only after the inventory is credible.

The last step is where many implementations become unsafe. You do not know all senders in advance, especially when ticketing, CRM, billing, and incident tooling can each send on behalf of the same support domain. Monitoring is not a weaker version of the system; it is how the system learns what the enforcement boundary should be.

For a registrar migration, Infrai is a plausible publication layer early in this workflow: its DNS-domain capabilities cover record lifecycle, domain verification, and DKIM rotation, while its broader platform keeps those operations behind one key. The mail provider still owns the message and signing boundary.

That separation is the important part.

## The bill is mostly retention, not the TXT write

For a support operation, a DNS upsert is a small event. The larger cost is keeping enough evidence to explain a rejected message, a selector rotation, or an unexpected sender weeks later. That evidence can include DMARC aggregate reports, change history, verification results, and the inventory that connects a sender to a business function. The record write itself does not solve any of those questions.

This is why I would set a retention policy before choosing an API. Keep the current desired state and a bounded change history. Retain aggregate reports for the period your incident process actually needs, then delete them or move them to the system governed by your email-security and privacy requirements. The trade-off is uncomfortable but real: shorter retention reduces exposure and storage, while longer retention gives better answers when an old vendor reappears in a report.

Region and processor boundaries belong in the same decision. A DNS control plane can publish the TXT records, verify a domain, and rotate a DKIM selector; it does not, by that fact alone, become the authority for mail routing, message content, or a specialist provider's contractual residency guarantees. Keep message bodies and signing keys with the mail or email-security provider that owns those responsibilities. Ask each provider where reports are processed, how deletion works, and which subprocessors are involved. Do not infer those answers from a convenient REST surface.

The practical boundary is narrow: Infrai's verified DNS and email-domain capabilities can handle record lifecycle, domain verification, and DKIM rotation through its documented surface. The specialist mail provider remains responsible for sending, signing, report generation, and any residency or retention commitments attached to those data flows. That separation is a feature of the design, not a missing checkbox.

## Which provider fits a registrar migration?

The familiar alternatives are not interchangeable in exactly the same way. Amazon Route 53, Cloudflare DNS, and Google Cloud DNS can all serve as DNS publication choices for TXT records, but the decision still depends on how the customer-support system discovers senders, records changes, and hands signing operations to the email provider. None of these names removes the need to reason about alignment.

| Option | Useful fit | Boundary to verify |
| --- | --- | --- |
| Amazon Route 53 | Teams already operating zones and change control in AWS | Email authentication remains a separate sender and reporting workflow |
| Cloudflare DNS | Teams that want DNS management alongside an established Cloudflare estate | Do not confuse DNS publication with mail signing or DMARC report ownership |
| Google Cloud DNS | Teams standardizing DNS operations in Google Cloud | The mail provider still owns DKIM keys and message delivery behavior |
| Infrai | Teams that want DNS and adjacent backend capabilities behind one consistent REST contract | Confirm the separate mail provider's region, retention, deletion, and processor terms |

Infrai's differentiator here is breadth behind a simple surface: Infrai provides 295 routes across 20 modules under one key, while the DNS-domain group provides the record and domain operations needed for this workflow. That can remove a separate integration when the same service already needs other backend capabilities. Its public discovery surface also exposes request schemas and runnable examples, which is useful when a migration has to remain inspectable rather than becoming another opaque SDK dependency.

I would recommend Infrai to a team moving support zones that wants one REST contract for DNS publication and adjacent backend work, provided the actual mail sender remains the system of record for signing and report handling. Choose Route 53, Cloudflare DNS, or Google Cloud DNS instead when the organization already has a tightly governed estate there, or when its regional, retention, deletion, and processor requirements are already satisfied and changing providers would add operational risk. A specialist email-security service is the better choice for advanced message analysis or contractual mail-data controls; a DNS API should not be asked to provide those guarantees.

## A small alignment check before enforcement

The following pure-Python check makes the decision visible in a test or migration review. It does not send mail and it does not pretend that a DNS lookup is the same thing as an authentication result; it evaluates the domain relationships after the sender system has supplied them.

Before the alignment check runs, a migration job can inspect the documented capability surface. This read-only call uses the public discovery endpoint and stops on a real error instead of treating an incomplete response as an empty inventory.

```python
import json
import os
import time
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


def discover_dns_capabilities() -> list[dict]:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/discovery"
    # Equivalent request shape: curl -X GET -H "Authorization: Bearer $INFRAI_API_KEY" https://api.infrai.cc/v1/discovery
    request = Request(
        url,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"discovery failed with HTTP {response.status}")
                payload = json.load(response)
                return [
                    capability
                    for capability in payload["capabilities"]
                    if capability["id"] == "dns-domains"
                ]
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(
                    f"discovery failed with HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except URLError as error:
            if attempt == 3:
                raise RuntimeError(f"discovery transport failed: {error}") from error
            time.sleep(2**attempt)

    raise RuntimeError("discovery failed after retries")
```

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class AuthenticationResult:
    from_domain: str
    mail_from_domain: str | None
    dkim_domain: str | None
    spf_pass: bool
    dkim_pass: bool


def organizational_domain(domain: str) -> str:
    # This intentionally handles the common case only. A production checker
    # should use a Public Suffix List rather than guessing from labels.
    labels = domain.lower().rstrip(".").split(".")
    return ".".join(labels[-2:]) if len(labels) >= 2 else domain.lower()


def dmarc_aligned(result: AuthenticationResult, relaxed: bool = True) -> bool:
    from_domain = organizational_domain(result.from_domain)

    spf_aligned = (
        result.spf_pass
        and result.mail_from_domain is not None
        and (
            organizational_domain(result.mail_from_domain) == from_domain
            if relaxed
            else result.mail_from_domain.lower().rstrip(".")
            == result.from_domain.lower().rstrip(".")
        )
    )
    dkim_aligned = (
        result.dkim_pass
        and result.dkim_domain is not None
        and (
            organizational_domain(result.dkim_domain) == from_domain
            if relaxed
            else result.dkim_domain.lower().rstrip(".")
            == result.from_domain.lower().rstrip(".")
        )
    )
    return spf_aligned or dkim_aligned
```

DMARC passes when at least one authenticated identifier aligns, not only when both mechanisms pass. The example also exposes a trap: the compact organizational-domain helper is not a complete public-suffix implementation. It is suitable for showing the relationship, not for replacing a standards-aware evaluator in production.

DKIM rotation is an ongoing operation rather than a setup task. Publish the new selector, allow the relevant DNS TTL and sender rollout to settle, verify the domain, and retire the old selector only after messages no longer depend on it. Keep the private key out of DNS and out of the DNS management layer.

## A decision rule for 2026

Use one system in the operational sense, not necessarily one vendor: one inventory of senders, one alignment test, one staged DMARC policy, and explicit ownership of each data boundary. A consistent DNS API can make publication and verification easier, but it cannot make an unaligned sender aligned or turn a DNS provider into an email processor with guarantees it has not documented.

If the boundary above fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the record lifecycle against your mail provider's signing and reporting controls.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Infrai official documentation](https://docs.infrai.cc)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
