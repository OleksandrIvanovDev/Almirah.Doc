---
title: "SECR-001: Stored XSS in Rendered Element Content"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Identified |
|   | 07-06-2026 | Analysed |
|   | 07-06-2026 | Mitigating |
| * | 13-06-2026 | Closed |

# Threat

A contributor who can land a Markdown file in any input repository plants a script element that executes in every reader's browser on the published docs domain — session theft, credential phishing on a trusted domain, pivoting against co-hosted services.

# Vulnerability

Author-supplied literal text was interpolated into element content raw: paragraph, heading, blockquote and table-cell text and fenced code block lines reached the generated HTML unescaped, so a paragraph containing a script element was reproduced verbatim and fired on page load — stored XSS baked into the static site at build time.

# Initial CVSS Vector

CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N

# Initial CVSS Score

6.1

# Mitigation

ADR-188 introduced a single escaping mechanism applied to every literal text token after Markdown structure is recognised, so Almirah's own formatting tags are preserved while author-supplied markup renders inert. Verified by the XSS regression suite rendering crafted payloads.

# Residual CVSS Vector

CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:N

# Residual CVSS Score

3.1

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall HTML-escape all author-supplied literal text rendered into element content, including paragraph, heading, blockquote, table-cell, and fenced code block text. | >[SRS-096] |

# Monitoring

The e2e XSS escaping suite renders script payloads through every content path on each build; a new document type or rendering path must route text through the shared escaping helper.
