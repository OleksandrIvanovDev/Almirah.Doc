---
title: "SECR-003: Script-Scheme URLs in Links and Images"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Identified |
|   | 07-06-2026 | Analysed |
|   | 07-06-2026 | Mitigating |
| * | 13-06-2026 | Closed |

# Threat

An author points a Markdown link or image at a javascript: or data: URI, so a reader's click — on a page they trust — executes script or opens attacker-controlled content in the page's origin.

# Vulnerability

Link and image URLs were emitted with no scheme check. The inline parser's accidental truncation of a naive javascript: URI at the first parenthesis was not a control: percent-encoding or a data: URI passed through intact.

# Initial CVSS Vector

CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N

# Initial CVSS Score

5.4

# Mitigation

ADR-188 admits a link or image URL only when it is a relative reference or uses an allowed scheme — http, https, or mailto — and renders any other scheme as inert text instead of a clickable target.

# Residual CVSS Vector

CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:L/I:N/A:N

# Residual CVSS Score

3.5

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall admit a link or image URL only when it is a relative reference or uses an allowed scheme, and shall render any other scheme as inert text. | >[SRS-098] |

# Monitoring

Scheme-injection payloads, including percent-encoded and data: forms, run in the e2e suite on each build; extending the scheme allow-list requires a decision record.
