# Almirah Software Requirements Specification

## Overview

This document defines software requirements applicable for Almirah software.

## Reference Documents

| Document ID | Document Title |
|---|---|
| ARCH | [Almirah Framework Architecture](./../arch/arch.md) |

## Definitions

| Term | Definition |
|---|---|
| Item Id | Unique identifier for a paragraph in the specification document defined in the following form "[AAA-NNN]", where AAA - is any combination of letters and NNN - is the number. |
| Controlled Item | Paragraph in a specification document that is started with the Item Id. |
| External Item Id | Controoled Item Id from another specification that is also managed by Almira system. It is defined as ">[BBB-NNN]" symbols at the end of Controlled Item, where BBB - is any combination of leters different from AAA and NNN - is the number. |

## Requirements

[SRS-001] The software shall allow to create a Controlled Items. >[ARCH-005]

>Example: "[ITM-001] This is a controlled item"

[SRS-004] The software shall allow to create a non-controlled items.

>Example: "This is a non-controlled item"

[SRS-002] The software shall allow to create a reference from a Controlled Item to external Controlled Item. >[ARCH-001]

>Example: "[ITM-001] This is a controlled item with the reference to the external controlled item >[EXT-004]"

[SRS-003] The software shall indicate whether a Controlled Item is referenced in another specification via External Item Id >[ARCH-004]

### Text Formating

[SRS-011] The software shall allow to use internal links in the text (links to any markdown files that are managed by this software)

>Example 1: [This is a short form of internal link to the System Level Specification](sys.md)

>Example 2: [This is a relative form of internal link to the System Level Specification](./../sys/sys.md)

[SRS-012] The software shall allow to use the links to any headings in the text

>Example 1: [This is a link to the heading in the same document](#overview)

>Example 2: [This is a link to the heading in another internal document](./../sys/sys.md#overview)

[SRS-013] The software shall allow to use external links in the text (links to any public resources)

>Example 1: [This is an external link](https://www.markdownguide.org/extended-syntax)

>Example 2: [This is an external link with the anchorx](https://markdownguide.offshoot.io/extended-syntax/#tables)

[SRS-014] The software shall allow to use italic font decoration

>Example 1: *This is a text in italic*

[SRS-015] The software shall allow to use bold font decoration

>Example 1: **This is a text in bold**. However since this is a part of the blockquote it could be rendered as bold and italic. It is OK.

[SRS-016] The software shall allow to use bold and italic font decoration simultaneously

>Example 1: ***This is a text in bold and italic***

### Statistics

[SRS-005] The software shall provide the "Number of Controlled Items" for each specification

[SRS-006] The software shall provide the "Number of Items w/ Up-links" for each specification

[SRS-007] The software shall provide the "Number of Items w/ Down-links" for each specification

[SRS-008] The software shall provide the "Number of Items w/ Test Coverage" for each specification

[SRS-009] The software shall provide the "Duplicated Item Ids found" (just a number) for each specification

[SRS-010] The software shall provide the "Last used Item Id" for each specification
