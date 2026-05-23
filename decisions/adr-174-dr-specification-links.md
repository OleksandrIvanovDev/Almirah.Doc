---
title: "ADR-174: Links Between Decision Records and Specifications"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 19-05-2026 | Proposed |
|   | 19-05-2026 | Accepted |
|   | 19-05-2026 | In-Progress |
|   | 19-05-2026 | Implemented |
| * | 23-05-2026 | Reopened |

# Context

ADR-170 introduced Decision Records (ADRs / issues / enhancements) as the project-management entity that replaces Redmine tasks for Almirah framework development. To keep that first ADR feasible, its "Out of Scope" section deliberately excluded the links between decision records and specifications (and protocols).

In practice, almost every accepted decision record causes one or more controlled paragraphs in a specification to be added or revised. Today the relationship is captured only in prose — usually in the "Scope" or "References" section ("SRS-039 through SRS-048 added or revised in srs.md") — which is not machine-readable, not surfaced in the rendered specification, and not visible from the specification side at all. A reader of `srs.md` cannot tell which decision record drove a given controlled paragraph; a reader of an ADR cannot click through to the requirements it produced.

The protocols side of the gap (linking decision records to test protocols and runs) is left for a separate ADR. This decision deals only with the specification side, which is the more common case and the one where the missing traceability is felt first.

# Decision

Introduce a bidirectional link between decision records and controlled paragraphs in specifications. The link is authored in the decision record (the decision is the cause; the specification change is the effect) and rendered on both sides.

## Author-side: new "Affected Documents" section in Decision Records

Every decision record gains a new section titled "Affected Documents", placed immediately before the "References" section. The section contains a single Markdown table with three columns:

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall ... (verbatim or paraphrased proposed wording of the controlled paragraph) | >[SRS-052] |
| 2 | ... | >[SRS-053] |

Column semantics:

- **#** — sequence number within this decision record, starting at 1.
- **Proposed Text** — the text the decision proposes for the controlled paragraph. For an addition, this is the new paragraph text. For a revision, this is the proposed new wording. The text need not be byte-identical to the eventual specification text; it captures intent at the time the decision was made.
- **Req-ID** — exactly one controlled-paragraph ID in a specification, written with the existing uplink syntax `>[AAA-NNN]` (e.g., `>[SRS-052]`). The arrow form is reused so the link parses identically to the uplink in test-protocol Req-ID cells and renders as a clickable cross-document anchor.

The section is optional. A decision record with no specification impact (purely a code or process decision) may omit it. When present, the table must have at least one data row.

## Reader-side: new "DR" column on controlled-paragraph tables

In the rendered HTML of every specification, the controlled-paragraph table gains one additional column after the existing "COV" column, placed at the right edge of the table: "| # |   | UL | DL | COV | DR |"

- Header text: `DR`.
- Tooltip ("title" attribute on the header): `Decision Record`.
- Cell content: zero, one, or many clickable links to decision records that list this controlled paragraph in their "Affected Documents" table. Each link's visible text is the decision-record ID (e.g., `ADR-174`, `ISSUE-171`, `ENH-173`); the `href` points to the rendered decision-record page.
- When the cell has more than one link, the same collapse-to-count widget used by the UL and DL columns is reused (a digit that expands the full list on click). When the cell has zero links, the cell is empty.
- Visual placement: the column lives only on table class="controlled" instances rendered for Specifications. It is not added to Protocols, the Decision Records Overview, the Specifications Index, or any traceability matrix.

## Linking model

The link is computed during the existing Parse → Link phase of the pipeline.

1. **Parse**: for each Decision document, locate the "Affected Documents" section (heading text equals "Affected Documents", case-sensitive, at any heading level). If present, parse its table and collect a list of `(step, proposed_text, req_id)` elements on the Decision instance — e.g., `decision.affected_items`. Decision Records with no such section yield an empty list. The first element (step) represents an id of the link as it is done for protocols (not used for visualization here, but required for implementation)
2. **Link**: extend the linker so that for each `(_, req_id)` pair, the target ControlledParagraph in the referenced specification gets the owning Decision appended to a new `decision_record_links` collection (analogous to the existing `coverage_links` used to populate the COV column).
3. **Check**: a Req-ID that does not resolve to an existing controlled-paragraph ID is reported as a broken reference, using the same broken-reference reporting that already covers uplinks and coverage links. The owning Decision Record is named in the error so the author can fix it at the source.
4. **Render** — Decision Record HTML: the "Affected Documents" section renders as a normal Markdown table, except the Req-ID cell is rendered as a cross-document hyperlink to the controlled paragraph (same href style as a UL cell).
5. **Render** — Specification HTML: the controlled-paragraph table emits the new "DR" column (title="Decision Record") after the COV column, and a class="item_id" per row containing the resolved decision-record links (or empty). Specifications whose paragraphs have no decision-record links still get the new column — the column header is always present on Specification documents so the table shape is uniform across the project.

## Out-of-scope clarifications carried over from ADR-170

ADR-170 excluded links between decision records and both specifications and protocols. This ADR closes the specifications half of that gap. Protocols remain out of scope here and will be addressed in a future decision record.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Proposed | 19-05-2026 |  | New SRS items in `srs.md` covering: the "Affected Documents" section convention; parsing of its table into `affected_items` on the Decision class; the bidirectional link computed during the Link phase; the new "DR" column on Specification controlled-paragraph tables (header text, tooltip, position after COV, collapsed-count widget for multi-link cells); broken-reference reporting for unresolved Req-IDs |
| Code | Proposed | 19-05-2026 |  | Decision parser learns the "Affected Documents" section; doc_linker populates `decision_record_links` on ControlledParagraph; controlled_paragraph.rb emits the DR column header and cell on Specification documents only; Decision renderer turns Req-ID cell text into a cross-document anchor |
| Tests | Proposed | 19-05-2026 |  | End-to-end tests under `spec/decisions_spec.rb` (or a new file) for: parsing the new section; bidirectional link resolution; rendered DR column in `srs.html` including multi-link collapse; rendered Req-ID anchors in a decision record's "Affected Documents" section; broken-reference detection for an unresolved Req-ID; absence of the DR column on Protocol HTML |

# Out of Scope

- Links between decision records and **test protocols** or **test runs**.
- Backfilling existing decision records (ADR-170, ISSUE-171, ADR-172, ENH-173) with an "Affected Documents" section.
- Synchronising the "Proposed Text" cell with the actual specification text. The cell records intent at the time the decision was made and is not re-validated against the live specification — divergence between proposed and final wording is expected and is not flagged as an error.
- Coverage of the new DR column by the existing Coverage matrix document. The DR column is rendered only inline on specification pages; no top-level matrix page is added.
- Multi-target Req-ID in one row (e.g., one row mapping to two controlled paragraphs). Each row carries exactly one Req-ID; authors who need to affect N paragraphs write N rows.

# Consequences

## Positive

- A reader of any specification can see, per controlled paragraph, which decision records drove its creation or revision.
- A reader of any decision record can click straight from the "Affected Documents" table into the affected paragraph.
- The convention reuses the existing `>[AAA-NNN]` uplink syntax and the existing Link phase plumbing, so the change is mostly additive rather than a new mechanism.
- Closes one of the two gaps explicitly left open by ADR-170, without forcing the protocols half to be designed at the same time.

## Negative

- All specification documents grow by one column.
- Authors must remember to add rows to "Affected Documents" when they create or revise a controlled paragraph as part of a decision. There is no automatic enforcement that every new SRS item has a corresponding decision-record row.

## Neutral

- The same mechanism will later host the analogous "DR-from-protocols" link when the protocols half is decided — the linker code path is shared, only the parse and render endpoints differ.
- Existing decision records without an "Affected Documents" section continue to render unchanged; the section is optional.
- The DR column is rendered only on Specification documents, not Protocols, mirroring the column-set scoping decisions already made for UL/DL/COV.

# Alternatives Considered

- **Author the link in the specification rather than in the decision record.** Add a back-syntax to controlled paragraphs (e.g., `<[ADR-174]`) that points at the originating decision. Rejected: the decision is the cause and the specification change is the effect — the link is more naturally maintained where the cause is described. Decision records already carry the rationale and timeline; specifications should stay focused on the requirement text. In addition to that requirements may change or be removed that may either overload the specification (multiple DRs for one SRS) or brake the link (in the case of SRS removal).
- **Encode the link inside the existing "Scope" table of the decision record** by adding a Req-ID column to it. Rejected: the Scope table tracks Requirements / Code / Tests rollup status across an entire ADR (one row per work item, not per affected paragraph), and overloading it would conflate two different granularities. A separate "Affected Documents" table keeps each concern in one place.
- **Drop the "Proposed Text" column and keep only `#` and `Req-ID`.** Rejected: the proposed text is the part of the decision record that a reviewer reads when judging whether the decision is well-formed, before the specification is even updated. Forcing the reviewer to click through to the spec defeats the point of capturing intent at decision time.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.0 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall recognise the "Affected Documents" section in Decision Records that indicates the list of Controlled Items with their text updated or created in scope of the Decision Record. | >[SRS-052] |
| 2 | The software shall recognise the "Affected Documents" section as a single markdown table with the following columns in order: "#", "Proposed Text", and "Req-ID". | >[SRS-053] |
| 3 | The software shall accept the External Controlled Item ID in the "Req-ID" column of the "Affected Documents" table in the form ">[BBB-NNN]", using the same syntax as test step references in Test Protocols. | >[SRS-054] |
| 4 | When a Decision Record contains an "Affected Documents" section, the software shall establish a link from each row of the section to the referenced Controlled Item in the target Specification document. | >[SRS-055] |
| 5 | The software shall report a broken reference if the "Req-ID" column in an "Affected Documents" table refers to a Controlled Item ID that does not exist, naming the owning Decision Record in the report. | >[SRS-056] |
| 6 | The software shall render the "Req-ID" cell in the "Affected Documents" table of a Decision Record as a clickable link to the referenced Controlled Item in the Specification document. | >[SRS-057] |
| 7 | For each Controlled Item software shall show a Decision Record ID the Item is affected by in form of clickable link in uppercase (decision record link). | >[SRS-058] |
| 8 | If there is more than one decision record link in the Controlled Item, the software shall show them in two steps. | >[SRS-059] |
| 9 | If User clicks on the decision record link, the software shall navigate to the Decision Record this item is affected by. | >[SRS-060] |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records and explicitly defers decision-record-to-specification (and -to-protocol) linking; this ADR closes the specification half of that gap
- [ADR-172](./adr-172-current-status-marker.md) — example of an ADR extending Decision Record parsing and rendering, used as a precedent for the parse/link/render split adopted here
- SRS-039 through SRS-051 in [srs.md](./../specifications/srs/srs.md) — current requirements covering decision records; new SRS items added under this ADR will extend that range
