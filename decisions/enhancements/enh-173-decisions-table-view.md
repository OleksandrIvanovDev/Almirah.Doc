---
title: "ENH-173: Decision Records Overview Table Style and Clickable ID"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-05-2026 | Proposed |
| * | 17-05-2026 | Accepted |

# Context

The Decision Records Overview page (`build/decisions/overview.html`) renders each record as one row of a `table.controlled` table with columns `#`, `Type`, `Status`, and `Title`. Two visual aspects of the table are weaker than they could be:

- Column header text inherits the controlled-table CSS rule `font-weight: normal`, so headers read at the same weight as body cells. There is no visual hierarchy between header and body rows.
- The `Title` cell content is rendered at default weight — the most user-recognisable column has no visual emphasis.
- The `#` cell wraps the sequence number in `<b>`, so it is bold, but its anchor's `href` points to the row itself (`href="#<id>"`). Clicking it does not navigate anywhere — only the `Title` cell carries a real hyperlink to the rendered decision page.

This makes the overview feel flatter than it could and forces the user's eye and pointer to the rightmost column to drill in to a record.

# Decision

Three changes to the Decision Records Overview page, scoped to that page only:

1. Column header text shall render in bold font weight.
2. The `Title` cell content shall render in bold font weight on every row.
3. The `#` cell shall be a hyperlink to the rendered decision page, using the same target as the existing `Title` link. The anchor's `name` and `id` attributes shall be retained so that deep links into the overview row (e.g., `overview.html#adr-170`) keep working; only the `href` changes from a same-page self-reference to a navigation target.

Implementation note: to keep these styling changes from affecting the Specifications Index and other `table.controlled` instances, the overview shall opt in via an additional CSS class (for example, `table.controlled.decisions_overview`), and the new CSS rules shall target that class.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Not-Started |  |  | `DecisionsOverview` emits an additional CSS class on the table; CSS rules for bold headers and bold Title cells scoped to that class; `#` anchor's `href` targets the decision page |

The changes are considered as consmetic, so there are neither requirements nor end-to-end tests are made in scope of this enhancement.

# Out of Scope

- Bold formatting of cells other than the column headers and `Title`. The `Type` and `Status` cells, and the body of the `#` cell, keep their current weight.
- Visual changes to other index pages (Specifications Index, Traceability Matrices, etc.). The change is scoped to the Decision Records Overview.
- Reordering or renaming overview columns.
- Adding new columns to the overview.

# Consequences

## Positive

- Clearer visual hierarchy: headers stand out from body cells, and `Title` (the most informative column) catches the eye first.
- Two clickable paths into each decision page (`#` and `Title`), so readers don't have to overshoot to the right side of the row to drill in.
- Deep-link targets into the overview rows (`overview.html#adr-170`) keep working unchanged.

## Negative

- The `#` cell's previous self-anchor `href` is lost. Anyone who deliberately clicked the number expecting nothing to happen will see navigation instead — minor surprise, but a one-time adjustment.

## Neutral

- This is the first overview-scoped CSS rule. It sets a precedent for per-overview-page styling that future overview pages can opt in to via the same class mechanism.

# Alternatives Considered

- **Apply bold headers globally to `table.controlled th`.** Rejected: changes the Specifications Index and other `table.controlled` instances at the same time, which is a wider change than the user asked for and should be considered separately.
- **Wrap Title text in `<b>` inline rather than via CSS.** Rejected: presentation belongs in CSS; an inline `<b>` is harder to revise later and would have to be repeated per row in the renderer.
- **Make the entire row clickable.** Rejected: more invasive, requires JS or `<a>` wrapping every cell, and offers little advantage over making two cells clickable.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ADR-170](./../adr-170-introduce-decision-records.md) — introduces decision records and the overview page
- [ADR-172](./../adr-172-current-status-marker.md) — introduces the `Status` column and current-status marker in the overview
