---
title: "ENH-199: Work-Item Gantt Header Separator Line"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-06-2026 | Proposed |
|   | 17-06-2026 | Accepted |
|   | 17-06-2026 | In-Progress |
|   | 17-06-2026 | Implemented |
| * | 09-07-2026 | Superseded |

# Context

The work-item swimlane Gantt introduced for ADR-198 (`render_workitem_gantt` in `decisions_overview.rb`, styled by `.workitem_gantt` in `templates/css/main.css`) draws its lanes as a CSS grid with a uniform `2px` gap and no ruled separators. As a result there is no line between the header row (the sticky `Owner` corner and the day-number cells) and the lane rows below it, so the axis header blends into the first lane.

# Decision

One cosmetic change, scoped to the `.workitem_gantt` styles on the Decision Records Overview only:

- Draw a horizontal line beneath the header row, spanning the `Owner` corner and every day-number cell, separating the header from the lane rows.

The line uses the same style as the page's table border line — `1px solid #bbb`, the border the `markdown_table` and `controlled` tables already use — so the Gantt reads as part of the same visual family.

Implementation note: the line is produced with cell `border-bottom` rules on the header cells, which stay continuous only without an intervening column gap. The grid therefore drops its `2px` gap in favour of `gap: 0`, and the bars take a small `margin` so they keep the same visual separation they had before. This is a CSS-only change in `main.css`; the rendered HTML structure and the scheduling logic are untouched.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  |  |  | Done | 17-06-2026 | 17-06-2026 | In `templates/css/main.css`, `.workitem_gantt`: set `.gantt_grid` to `gap: 0` and give `.gantt_bar` a `1px` margin to preserve bar separation; add `border-bottom: 1px solid #bbb` to `.gantt_corner` and `.gantt_day_head` so a continuous rule sits beneath the header row |

This change is cosmetic, so — as with [ENH-173](./../../release%200.4.0/enhancements/enh-173-decisions-table-view.md) — there are neither requirements nor end-to-end tests in its scope.

# Out of Scope

- **Row separator lines within the `Owner` column.** Separating each owner lane label from the next was considered and dropped; the first column keeps no inter-row rules.
- **Unbolding the first column.** The `Owner` corner and per-owner labels keep their bold (`font-weight: 600`) weight.
- Row separator lines across the day columns (between lanes in the chart body); the lane bodies keep floating bars.
- Changing the existing `.workitem_gantt` container and `Owner`-column right borders (currently `#e1e4e8`); only the header rule adopts the `#bbb` table-border style.
- Any change to lane ordering, bar placement, colouring, or the scheduling logic from ADR-198.

# Consequences

## Positive

- The header is clearly separated from the first lane, so the chart is easier to read.
- The line reuses the established table-border style, keeping the Gantt visually consistent with the rest of the page.

## Negative

- Dropping the grid gap in favour of bar margins is a slightly less direct way to space bars; future spacing tweaks must adjust the bar margin rather than a single grid `gap`.

## Neutral

- No HTML or Ruby change is involved, so the ADR-198 renderer and its end-to-end tests are unaffected.

# Alternatives Considered

- **Keep the `2px` grid gap and put `border-bottom` on the header cells anyway.** Rejected: the column gap breaks the line into dashes under each day cell, so it would not read as a continuous table-style rule.
- **Insert a dedicated full-width separator element spanning all columns.** Rejected: the header cells are `position: sticky`, so a non-sticky spanning rule would scroll out of view while the headers stay pinned; a border on the sticky header cells themselves stays put.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-198](./../adr-198-workitem-gantt-visualization.md) — introduced the work-item swimlane Gantt this enhancement restyles
- [ENH-173](./../../release%200.4.0/enhancements/enh-173-decisions-table-view.md) — the precedent for an overview-scoped, cosmetic-only CSS enhancement with no requirements or tests

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)

