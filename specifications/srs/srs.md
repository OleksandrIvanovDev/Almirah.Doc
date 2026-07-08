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

[SRS-102] The software shall expose a Target Date attribute on each Decision Record, computed as the latest calendar date found in the Date column of the Decision Record's Status table and in the Target Date column of the Decision Record's Scope table.

[SRS-103] The software shall recognise dates in the form "DD-MM-YYYY" when computing the Target Date attribute of a Decision Record. Cells whose contents do not match this form shall be skipped without raising an error.

[SRS-104] The software shall identify the Date column of a Decision Record's Status table and the Target Date column of a Decision Record's Scope table by their header text, case-sensitive, and not by column position.

[SRS-105] When neither the Status table's Date column nor the Scope table's Target Date column of a Decision Record contains a date matching the recognised form, the Target Date attribute of that Decision Record shall be undefined.

[SRS-106] The Decision Records Overview page shall render the Target Date attribute of each Decision Record in the existing Target Date column, formatted as "DD-MM-YYYY". The cell shall be empty when the Target Date attribute is undefined.

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

[SRS-083] The Decision Records Overview page shall include a horizontal bar chart visualising the number of Decision Records grouped by their current status, placed in the third chart cell of the Decision Records Overview charts grid.

[SRS-084] The current-status distribution chart shall count each Decision Record under its current status — the status value of the row marked with "*" in the record's Status table — with status text compared by exact case-sensitive equality after stripping surrounding whitespace.

[SRS-085] The current-status distribution chart shall group every Decision Record whose current status is undefined under a single "Undefined" category, so that records with a missing or ambiguous current-status marker are surfaced for correction.

[SRS-086] The current-status distribution chart shall use a linear value scale and shall display each category's record count within its bar label, so that categories with small counts remain legible alongside categories with large counts.

[SRS-087] The categories of the current-status distribution chart shall be ordered by their first appearance in Decision Record parse order, with the "Undefined" category, when present, placed last.

## Risk Records

[SRS-166] The software shall collect risk records from the first-level subfolders of a risks folder at the project root, each subfolder forming a risk registry, each record identified by the letters-digits prefix of its file name, and shall render each record to its own HTML page.

>Example 1: "risks/project/prjr-001-expertise-loss.md" — registry is "project", ID is "prjr-001", rendered to "build/risks/project/prjr-001.html"

>Example 2: a "risks/product/overview.md" file is a registry preface, not a risk record.

[SRS-167] The software shall derive a risk record's current status from the row of its Status table marked with an asterisk in the leading column, in the same way as for decision records.

>Example: a Status table row containing "*" in the leading column and "Mitigating" in the Status column makes "Mitigating" the current status of the risk record.

[SRS-168] The software shall render each risk registry to an overview page consisting of the registry's overview.md content followed by a register table with one row per risk record, whose leading columns are the record ID, displayed uppercased and linked to the record page, and the record title with the record's own leading ID prefix removed, and whose further columns are configured per registry and filled from each record's section whose heading matches the column name, the Status column being filled from the record's current lifecycle status.

>Example 1: a "risks/project" registry configured in project.yml with "columns: [Probability, Impact, Status]" renders "build/risks/project/overview.html" with the table columns #, Title, Probability, Impact, Status; the Probability and Impact cells hold the content of each record's "# Probability" and "# Impact" sections.

>Example 2: the record "secr-001-sql-injection.md" titled "SECR-001: SQL Injection in Search" shows "SECR-001" in the ID column and "SQL Injection in Search" in the Title column; a title that does not start with the record's own ID renders unchanged.

>Example 3: a record without an "# Impact" section gets an empty Impact cell.

>Example 4: a registry with no entry under the "risks:" root in project.yml renders the implicit columns plus Status only.

[SRS-169] The software shall append to a risk register table one computed column per RPN group configured for the registry, each group having a name and an ordered list of input section names, the cell value being the product of the record's numeric input section values and blank when any input is missing or not numeric.

>Example 1: a group named "Initial" with "inputs: [Severity, Occurrence, Detection]" appends an "Initial RPN" column; a record whose sections hold 8, 3, and 2 shows 48.

>Example 2: a record whose "# Occurrence" section holds "TBD" gets a blank "Initial RPN" cell, not zero.

>Example 3: a group with the single input "CVSS Score" surfaces that section's value unchanged as a rankable RPN column.

[SRS-170] The software shall colour an RPN cell by the group's optional thresholds: acceptable style at or below the acceptable bound, unacceptable style at or above the unacceptable bound, caution style between the bounds, and no colouring when thresholds are not configured.

>Example: with "acceptable: 20" and "unacceptable: 100", the value 18 renders in the acceptable style, 48 in the caution style, and 120 in the unacceptable style.

[SRS-171] The software shall parse a risk record's Affected Documents section as a controlled table whose Req-ID column carries uplinks to controlled paragraphs, as for decision records, and shall render the register table's Affected Documents column as only the distinct linked controlled-paragraph IDs, each a clickable link to its paragraph.

>Example 1: a risk record row "| 1 | The software shall sanitise search input. | >[SRS-123] |" links the record to [SRS-123]; the SRS paragraph shows the risk record among its downlinks, and the register cell shows "SRS-123" as a clickable link without the Proposed Text.

>Example 2: a Req-ID referencing a non-existing controlled paragraph renders in the register cell in the existing broken-link style instead of being dropped.

[SRS-172] The software shall add a Risks entry to the top menu bar, present only when the project has at least one risk registry, leading to a summary page holding one row per registry with the columns Risk Registry, Total Risks, Open Risks, Highest RPN and Average RPN, where the Risk Registry cell shows the registry preface's frontmatter title, falling back to the registry folder name when no preface title exists, linked to its registry page, the open count excludes records whose current status is Closed, and the RPN aggregates are computed over the registry's leading RPN group ignoring blank values.

>Example 1: a registry with four records of which one is Closed shows Total Risks 4 and Open Risks 3; a record without a current-status marker counts as open.

>Example 2: a "risks/security" registry whose overview.md frontmatter carries the title "Security Risk Register" shows "Security Risk Register" in the Risk Registry cell; a registry without a preface, or whose preface has no frontmatter title, shows its folder name.

>Example 3: a registry whose leading RPN group carries thresholds renders its Highest RPN cell in the threshold style of that value; a registry with no RPN configuration shows blank Highest and Average RPN cells.

>Example 3: a project without a risks folder shows no Risks menu entry and no summary page.

## Planning

[SRS-107] The Decision Record Scope table shall support work-item rows — including an Analysis row — each carrying an Owner and a Status whose value is one of To Do, In-Progress, or Done. These per-row statuses shall be independent of the Decision Record's hand-marked lifecycle status.

[SRS-108] The software shall associate each Scope row with the owner named in its Owner column, and shall expose on each Decision Record the distinct, first-seen-ordered list of its rows' owners. The Owner column shall be identified by its header text, case-sensitive, and not by column position.

[SRS-109] When a Scope row has no owner, that row shall contribute no owner; when a Decision Record has no Scope table, no Owner column, or an empty Owner column, its distinct owner list shall be empty.

[SRS-110] The Decision Records Overview page shall render the distinct owner list of each Decision Record in the existing Owner column, comma-separated when there is more than one owner, and empty when the list is empty.

[SRS-111] The Decision Records Overview page shall render a Work-In-Progress by Owner chart with one bar for every owner named in any Decision Record's Scope table, each bar's length being the number of Scope rows, across all Decision Records, whose row Status is In-Progress and whose Owner is that owner, taken as zero when the owner has none, and shall draw a reference line at the configured work-in-progress freeze limit.

[SRS-112] The software shall read an optional planning work-in-progress limit from the project configuration. When the limit is absent or invalid, the software shall apply a default of 2.

[SRS-113] The Decision Record Scope table shall support a leading step-number column that establishes the order of work-item rows, where rows sharing a step number are concurrent and, when the column is absent, the intrinsic row order applies.

[SRS-114] The software shall report, without failing the build, each Scope row that is In-Progress or Done while any lower-numbered step in the same Decision Record is not Done, naming the record, the row, and the blocking step.

[SRS-115] The Decision Record Scope table shall support a per-row Depends On column, identified by header text and not position, in which the row carrying a reference is the dependent work item; each reference to another Decision Record shall be resolved to that record's work item whose activity type (its Item) matches the dependent row's, falling back to the record's nearest earlier activity type when no exact match exists, and the row's Owner shall not affect this resolution. A Decision Record shall not reference its own rows.

[SRS-116] The software shall define a Scope row's predecessors as its lower-numbered same-record steps together with its resolved cross-record work items, and shall consider the row fully kitted when every predecessor's Status is Done, including when it has no predecessors.

[SRS-117] The software shall report, without failing the build, each started Scope row (Status In-Progress or Done) whose resolved cross-record predecessor work item is not Done, naming the record, the row, and the unsatisfied predecessor.

[SRS-118] The software shall report, without failing the build, each Depends On reference that does not resolve to a managed Decision Record, naming the referencing Decision Record and the unresolved reference.

[SRS-119] The Decision Records Overview page shall render a Kit column indicating, for each Decision Record, whether it has no declared prerequisites, is fully kitted, or is blocked by an unsatisfied prerequisite.

[SRS-120] The Decision Record Scope table shall support two optional estimate columns, a focused estimate and a safe estimate, each expressing a work-item row's effort as a non-negative number of working days. The columns shall be identified by their header text, case-sensitive, and not by column position.

[SRS-121] The software shall treat each Scope row's focused estimate as its scheduling duration, treating empty or unparseable estimate cells as zero.

[SRS-122] For each decision-record group, the software shall construct a planning network whose nodes are the not-Done Scope rows of the records in that group, with intra-record edges following the step-number order and cross-record edges placing each row that carries a Depends On reference after its activity-type-aligned predecessor work item in the referenced record.

[SRS-123] The software shall schedule the planning network with a deterministic resource-levelling rule in which each row starts at the earliest day, at or after the finish of all its predecessor rows, where its owner's lane holds no other row for the row's whole duration, so that a row may fill an idle gap between rows placed earlier and rows sharing the same owner never run concurrently.

[SRS-124] The software shall identify the critical chain of a decision-record group as the sequence of Scope rows that determines the group's completion in the resource-levelled schedule.

[SRS-125] The software shall compute a project buffer for each decision-record group as the configured buffer ratio multiplied by the aggregated safety along the critical chain, where each row's safety contribution is its safe estimate minus its focused estimate clamped to zero, rounded up to a whole working day.

[SRS-126] The software shall read an optional planning buffer ratio from the project configuration, applying a default of 0.5 when the value is absent or outside the range greater than 0 and at most 1.

[SRS-127] The software shall render the per-decision-record-group critical chain, project buffer, and projected duration on a dedicated Critical Chain page rather than on the Decision Records Overview, indicating when a group has no estimated work.

[SRS-136] The Decision Records Overview page shall render a work-item schedule between the status charts and the records table, in a scrollable container whose leading Owner column does not scroll horizontally and whose remaining columns are indexed by day, omitting the schedule when no work item can be placed.

[SRS-137] The software shall draw each Scope work item across all Decision Records as a bar on its Owner's lane whose length in day columns is its focused estimate rounded up to a whole day, with a one-day minimum so that an unestimated row remains visible.

[SRS-138] The software shall start each work item's bar no earlier than the latest finish among its predecessors, counting both lower-numbered same-record steps and resolved cross-record dependencies.

[SRS-139] The software shall place work items sharing an Owner so that their bars do not overlap on that Owner's lane, serialising the lane by resource levelling, and shall produce the same schedule on repeated runs.

[SRS-140] Each work-item bar shall indicate its row Status, and a started work item whose cross-record predecessor is not Done shall be visually emphasised, consistent with the Overview Kit cell.

[SRS-164] The Decision Records Overview Gantt shall visually distinguish the work-item bars lying on their group block's critical chain, using a channel separate from the row-Status colour and the blocked-work-item emphasis so the three can be read together.

[SRS-165] The Decision Records Overview Gantt shall provide, behind a toggle hidden by default, a per-owner tracking lane that draws each work item's committed window — its Scope Start to Target dates, running to the current date when no target is given — and its real logged span — its earliest to latest Effort date — on the same calendar axis as the computed schedule, extending that axis to keep an overrun visible.

[SRS-141] The Decision Records Overview work-item schedule shall be segmented into one block per decision-record group, the blocks laid left to right in the groups' folder-encounter order with a gutter column between adjacent blocks so they never overlap, each block carrying its own day-index axis beginning at one.

[SRS-142] The software shall render a group band row between the day-header and the resource lanes, with one labelled cell per group spanning that group's day columns.

[SRS-143] The software shall schedule each group's work items independently with per-owner resource levelling scoped to the group, treating any predecessor belonging to another group as an already-available input rather than a scheduled work item.

[SRS-144] The Decision Records Overview shall render a Buffer lane as the last row below the resource lanes, drawing one buffer bar per group positioned after that group's last work item, its length the group's computed project buffer.

[SRS-145] The software shall produce the same segmented layout, block order, and per-group schedule on repeated runs over unchanged input.

[SRS-146] The software shall place a "Critical Chain" link in the top navigation menu immediately after the Decision Records link, pointing at the dedicated Critical Chain page, and shall show it exactly when the Decision Records link is shown.

[SRS-147] The software shall derive a consensus owner order across all Decision Records by tallying, for each record's distinct owner sequence, every ordered pair of owners that appears earlier-before-later, and ranking the owners by Copeland score — each owner scoring one for every other owner it precedes more often than it follows, minus one for every owner it follows more often — with the first-seen owner order as a tiebreak so the result is identical across repeated runs over unchanged input.

[SRS-148] The Decision Records Overview Gantt shall order its resource lanes by the consensus owner order, while the Work-In-Progress by Owner chart shall retain its descending-in-progress-count order.

[SRS-128] A Decision Record shall support an optional Effort section containing an append-only table whose columns are a date in DD-MM-YYYY form, an Item tying the entry to a Scope row, an optional owner, a non-negative number of hours, and an optional note. The columns shall be identified by header text, not by position.

[SRS-129] The software shall expose on each Decision Record the total actual effort, an as-of-date total, and a per-Scope-row as-of-date effort summing the Hours of entries whose Item matches the row and whose date is on or before the given date.

[SRS-130] The software shall read an optional planning hours-per-day value from the project configuration, applying a default of 8 when absent or non-positive, and shall use it to convert logged hours to working days.

[SRS-131] For each decision-record group, the software shall compute the critical-chain completion as the focused-estimate-weighted fractional progress of the group's critical-chain rows, where a row's progress is its as-of-date actual days over its focused estimate clamped to one, and a row whose Status is Done credits full progress at the current date.

[SRS-132] For each decision-record group, the software shall compute the buffer consumption as the percentage of the project buffer consumed, where consumed days equal the sum over the critical-chain rows of the amount by which each row's as-of-date actual days exceed its focused estimate, clamped to zero, and where the consumption may exceed one hundred percent.

[SRS-133] The Critical Chain page shall render, per decision-record group with an estimated critical chain, a fever chart plotting buffer consumption against critical-chain completion, placed beside that group's critical-chain table.

[SRS-134] The fever chart shall divide its area into green, yellow, and red zones by the conventional one-third and two-thirds diagonal boundaries, indicating respectively on-track, watch, and act conditions.

[SRS-135] The fever chart shall render a trail of points, one per recent Friday, each computed from the as-of-date actual effort of the critical-chain rows on both axes, showing the trajectory of the decision group toward its current zone.

[SRS-161] The Critical Chain page shall display, beside each decision-record group's project buffer figure, the buffer consumed as a percentage and as the consumed working days out of the group's baseline project buffer, omitting the line when the group has no baseline buffer to consume.

[SRS-162] Each fever chart point shall carry the calendar date it was sampled on — a recent Friday for each trail point and the render date for the live point — and the chart's point tooltip shall present that date followed by the point's completion and consumption values.

[SRS-149] The software shall read an optional planning start date from the project configuration in DD-MM-YYYY form, using it as the calendar anchor for the first working day and falling back to the render date when the value is absent or unparseable.

[SRS-150] The software shall read an optional planning holidays list from the project configuration, each a DD-MM-YYYY date, and shall treat those dates, together with Saturdays and Sundays, as non-working days.

[SRS-163] The software shall read an optional per-group planning start date for each decision-record group from the project configuration in DD-MM-YYYY form, and shall anchor that group's Gantt block calendar — the dates labelling its day columns and the start its projected completion date is counted from — at that start date, falling back to the project start date when the group declares none. The blocks remain laid out sequentially (SRS-141); a group's start date sets the dates shown for its block, not the block's position.

[SRS-151] The software shall map each working-day index of a decision-record group's schedule to a calendar date counted from that group's planning start date (SRS-163), skipping non-working days, so that the Nth working day falls on the Nth working date on or after that anchor.

[SRS-152] The Decision Records Overview Gantt shall render a day column for each working day and for each holiday that falls on a weekday, and shall not render columns for Saturdays and Sundays.

[SRS-153] The Decision Records Overview Gantt shall render a calendar header naming the month spanning its day columns and numbering the day of the month for each column.

[SRS-154] The software shall span each work-item bar and each buffer bar across the business-day columns from its first to its last working day inclusive, so that bars cover any weekday-holiday columns they cross without counting them as work.

[SRS-155] The Critical Chain page shall render, per decision-record group with an estimated critical chain, a projected completion date obtained by counting the projected duration in working days from that group's planning start date (SRS-163) across non-working days.

[SRS-158] The Decision Records Overview Gantt shall mark each weekday-holiday column as non-working, shading it distinctly and drawing work-item bars across it.

[SRS-159] The Decision Records Overview Gantt shall highlight Friday columns to indicate the boundaries of the five-day working week.

[SRS-160] The Decision Records Overview Gantt shall mark the current date with a full-height vertical rule at the column of the first working day on or after the current date, drawn only in a group block whose calendar range contains the current date and omitted from blocks the current date falls before or after.

## Console Output

[SRS-079] While processing a project, the software shall emit a concise progress summary to standard output consisting of one line per processing phase, each line pairing a phase label with the number of items processed in that phase.

[SRS-080] The software shall print the path of the generated index page as the final line of the progress summary.

[SRS-081] The software shall apply ANSI colour to its progress output only when standard output is an interactive terminal, and shall emit uncoloured text otherwise.

[SRS-082] When composing the generated index path for the progress summary, the software shall resolve the project directory argument to a normalised path free of duplicated or missing path separators, regardless of the form of the argument. The path shall be shown relative to the current directory as "./build/index.html" when the project directory resolves to the current working directory, and as an absolute path otherwise.

## Cross-Document Links

[SRS-088] The software shall resolve a Markdown link whose target, after resolving the link's relative path against the linking document's source directory, is a managed document's source file, and shall render it in the HTML output as a relative link to that target document's generated page.

[SRS-089] The software shall preserve the on-disk relative validity of a Markdown cross-document link, so that the link in the source Markdown navigates to the target Markdown file while the generated HTML link navigates to the corresponding generated page.

[SRS-090] The software shall support a double-bracket cross-document link of the form `[[target]]` that resolves the target to a managed document by its unique document identifier or filename, independent of the document's folder location.

[SRS-091] The software shall support an alias in a double-bracket link of the form `[[target|display text]]`, rendering the display text as the visible link text.

[SRS-092] The software shall support an anchor fragment in a cross-document link, written as `target#fragment` in a Markdown link and as `[[target#fragment]]` in a double-bracket link, producing an HTML link to that fragment within the target document's generated page.

[SRS-093] The software shall compute the relative URL of every internal link from the location of the generated page that contains the link to the location of the target's generated page, using forward-slash separators.

[SRS-094] The software shall report a cross-document link whose target cannot be resolved to a managed document as a broken reference, naming the linking document, and shall render it as a visibly broken link without aborting the build.

[SRS-095] The software shall leave links with an external scheme (such as `http`, `https`, or `mailto`) unchanged and shall not treat them as cross-document targets.

## HTML Output Safety

[SRS-096] The software shall HTML-escape all author-supplied literal text rendered into element content — including paragraph, heading, blockquote, table-cell, and fenced code block text — so that markup present in the source Markdown is rendered as inert text and cannot introduce HTML elements.

[SRS-097] The software shall escape author-supplied values interpolated into HTML attributes — including an image's source and alternate text and a link's address and visible text — so that the value cannot terminate the attribute or introduce additional attributes or elements.

[SRS-098] The software shall admit a link or image URL only when it is a relative reference or uses an allowed scheme (`http`, `https`, or `mailto`), and shall render any other scheme (such as `javascript`, `data`, or `vbscript`) inert rather than emitting it.

[SRS-099] The software shall render author-derived values displayed by client-side scripts — including search results and the image caption — using DOM text and attribute interfaces rather than HTML parsing, and shall admit a URL it assigns to a link only when it is a relative reference or uses an allowed scheme, so that indexed or displayed content cannot execute as script.

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

[SRS-058] For each Controlled Item software shall show a Decision Record ID the Item is affected by in form of clickable link in uppercase (decision record link).

[SRS-059] If there is more than one decision record link in the Controlled Item, the software shall show them in two steps:

1. Initially the software show the number of decision record links (clickable);
1. When User clicks on the number of decision record link, the number is replaced with the list of clickable decision record links;

[SRS-060] If User clicks on the decision record link, the software shall navigate to the Decision Record this item is affected by.

### Decision Records

[SRS-041] The software shall provide a Decision Records Overview page listing every decision record with the following columns: Sequence Number, Type, Title.

[SRS-042] When a User clicks on the Title in the Decision Records Overview, the software shall navigate to the rendered page of the selected decision record.

[SRS-048] The software shall provide a clickable "Decision Records" link in the top navigation bar of every rendered page, when at least one decision record exists in the project. The link shall lead to the Decision Records Overview page.

[SRS-051] The Decision Records Overview page shall include a "Status" column between the "Type" and "Title" columns, displaying each decision record's current status. The cell shall be empty when the current status is undefined.

### Search

[SRS-077] The software shall provide a full-text search on the Index page that, for a user-entered term, returns matching specification content showing the source document title, a link to the containing section, and a text snippet.

[SRS-078] When no indexed content matches the entered search term, the software shall indicate that there are no matches.

### Navigation

[SRS-100] When the User navigates to an in-page anchor, the software shall position the target so that it is fully visible and not obscured by the fixed top navigation bar.

[SRS-101] When the document sections tree in the navigation pane is taller than the viewport, the software shall allow the pane to scroll so that every item, including the last one, can be brought fully into view.