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

[SRS-017] The software shall allow using unordered lists with either "*" or "-" as the item marker

>Example 1: an unordered list using "*" as the marker

* This is a first unordered list item
* This is a second unordered list item
* This is a third unordered list item

>Example 2: an unordered list using "-" as the marker

- This is a first unordered list item
- This is a second unordered list item
- This is a third unordered list item

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

[SRS-038] The software shall allow to create a reference from a Source Code File to external Controlled Item.

>Example: "<REQ> Some Text >[EXT-004] </REQ>"

## Decision Records

[SRS-039] The software shall allow to create decision records as a markdown files (one file for each record)

[SRS-040] The software shall derive the Decision Record ID as the filename prefix made up of letters, a dash, and digits. If the filename does not match this pattern, the full filename stem shall be used as the ID.

>Example 1: "adr-170-introduce-decision-records.md" — ID is "adr-170"

>Example 2: "ise-1892.md" — ID is "ise-1892"

>Example 3: "meeting-notes.md" — ID is "meeting-notes"

[SRS-043] The software shall accept decision records placed in the `decisions/` folder at the project root, including nested subfolders.

[SRS-044] The software shall derive the Decision Record Sequence Number as the digits portion of the ID.

>Example: ID "adr-170" — Sequence Number is "170"

[SRS-045] The software shall derive the Decision Record Type as the letters portion of the ID, in upper case.

>Example: ID "adr-170" — Type is "ADR"

[SRS-046] The software shall accept the Decision Record title from the YAML frontmatter `title` field of the source markdown file.

>Example: a file with frontmatter `title: "ADR-170: Introduce Decision Records"` renders with that title.

[SRS-047] The software shall render each decision record to an individual HTML page whose filename matches the Decision Record ID.

>Example: "decisions/adr-170-introduce-decision-records.md" → "build/decisions/adr-170.html"

[SRS-049] The software shall recognise a leading marker column in a Decision Record Status table and shall expose the status value of the row marked with "*" as the Decision Record's current status. When zero rows or more than one row carry the marker, the current status shall be undefined.

>Example: a Status table row containing "*" in the leading column and "Accepted" in the Status column makes "Accepted" the current status of the record.

[SRS-050] The software shall render the "*" in the leading marker column of a Decision Record Status table as a solid right-pointing triangle ("▶") in the generated HTML, and shall visually highlight the entire marked row so that the current state is distinguishable from past and planned rows. Other tables and other columns shall be unaffected.

>Example: the source row "| * | 17-05-2026 | Accepted |" renders with "▶" in the first column cell and a highlighted row background.

[SRS-052] The software shall recognise the "Affected Documents" section in Decision Records that indicates the list of Controlled Items with their text updated or created in scope of the Decision Record.

[SRS-053] The software shall recognise the "Affected Documents" section as a single markdown table with the following columns in order: "#", "Proposed Text", and "Req-ID".

[SRS-054] The software shall accept the External Controlled Item ID in the "Req-ID" column of the "Affected Documents" table in the form ">[BBB-NNN]", using the same syntax as test step references in Test Protocols.

>Example: a row "| 1 | The software shall ... | >[SRS-052] |" links the Decision Record to the Controlled Item [SRS-052].

[SRS-055] When a Decision Record contains an "Affected Documents" section, the software shall establish a link from each row of the section to the referenced Controlled Item in the target Specification document.

[SRS-056] The software shall report a broken reference if the "Req-ID" column in an "Affected Documents" table refers to a Controlled Item ID that does not exist, naming the owning Decision Record in the report.

[SRS-057] The software shall render the "Req-ID" cell in the "Affected Documents" table of a Decision Record as a clickable link to the referenced Controlled Item in the Specification document.

[SRS-061] The software shall expose a Start Date attribute on each Decision Record, computed as the earliest calendar date found in the Date column of the Decision Record's Status table and in the Start Date column of the Decision Record's Scope table.

[SRS-062] The software shall recognise dates in the form "DD-MM-YYYY" when computing the Start Date attribute of a Decision Record. Cells whose contents do not match this form shall be skipped without raising an error.

[SRS-063] The software shall identify the Date column of a Decision Record's Status table and the Start Date column of a Decision Record's Scope table by their header text, case-sensitive, and not by column position.

[SRS-064] When neither the Status table's Date column nor the Scope table's Start Date column of a Decision Record contains a date matching the recognised form, the Start Date attribute of that Decision Record shall be undefined.

[SRS-065] The Decision Records Overview page shall render the Start Date attribute of each Decision Record in the existing Start Date column, formatted as "DD-MM-YYYY". The cell shall be empty when the Start Date attribute is undefined.

[SRS-066] The software shall expose a Target Release Version attribute on each Decision Record, computed by locating the Decision Record's Software Versions section, finding the row of its table whose Software Version Category cell equals "Target Release Version", and taking the value of that row's Software Version ID cell.

[SRS-067] The software shall identify the Software Version Category column and the Software Version ID column of a Decision Record's Software Versions table by their header text, case-sensitive, and not by column position.

[SRS-068] The software shall identify the Target Release Version row of a Decision Record's Software Versions table by an exact, case-sensitive match against its Software Version Category cell after stripping surrounding whitespace.

[SRS-069] When a Decision Record's Software Versions section, the table within it, the Software Version Category or Software Version ID column, the Target Release Version row, or the cell value is missing or empty, the Target Release Version attribute of that Decision Record shall be undefined.

[SRS-070] The Decision Records Overview page shall include a "Release" column positioned between the "Target Date" and "Owner" columns, displaying each decision record's Target Release Version attribute as plain text. The column header shall carry an HTML title attribute with the value "Target Release Version". The cell shall be empty when the attribute is undefined.

[SRS-071] The Decision Records Overview page shall include a stacked bar chart visualising decision-record counts grouped by status across a trailing six-week window, placed in the second chart cell of the Decision Records Overview charts grid.

[SRS-072] The velocity chart shall display six bars, one per Friday, corresponding to the six most recent Fridays on or before the date the page is rendered, ordered left-to-right oldest-to-newest, with each X-axis label formatted as "DD-MM-YYYY".

[SRS-073] The status of a Decision Record as of a given calendar date shall be computed as the Status table row whose parseable date is the latest one that is on or before that calendar date; when multiple rows share that latest date, the row appearing later in the table shall be selected.

[SRS-074] When the earliest parseable Status table date of a Decision Record is later than the calendar date of a velocity chart bar, that Decision Record shall not contribute to that bar.

[SRS-075] Decision Records that have no Status section, no table in their Status section, or no Status table row whose Date cell is parseable as "DD-MM-YYYY" shall not contribute to any bar of the velocity chart.

[SRS-076] The set of stack segments in the velocity chart shall be the union of every distinct status text encountered across all Decision Records and all six bars, with status text compared by exact case-sensitive equality after stripping surrounding whitespace.

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

### Decision Record Links

[SRS-058] For each Controlled Item software shall show a Decision Record ID the Item is affected by in form of clickable link (decision record link).

[SRS-059] If there is more than one decision record link in the Controlled Item, the software shall show them in two steps:

1. Initially the software show the number of decision record links (clickable);
1. When User clicks on the number of decision record link, the number is replaced with the list of clickable decision record links;

[SRS-060] If User clicks on the decision record link, the software shall navigate to the Decision Record this item is affected by.

### Decision Records

[SRS-041] The software shall provide a Decision Records Overview page listing every decision record with the following columns: Sequence Number, Type, Title.

[SRS-042] When a User clicks on the Title in the Decision Records Overview, the software shall navigate to the rendered page of the selected decision record.

[SRS-048] The software shall provide a clickable "Decision Records" link in the top navigation bar of every rendered page, when at least one decision record exists in the project. The link shall lead to the Decision Records Overview page.

[SRS-051] The Decision Records Overview page shall include a "Status" column between the "Type" and "Title" columns, displaying each decision record's current status. The cell shall be empty when the current status is undefined.