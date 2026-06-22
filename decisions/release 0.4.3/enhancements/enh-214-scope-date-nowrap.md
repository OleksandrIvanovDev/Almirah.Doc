---
title: "ENH-214: Keep Scope Dates On One Line"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 22-06-2026 | Proposed |
|   | 22-06-2026 | Accepted |
|   | 22-06-2026 | In-Progress |
| * | 22-06-2026 | Implemented |

# Context

In a narrow Scope column a DD-MM-YYYY date wrapped at its hyphens, rendering as three stacked fragments. Purely cosmetic.

# Decision

Tag the Scope table's Start Date and Target Date cells (located by header, [ADR-210](./../adr-210-scope-table-format.md)) with a `scope_date` class and give it `white-space: nowrap`, so the date stays on one line. No other column is affected.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  | 1 | 1 | Done | 22-06-2026 | 22-06-2026 | Emit `<td class="scope_date">` for the Start/Target Date cells in `scope_table.rb`; add the `nowrap` rule in `main.css` |

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-210](./../adr-210-scope-table-format.md) — the Scope table whose date cells are kept on one line
- [ADR-213](./../adr-213-gantt-actuals-lanes.md) — added the Start/Target Date columns these cells render

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
