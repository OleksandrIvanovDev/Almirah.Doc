---
title: "ADR-213: Plan-vs-Actual Tracking Lanes on the Gantt"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 22-06-2026 | Proposed |
|   | 22-06-2026 | Accepted |
|   | 22-06-2026 | In-Progress |
| * | 22-06-2026 | Implemented |

# Context

The Decision Records Overview Gantt ([ADR-198](./adr-198-workitem-gantt-visualization.md)) draws one bar per Scope work item, one lane per owner, segmented into per-group blocks ([ADR-201](./adr-201-gantt-group-segmentation.md)) on a working-day calendar axis ([ADR-205](./adr-205-calendar-gantt-view.md), [ADR-206](./adr-206-working-day-columns.md)) anchored at each group's start ([ADR-211](./adr-211-group-start-dates.md)). Every bar's position is computed by `GanttScheduler` — a resource-levelled forward pass over the dependency network ([ADR-194](./adr-194-full-kit-readiness.md), [ADR-195](./adr-195-critical-chain-buffer.md)). That bar is therefore the *computed ideal schedule*: where the constraint says the work should fall, not where the author committed it nor where it actually happened.

Two authored facts the Gantt already parses but never plots against that schedule:

- The Scope table's per-row `Start Date` and `Target Date` — the author's *committed window* for the row. These are read at the record level for the overview table but are not carried on the work item, so a row's own committed window is invisible on the chart.
- The `# Effort` log ([ADR-196](./adr-196-buffer-fever-chart.md)) — dated hours per Scope `Item`. The fever chart sums these, but the calendar span of real logged work (its first to its last dated entry) is never drawn.

A reader comparing where a bar sits to where the work was actually committed and logged has to leave the Gantt and cross-read the Scope and Effort tables row by row. The data is in hand; only the comparison is missing. Three distinct timelines exist for every row — the computed schedule (the existing bar), the committed window (Scope dates), and the real logged span (Effort dates) — and only the first is drawn.

# Decision

Add an optional second lane per owner — the *tracking* lane — that draws the committed window and the real logged span beside the computed schedule, behind a toolbar toggle.

- **A toolbar above the chart.** A thin control strip is stuck to the top edge of the Gantt, outside the grid, carrying a single button that reads "Show Actuals" / "Hide Actuals" by current state. The strip is the home for future Gantt controls; this decision adds only the one button.
- **A tracking lane per owner.** Each owner keeps its existing lane, labelled with the bare role (BA, DEV, …) and unchanged. Immediately below it sits a second lane for the same owner, labelled with the role plus " (tracking)". The tracking lanes are hidden by default and toggled as a set by the button.
- **Two layers on a tracking bar.** For each work item, on its owner's tracking lane:
  - A **committed layer** (grey, behind): the row's Scope `Start Date` to `Target Date`. When only a `Start Date` is given, it runs from there to the earlier of today and the group's last calendar day — so a missing target reads as an open, overrunning commitment rather than a point, surfacing the reporting gap.
  - A **logged layer** (blue, in front): the calendar span from the earliest to the latest `# Effort` date crediting this row's `Item`. Absent when the row has no dated effort.
  - The two layers coincide or diverge by the data; both are acceptable. The bar's label is drawn on each layer so it stays readable when one layer is absent or shorter.
- **The axis covers the actuals.** Because a committed target or a logged date can fall past the computed schedule plus its buffer, each block's calendar width is extended to span the latest of its computed finish, every committed and logged date it carries, and today — so an overrun is drawn, never clipped at the buffer edge. Committed and logged dates landing on a weekend or holiday snap to the nearest business-day column (the axis carries no non-working columns, ADR-206).

This is additive to the render. The schedule, the existing bars, the chain ([ADR-212](./adr-212-critical-chain-highlight.md)), and the buffer are unchanged; the tracking lanes and the wider axis are new, and the lanes collapse to nothing when hidden so the default chart is unchanged.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 1 | Done | 22-06-2026 | 22-06-2026 | State SRS-165: the Gantt shall provide a toggled per-owner tracking lane drawing each work item's committed window (Scope Start/Target dates) and real logged span (Effort dates) beside the computed schedule |
| 2 | Code | DEV | >[ADR-210], >[ADR-196] | 3 | 5 | Done | 22-06-2026 | 22-06-2026 | Parse per-row Start/Target dates onto the WorkItem; add an Effort date-range reader per Item; extend each block's calendar width to cover committed, logged, and today dates; emit interleaved hidden "(tracking)" lanes with the grey committed and blue logged layers; add the toolbar and the show/hide toggle |
| 3 | Tests | TEST |  | 1 | 2 | Done | 22-06-2026 | 22-06-2026 | Add an `Almirah.Code` e2e asserting the tracking lanes carry both layers from the authored dates and are hidden by default; rebuild `Almirah.Doc` and confirm no new broken links or parse errors |

# Out of Scope

- **Repositioning the computed bar on the authored dates.** The existing bar stays the scheduler's ideal; the committed window is shown beside it, not substituted for it.
- **An Owner column in the Effort table.** A row's owner is recovered through its Scope `Item`, so the logged layer needs no new Effort column.
- **Per-day effort intensity, partial fills, or a burn-up inside the bar.** The logged layer is a plain first-to-last span; the fever chart ([ADR-196](./adr-196-buffer-fever-chart.md)) remains the place for consumption-versus-progress.
- **Persisting the toggle state** across page loads or sharing it through the URL.
- **Any change to scheduling, the chain, or the buffer.** The tracking lanes only read authored dates already parsed.

# Consequences

## Positive

- The three timelines become directly comparable on one chart: the computed schedule, the committed window, and the real logged span, per row, without cross-reading the Scope and Effort tables.
- A missing `Target Date` or stale effort log reads as a visible overrun or gap rather than a silent omission, nudging authors to keep the dates honest.
- The default chart is untouched: tracking lanes are hidden until asked for, and the toolbar leaves room for later Gantt controls.

## Negative

- Showing the tracking lanes doubles the owner-lane count, so a project with many owners gets a taller chart when the toggle is on.
- Extending the axis to the latest actual date widens a block whose work overran or whose effort is logged late, so an outlier date stretches that block's columns.

## Neutral

- The grey committed layer is the *authored commitment*, not "actual" in the logged sense; only the blue layer is real work. The lane is labelled "(tracking)" rather than "(actual)" to carry both honestly.
- A row with neither Scope dates nor effort yields an empty tracking bar; harmless, and itself a signal that the row is untracked.
- A `Start Date` earlier than the group's anchor clamps to the block's first column, consistent with how the calendar axis already begins at the group start (ADR-211).

# Alternatives Considered

- **Reposition the existing bar on the Scope Start/Target dates.** Rejected: it would discard the computed schedule the rest of the Gantt (chain, buffer, today line) is built on, and conflate plan with commitment. A separate lane keeps all three legible.
- **A single combined bar with a plan/actual split fill.** Rejected: three timelines on one bar overload a single row; the two-layer tracking bar on its own lane keeps the committed and logged spans distinct and the schedule untouched.
- **Always-on tracking lanes.** Rejected as too heavy for the common read: most viewers want the schedule, so the actuals are opt-in behind the toggle, matching the toolbar-driven controls this strip will grow.
- **A separate plan-vs-actual page.** Rejected: the comparison is most useful in place, on the same axis as the schedule it is measured against, not on a second page the reader must reconcile by eye.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

This decision adds SRS-165.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview Gantt shall provide, behind a toggle hidden by default, a per-owner tracking lane that draws each work item's committed window (its Scope Start and Target dates) and its real logged span (its earliest to latest Effort date) on the same calendar axis as the computed schedule, extending the axis to keep an overrun visible. | >[SRS-165] |

# References

- [ADR-198](./adr-198-workitem-gantt-visualization.md) — the work-item Gantt these lanes extend
- [ADR-195](./adr-195-critical-chain-buffer.md) — the computed schedule and buffer the committed and logged spans are compared against
- [ADR-196](./adr-196-buffer-fever-chart.md) — the Effort log the logged layer spans
- [ADR-201](./adr-201-gantt-group-segmentation.md) — the per-group blocks each lane and axis are scoped to
- [ADR-210](./adr-210-scope-table-format.md) — the Scope table whose per-row Start/Target dates the committed layer reads
- [ADR-211](./adr-211-group-start-dates.md) — the group-anchored calendar axis the dates map onto
- [ADR-212](./adr-212-critical-chain-highlight.md) — the most recent additive Gantt render this one follows

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
