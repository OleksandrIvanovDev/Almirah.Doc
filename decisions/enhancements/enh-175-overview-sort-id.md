---
title: "ENH-175: Sort Decision Records Overview Table by ID"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-05-2026 | Proposed |
|   | 20-05-2026 | Accepted |
|   | 20-05-2026 | In-Progress |
| * | 20-05-2026 | Implemented |

# Context

The Decision Records Overview page (`build/decisions/overview.html`) renders one row per decision record with columns `#`, `Type`, `Status`, `Title`, `Start Date`, `Target Date`, `Owner` (as established by ADR-170, ADR-172, and ENH-173).

Today the rows appear in the order in which the underlying `Decision` instances were discovered while walking the `decisions/` folder tree — top-level `adr-*.md` files first, then `enhancements/enh-*.md`, then `issues/issue-*.md`, each group in filesystem (and therefore filename-string) order. The numeric sequence shared across all decision-record types (per ADR-170 — the sequence continues from Redmine's last task #169) is not used for sorting at all.

As the project accumulates decision records, this order becomes unhelpful:

- A reader who knows a record's ID (e.g., "ADR-172") cannot scan top-to-bottom and stop when the sequence number passes the target — the rows are not in numeric order.
- Records that belong to the same conversation (e.g., ADR-170 → ISSUE-171 → ADR-172 → ENH-173 → ADR-174) are scattered across type groups, even though their adjacent numbers reflect that they were created in close succession.
- New records appear in unpredictable positions on the overview depending on which folder they land in.

# Decision

The rows of the Decision Records Overview table shall be sorted in ascending order by the numeric sequence number embedded in each record's ID — the digits portion of `<type>-<digits>` (e.g., `adr-170` → 170, `issue-171` → 171, `enh-173` → 173). The type prefix is ignored when sorting; the sequence is shared across all decision-record types per ADR-170.

Sorting is performed once, in the `DecisionsOverview` renderer, immediately before rows are emitted. The sort is stable: in the (not expected) case of two records sharing the same sequence number, their relative order is whatever the discovery walk produced.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Done | 20-05-2026 | 20-05-2026 | `DecisionsOverview` sorts the list of `Decision` instances by the integer parsed from the digits portion of the decision-record ID before emitting table rows |

The change is considered cosmetic — there are no new requirements and no end-to-end tests are added in scope of this enhancement.

# Out of Scope

- Descending or user-controlled sort order. The overview is always ascending by sequence number.
- Multi-column sort (e.g., by `Status` then by `#`) or interactive client-side sorting via JavaScript.
- Sorting any other index page (Specifications Index, Traceability Matrices, etc.). The change is scoped to the Decision Records Overview.
- Validating that sequence numbers are unique across decision records, or warning when a gap appears in the sequence.
- Changes to the rendered Decision Record pages themselves — only the row order in the overview table changes.

# Consequences

## Positive

- A reader who knows a record's ID can find it on the overview with a quick top-to-bottom scan.
- Records created in close succession (which usually relate to one another) appear adjacent on the overview, mirroring the order in which the project's thinking evolved.
- The row position of a record is stable and predictable: it depends only on its sequence number, not on the type-folder it happens to live in.

## Negative

- None identified. The change is additive and does not alter rendered content beyond row order.

## Neutral

- Sorting by sequence number deliberately interleaves types (ADR, ISSUE, ENH). A reader who wants to see only one type can filter by the `Type` column visually; a dedicated per-type overview is not introduced here.

# Alternatives Considered

Not applicable. Just a simple enhancement.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ADR-170](./../adr-170-introduce-decision-records.md) — introduces decision records, the shared sequence number across types, and the overview page

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 