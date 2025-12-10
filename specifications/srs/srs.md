% Almirah Software Requirements Specification

# Overview

This document defines software requirements applicable for Almirah software.

# Reference Documents

| Document ID | Document Title |
|---|---|
| ARCH | [Almirah Framework Architecture](./../arch/arch.md) |

# Definitions

| Term | Definition |
|---|---|
| Item ID | Unique identifier for a paragraph in the specification document defined in the following form "[AAA-NNN]", where AAA - is any combination of letters and NNN - is the number. |
| Controlled Item | Paragraph in a specification document that is started with the Item ID. |
| External Item ID | Controlled Item ID from another specification that is also managed by Almirah framework. It is defined as ">[BBB-NNN]" symbols at the end of Controlled Item, where BBB - is any combination of letters different from AAA and NNN - is the number. |

# Requirements

## Specifications

### Paragraph Types

[SRS-001] The software shall allow creating Controlled Items. >[ARCH-005], >[SYS-005]

>Example: "[ITM-001] This is a controlled item"

[SRS-004] The software shall allow to create a non-controlled items. >[ARCH-003], >[SYS-005]

>Example: "This is a non-controlled item"

[SRS-002] The software shall allow to create a reference from a Controlled Item to external Controlled Item. >[ARCH-002], >[SYS-001]

>Example: "[ITM-001] This is a controlled item with the reference to the external controlled item >[EXT-004]"

[SRS-003] The software shall indicate whether a Controlled Item is referenced in another specification via External Item ID >[ARCH-004]

### Text Formatting

[SRS-011] The software shall allow using internal links in the text (links to any markdown files that are managed by this software)

>Example 1: [This is a short form of internal link to the System Level Specification](sys.md)

>Example 2: [This is a relative form of internal link to the System Level Specification](./../sys/sys.md)

[SRS-012] The software shall allow to use the links to any headings in the text

>Example 1: [This is a link to the heading in the same document](#overview)

>Example 2: [This is a link to the heading in another internal document](./../sys/sys.md#overview)

[SRS-013] The software shall allow using external links in the text (links to any public resources)

>Example 1: [This is an external link](https://www.markdownguide.org/extended-syntax)

>Example 2: [This is an external link with the anchor](https://markdownguide.offshoot.io/extended-syntax/#tables)

[SRS-014] The software shall allow using italic font decoration

>Example 1: *This is a text in italic*

[SRS-015] The software shall allow using bold font decoration

>Example 1: **This is a text in bold**. However since this is a part of the blockquote it could be rendered as bold and italic. It is OK.

[SRS-016] The software shall allow using bold and italic font decoration simultaneously

>Example 1: ***This is a text in bold and italic***

[SRS-017] The software shall allow using unordered lists

>Example of an unordered list is shown below:

* This is a first unordered list item
* This is a second unordered list item
* This is a third unordered list item

[SRS-018] The software shall allow using ordered lists

>Example of an ordered list is shown below:

1. This is a first item of ordered list
1. This is a second item of ordered list
1. This is a third item of ordered list

[SRS-019] The software shall allow using nested levels of ordered and unordered lists

>Example of an ordered list with nested levels is shown below:

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

1. This is a standalone list separated from previous one by empty line in the Markdown source file

[SRS-024] The software shall allow using unordered lists with bold, italic, and mixed formatting

>Example of an unordered list is shown below:

* *This is a first unordered list item formatted in italic*
* **This is a second unordered list item formatted in bold**
* ***This is a third unordered list item formatted in a mixed way***

[SRS-025] The software shall allow using ordered lists with bold, italic, and mixed formatting

>Example of an unordered list is shown below:

1. *This is a first ordered list item formatted in italic*
1. **This is a second ordered list item formatted in bold**
1. ***This is a third ordered list item formatted in a mixed way***

[SRS-026] The software shall allow using nested levels of ordered and unordered lists with bold, italic, and mixed formatting

>Example of an ordered list with nested levels is shown below:

1. This is a first item of ordered list
   1. *This is a first item of ordered sub-list level 2 (formatted in italic)*
   1. This is a second item of ordered sub-list level 2
      1. This is a first item of ordered sub-list level 3
      1. **This is a second item of ordered sub-list level 3 [formatted in bold]**
   1. *This is a third item of ordered sub-list level 2 formatted in italic*
1. This is a second item of ordered list
   * ***This is a first item of unordered sub-list level 2 {formatted in a mixed way}***
     1. This is a first item of ordered sub-list level 3
   * ***This is a second item of unordered sub-list level 2 formatted in a mixed way***
   * This is a third item of unordered sub-list level 2
     * **This is a first item of unordered sub-list level 3 before the skip-level list item**
1. This is a third item of ordered list

[SRS-027] The software shall allow using code blocks (see example below).

```JSON
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

[SRS-037] The software shall allow to align the text in markdown table clolumns using the following options:

* '| :--- |' - alignment left;
* '| ---: |' - alignment right;
* '| :---: |' - alignment center.

Table example:

| Column with Alignment Left | Column with Alignment Center | Column with Default Alignment | Coulmn with Alignment Rignt |
| :--- | :---: | --- | ---: |
| Text A | Text B | Text C | Text D |

### Traceability

[SRS-020] The software shall provide a traceability reports for all specifications that contains Controlled Items with the references to external controlled item. The level of traceability is item to item (paragraph to paragraph) >[ARCH-001]

[SRS-021] The software shall generate traceability reports in HTML format. >[ARCH-004]

[SRS-022] The software shall additionally provide the traceability report in CSV format if appropriate software configuration is enabled.

[SRS-023] The software shall indicate if there is a link to the item that does not exist. (like the following one) >[ARCH-999], (or that one) >[DNEXST-001]

### Statistics

[SRS-005] The software shall provide the "Number of Controlled Items" for each specification

[SRS-006] The software shall provide the "Number of Items w/ Up-links" for each specification

[SRS-007] The software shall provide the "Number of Items w/ Down-links" for each specification

[SRS-008] The software shall provide the "Number of Items w/ Test Coverage" for each specification

[SRS-009] The software shall provide the "Duplicated Item Ids found" (just a number) for each specification

[SRS-010] The software shall provide the "Last used Item ID" for each specification

## Source Code

[SRS-037] The software shall allow to create a reference from a Source Code File to external Controlled Item.

>Example: "<REQ> Some Text >[EXT-004] </REQ>"

## User Interface

### Up-links

[SRS-028] For each Controlled Item software shall show an External Controlled Item ID the Item refers to in form of clickable link (up-link).

[SRS-031] If there is more than one up-link in the Controlled Item, the software shall show them in two steps:

1. Initially the software show the number of up-links (clickable);
1. When User clicks on the number of up-link, the number is replaced with the list of clickable up-links;

[SRS-032] If User clicks on the up-link, the software shall navigate to an External Controlled Item this up-link refers to.

### Down-links

[SRS-029] For each Controlled Item software shall show an External Controlled Item ID the Item is referenced in form of clickable link (down-link).

[SRS-033] If there is more than one down-link in the Controlled Item, the software shall show them in two steps:

1. Initially the software show the number of down-links (clickable);
1. When User clicks on the number of down-link, the number is replaced with the list of clickable down-links;

[SRS-034] If User clicks on the down-link, the software shall navigate to an External Controlled Item this item is referenced in.

### Coverage Links

[SRS-030] For each Controlled Item software shall show a test case and test step IDs the Item is verified by in form of clickable link (coverage link).

[SRS-035] If there is more than one coverage-link in the Controlled Item, the software shall show them in two steps:

1. Initially the software show the number of coverage links (clickable);
1. When User clicks on the number of coverage link, the number is replaced with the list of clickable coverage links;

[SRS-036] If User clicks on the coverage link, the software shall navigate to a particular test case and test step this item is verified by.
