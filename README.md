# VDP Methodology Writeup — Credential Exposure via Third-Party Vendor API

**Status:** Report submitted to a Vulnerability Disclosure Program (VDP) via HackerOne. Pending vendor triage/remediation.

> **Note on disclosure:** Per responsible disclosure norms and the host program's terms, the target organization, exact endpoints, and account identifiers are withheld until the vendor confirms remediation and grants public disclosure. This writeup focuses on methodology, reasoning, and the general vulnerability class rather than reproducible exploit details.

## Summary

While mapping the external attack surface of an in-scope web asset for a VDP program, I identified that a publicly reachable configuration file disclosed a tenant/account identifier for a third-party mapping/wayfinding vendor integration. That identifier could be submitted — unauthenticated — to the vendor's own credential-provisioning API, which returned a **live Azure Service Bus connection string** (including a valid SharedAccessKey) scoped to the vendor's analytics/telemetry namespace.

**Classification:** Insufficiently Protected Credentials (CWE-522) / Missing Authentication for Critical Function
**Severity:** High (CVSS 4.0, AV:N/AC:L/AT:N/PR:N/UI:N — full vector withheld pending disclosure)
**Scope note:** Testing was unauthenticated and strictly read-only, performed against owned/in-scope assets from an isolated sandbox. No data was written to or consumed from the downstream system.

## Methodology

1. **Recon** — Enumerated public-facing subdomains and static assets of the in-scope target. Found a publicly accessible JSON configuration file used to bootstrap a third-party interactive-maps widget.

2. **Identifier discovery** — The config file exposed an `accountId` used to scope the widget to the organization's tenant on the vendor's platform. This is standard practice for many SaaS embed integrations — but it becomes a problem when the vendor's backing API trusts that identifier alone as an authorization boundary.

3. **Trust-boundary testing** — Queried the vendor's documented (but not intended for direct client use) credential-provisioning endpoint with the harvested `accountId`. The endpoint returned a live cloud message-bus connection string with no authentication challenge of any kind.

4. **Validation (not exploitation)** — To confirm the response was a genuine, tenant-specific credential rather than a static placeholder or decoy, I submitted a syntactically invalid account identifier to the same endpoint. The vendor API responded with `400 Bad Request`, demonstrating that it actively validates the identifier server-side and that the credential returned for the real identifier is live and tenant-scoped — not a fixture.

5. **Stopped at validation.** I did not connect to the disclosed message bus, read or write any data, or use the credential for anything beyond confirming it was real. This is a hard line in my testing methodology: proving exposure, never exercising access.

## Why this matters

This is a class of bug that's easy to miss because the vulnerable logic doesn't live on the target organization's own infrastructure — it lives in a **vendor's API trusting a client-supplied, publicly visible identifier as if it were a secret**. The fix belongs partly to the vendor (require real authentication on credential-issuing endpoints) and partly to the integrating organization (avoid exposing tenant identifiers in public configs where the downstream trust model is unclear).

## Skills demonstrated

- External attack surface mapping and config/asset discovery
- Third-party/vendor trust-boundary analysis (a step many scanners miss)
- Evidence-based validation methodology (control testing to rule out false positives)
- CVSS 4.0 scoring, including the Vulnerable System vs. Subsequent System distinction
- Responsible, scope-respecting, read-only testing discipline
- Clear technical writing for triage audiences

## Tooling

- `curl`, `jq` for request/response inspection
- Manual recon and source review (no automated scanner flagged this)

---

*Full technical details, exact endpoints, and the HackerOne report link will be added here once the vendor confirms remediation and the program authorizes public disclosure, consistent with coordinated disclosure practice.*
