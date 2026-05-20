---
title: "ENH-176: Rollback Bold Font in Decision Records Overview Table"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-05-2026 | Proposed |
|   | 20-05-2026 | Accepted |
|   | 20-05-2026 | In-Progress |
| * | 20-05-2026 | Implemented |

# Context

ENH-173 introduced bold font in the Decision Records Overview page (`build/decisions/overview.html`) for column headers, the `#` cell, and the `Title` cell — via three CSS rules scoped to `table.controlled.decisions_overview`.

After using the Almirah tool on real projects, the bold styling visibly contradicts the Specifications Index and other index pages, which render their `table.controlled` tables in normal weight. The overview now looks heavier than every other index page in the same build.

# Decision

Rollback all bold-font styling introduced by ENH-173 in the Decision Records Overview table. The overview shall render headers and all body cells in normal weight, matching the other index pages.

Remove the three CSS rules from `lib/almirah/templates/css/main.css`:

- `table.controlled.decisions_overview th { font-weight: bold; }`
- `table.controlled.decisions_overview td.item_id { font-weight: bold; }`
- `table.controlled.decisions_overview td.item_text { font-weight: bold; }`

The `decisions_overview` class on the table element is retained — it is still useful as an opt-in hook for future per-overview styling, and is also used by [[enh-175-overview-sort-id]] context.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Done | 20-05-2026 | 20-05-2026 | Three `font-weight: bold` rules scoped to `table.controlled.decisions_overview` removed from `main.css` |

Cosmetic change — no requirements or end-to-end tests added.

# Out of Scope

- Removal of the `decisions_overview` CSS class itself.
- Changes to other ENH-173 outcomes: clickable `#` hyperlink, three appended columns (`Start Date`, `Target Date`, `Owner`), and sort order from [[enh-175-overview-sort-id]] all remain.
- Bold styling in any other page or table.

# Consequences

## Positive

- Decision Records Overview visually matches the Specifications Index and other index pages.

## Negative

- Loss of the visual hierarchy added by ENH-173 (headers and `Title` no longer stand out). Accepted as the price of cross-page consistency.

## Neutral

- The `decisions_overview` class remains available for future scoped styling.

# Alternatives Considered

- **Bold the headers/Title across all index pages instead.** Rejected: wider change than the observed UX issue requires; consistency restored by rolling back is sufficient.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ENH-173](./enh-173-decisions-table-view.md) — introduced the bold styling being rolled back here
- [ENH-175](./enh-175-overview-sort-id.md) — adjacent overview-page enhancement