---
title: "SECR-004: DOM XSS in the Image-Zoom Caption"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Identified |
|   | 07-06-2026 | Analysed |
|   | 07-06-2026 | Mitigating |
| * | 13-06-2026 | Closed |

# Threat

A reader opens an image in the zoom modal and the author-supplied alternate text executes as script — even though the page's static HTML is correctly escaped.

# Vulnerability

The image-zoom modal assigned the alt text to its caption via innerHTML. Reading the alt property in script returns the decoded value, so the attribute escaping applied at build time was undone and any markup in the alt text was re-parsed as live HTML — a DOM-based XSS the static escaping cannot close.

# Initial CVSS Vector

CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N

# Initial CVSS Score

6.1

# Mitigation

ADR-188 switched the caption — and set the rule for every author-derived value displayed by client-side script, the search results included — to DOM text and attribute interfaces, so decoded values render as inert text regardless of their content.

# Residual CVSS Vector

CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:N

# Residual CVSS Score

3.1

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall render author-derived values displayed by client-side scripts, including search results and the image caption, using DOM text and attribute interfaces rather than HTML re-parsing. | >[SRS-099] |

# Monitoring

The client-rendering safety spec statically asserts the shipped JS uses no HTML re-parsing interface for author-derived values; any new client-side rendering of authored content repeats the review.
