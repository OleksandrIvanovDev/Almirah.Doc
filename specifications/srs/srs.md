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

### Paragraph Types

[SRS-001] The software shall allow to create a Controlled Items. >[ARCH-005]

>Example: "[ITM-001] This is a controlled item"

[SRS-004] The software shall allow to create a non-controlled items. >[ARCH-003]

>Example: "This is a non-controlled item"

[SRS-002] The software shall allow to create a reference from a Controlled Item to external Controlled Item. >[ARCH-002]

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

[SRS-017] The software shall allow to use unordered lists

>Example of a unordered list is shown below:

* This is a first unordered list item
* This is a second unordered list item
* This is a third unordered list item

[SRS-018] The software shall allow to use ordered lists

>Example of a ordered list is shown below:

1. This is a first item of ordered list
1. This is a second item of ordered list
1. This is a third item of ordered list

[SRS-019] The software shall allow to use nested levels of ordered and unordered lists

>Example of a ordered list with nested levels is shown below:

1. This is a first item of ordered list
   1. This is a first item of ordered sub-list level 2
   1. This is a second item of ordered sub-list level 2
      1. This is a first item of ordered sub-list level 3
      1. This is a second item of ordered sub-list level 3
   1. This is a third item of ordered sub-list level 2
1. This is a second item of ordered list
   * This is a first item of unordered sub-list level 2
     1. This is a first item of ordered sub-list level 3
   * This is a second item of unordered sub-list level 2
   * This is a third item of unordered sub-list level 2
1. This is a third item of ordered list

1. This is a standalone list separated from previous one by emply line in the Markdown source file

### Traceability

[SRS-020] The software shall provide a traceability reports for all specificeations that contains Controlled Items with the references to external controlled item. The level of traceability is item to item (paragraph to paragraph) >[ARCH-001]

[SRS-021] The software shall generate trceability reports in HTML format. >[ARCH-004]

[SRS-022] The software shall additionally provide the traceability report in CSV fomrat if appropriate software configuration is enabled.

### Statistics

[SRS-005] The software shall provide the "Number of Controlled Items" for each specification

[SRS-006] The software shall provide the "Number of Items w/ Up-links" for each specification

[SRS-007] The software shall provide the "Number of Items w/ Down-links" for each specification

[SRS-008] The software shall provide the "Number of Items w/ Test Coverage" for each specification

[SRS-009] The software shall provide the "Duplicated Item Ids found" (just a number) for each specification

[SRS-010] The software shall provide the "Last used Item Id" for each specification
