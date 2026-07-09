---
title: Security Risk Register
---

# Security Risk Register

This registry collects the security risks of the Almirah gem itself, in the ISO 27005 shape: each record names a threat, the vulnerability it exploits, and the mitigation that controls it. The initial entries capture the findings of the HTML rendering-path security review that led to [[adr-188-html-output-escaping]] — authored Markdown was interpolated into the generated site essentially raw, so anyone able to land a file in an input repository could plant script that ran in every reader's browser.

## Scoring

Every record carries two CVSS v3.1 base scores, each precomputed from the vector written beside it:

- **Initial CVSS Score** — the vulnerability as found by the review, before any control existed.
- **Residual CVSS Score** — the vulnerability re-assessed after the ADR-188 controls (context-aware output escaping, the URL scheme allow-list, and DOM text interfaces) shipped in release 0.4.2. What remains is the regression risk of a future rendering path bypassing the escaping helpers.

The register surfaces both scores unchanged as the Initial CVSS RPN and Residual CVSS RPN columns — single-input groups, no product is computed. The colour bands follow the CVSS severity boundaries:

| Band | Score | Register colour |
|---|---|---|
| Low | 0.1 – 3.9 | acceptable (green) |
| Medium / High | 4.0 – 8.9 | caution (amber) |
| Critical | 9.0 – 10.0 | unacceptable (red) |

## Lifecycle

Records move through Identified, Analysed, Mitigating, Accepted, and Closed. Every record whose current status is not Closed counts as open in the registries summary. The ADR-188 findings are Closed: their mitigations are implemented, verified by the XSS regression suite in `Almirah.Code`, and stated as SRS-096 through SRS-099, which each record links under Affected Documents.
