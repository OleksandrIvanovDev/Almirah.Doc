---
title: "SECR-002: Attribute Breakout via Image and Link Values"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Identified |
|   | 07-06-2026 | Analysed |
|   | 07-06-2026 | Mitigating |
| * | 13-06-2026 | Closed |

# Threat

An author crafts an image alternate text or a link value that closes its HTML attribute and injects an event handler — script execution without ever writing a script element, e.g. an alt text of a quote followed by an onerror handler.

# Vulnerability

Values interpolated into HTML attributes — an image's source and alternate text, a link's address and visible text — were emitted without attribute escaping, so a double quote in authored content broke out of the attribute and introduced new attributes or elements on the tag.

# Initial CVSS Vector

CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N

# Initial CVSS Score

5.4

# Mitigation

ADR-188 attribute-escapes every author-supplied value interpolated into an HTML attribute, so quotes and angle brackets render as inert text inside the attribute value. Verified by the XSS regression suite with breakout payloads in image and link positions.

# Residual CVSS Vector

CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:N

# Residual CVSS Score

3.1

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall escape author-supplied values interpolated into HTML attributes, including an image's source and alternate text and a link's address and visible text. | >[SRS-097] |

# Monitoring

Breakout payloads in every attribute position run in the e2e suite on each build; new attributes fed from authored content must use the shared attribute-escaping helper.
