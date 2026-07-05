---
title: "ADR-205: Calendar Gantt View and Working-Day Calendar"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-06-2026 | Proposed |
|   | 20-06-2026 | Analysis |
|   | 20-06-2026 | Accepted |
|   | 20-06-2026 | In-Progress |
|   | 20-06-2026 | Implemented |
|   | 22-06-2026 | Reopened |
|   | 22-06-2026 | In-Progress |
| * | 22-06-2026 | Implemented |

# Context

The overview work-item Gantt ([[adr-198-workitem-gantt-visualization]]), its per-group segmentation and Buffer lane ([[adr-201-gantt-group-segmentation]]), and the critical chain and project buffer ([[adr-195-critical-chain-buffer]]) all run on an **abstract, 1-based working-day axis**: the scheduler returns integer `start_days`, a `day_count`, a `makespan`, and a critical chain whose lengths and buffer are working-day counts. There is no calendar — `Est (focused)` / `Est (safe)` are working days, and the Gantt header numbers columns `1, 2, 3, …` per block.

This was deliberate: [[adr-195-critical-chain-buffer]] and [[adr-196-buffer-fever-chart]] both placed "calendars, working-hours, holidays, and absolute date placement" out of scope, deferring a calendar/Gantt date view. This record lifts that deferral: it adds a **calendar view** to the Gantt that names the month, numbers the day of the month, and shows weekends (and holidays) as **non-working days**, and it surfaces a **projected completion date** on the Critical Chain page.

## Altitude note

The decisive design choice is that **the calendar is a presentation projection over the existing working-day schedule, not a change to scheduling.** Estimates are working days, so weekends are *already* excluded from the chain — there is nothing to remove from the resource-levelling, the chain trace, or the buffer. The only new behaviour is mapping a working-day index to a real calendar date by skipping non-working days. The `WorkItemScheduler` / `ChainScheduler` / `CriticalChain` math is therefore **unchanged**; a new `WorkingCalendar` converts working-day positions to calendar dates and columns for rendering, and the chain's projected duration to a projected date. This keeps the change additive and the planning core stable.

# Decision

Add a **`WorkingCalendar`** that maps the working-day axis onto real dates, anchor it at a configurable project **start date**, treat **weekends and an optional holiday list** as non-working days, render the group-segmented Gantt as a **calendar** (month and day-of-month headers, shaded non-working columns, bars spanning calendar columns), and show a **projected completion date** per decision group on the Critical Chain page.

## Working calendar

Introduce `WorkingCalendar` (a pure, deterministic helper alongside `CriticalChain` / `FeverChart`), constructed with an anchor date and a set of holiday dates:

- `date_for(working_day)` — the calendar date of the Nth working day (1-based) counted from the anchor, skipping weekends (Saturday, Sunday) and holidays.
- `columns(working_day_count)` — the inclusive run of calendar dates from the anchor through the last working day, **including** the intervening non-working dates, paired with a working-day to column-index map so bars can be placed.
- `working?(date)` / `non_working?(date)` — Saturday, Sunday, and any holiday are non-working.

The anchor is a single project-wide start date: every decision group's calendar begins on the same date (each group is planned "from the start date"). Per-group start dates are a later refinement.

## Configuration

Extend the `planning:` key:

```yaml
planning:
  start_date: 22-06-2026
  holidays:
    - 25-12-2026
    - 01-01-2027
```

- `start_date` is the calendar anchor for working day 1, in DD-MM-YYYY form. Absent or unparseable falls back to the render date (today). Setting it explicitly makes builds reproducible; defaulting to today means the Gantt shifts day to day, as the velocity and fever charts already do.
- `holidays` is an optional list of DD-MM-YYYY dates treated as non-working, exactly like weekends. Absent or empty means weekends only.

## Calendar Gantt rendering

Render the group-segmented Gantt ([[adr-201-gantt-group-segmentation]]) on calendar-day columns instead of working-day columns:

- **Columns** are calendar days, including weekends and holidays, produced by `WorkingCalendar#columns` per block from the shared anchor. A block's calendar width is its working span (work days plus buffer) stretched across the non-working days it crosses.
- **Two header rows** replace the single numbered row: a **month band** (each month spanning its day columns, labelled with the month name) above a **day-of-month** row. The group band and the owner and Buffer lanes shift down one row accordingly.
- **Non-working columns** (weekends and holidays) carry a `gantt_nonworking` class and are shaded, so weekends are visible as columns rather than implied by a gap.
- **Bars** span from the calendar column of a work item's first working day to that of its last working day inclusive; the shaded non-working columns show through beneath the bar, so a three-working-day bar that starts on a Friday visibly covers the weekend without counting it. The Buffer lane's bars are projected the same way.

## Projected completion date

On the Critical Chain page ([[enh-202-critical-chain-page]]), add a **projected completion date** per decision group, computed as `WorkingCalendar#date_for(projected_duration)` — the working-day projected duration ([[adr-195-critical-chain-buffer]]) turned into a real date by skipping non-working days. The working-day figures (chain length, buffer, projected duration) remain shown; the date is an addition, not a replacement.

## Amendment — 22-06-2026: holiday highlight colour

Reopened to recolour the non-working highlight. The shaded `gantt_nonworking` column and header were a neutral grey. After [[adr-206-working-day-columns]] dropped weekends from the axis, the only non-working columns left are **weekday holidays**, and grey made them too easy to miss. Recolour them to the **Chart.js palette red** already used across the planning views (`rgb(255, 99, 132)`, the blocked-bar pulse and WIP-limit colour), and give the holiday column the **same diagonal stripe texture as the project-buffer bar** (a 45° `repeating-linear-gradient`) so a non-working span reads as a hatched red column; the day-of-month header keeps a solid red tint so its number stays legible. Both are light tints, so work-item bars still read over the column. This is a CSS-only change in `main.css` — the calendar projection, the columns chosen, and SRS-152 ("mark each non-working column distinctly") are all unchanged; only the distinguishing fill differs.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-201] | 2 | 4 | Done | 20-06-2026 | 20-06-2026 | This decision record: the calendar-as-projection model; the `WorkingCalendar` API; the start-date and holidays config; the calendar-column Gantt with month/day headers and shaded non-working columns; the projected completion date |
| 2 | Requirements | BA | >[ADR-201] | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | New SRS items SRS-149 through SRS-155 in the `srs.md` "Planning" chapter: the `planning.start_date` anchor (DD-MM-YYYY, default today) and the `planning.holidays` list; the working-day-to-calendar-date mapping skipping weekends and holidays; the calendar-day Gantt columns including non-working days; the month-name and day-of-month header rows; the shaded non-working columns; the calendar span of work-item and buffer bars; and the per-group projected completion date on the Critical Chain page |
| 3 | Code | DEV | >[ADR-201] | 3 | 6 | Done | 20-06-2026 | 20-06-2026 | Add `WorkingCalendar` (`date_for`, `columns`, `working?`/`non_working?`) under `lib/almirah/project`; read `planning.start_date` and `planning.holidays` in `project_configuration.rb` (default anchor today, holidays empty); in `decisions_overview.rb` build each Gantt block's columns from `WorkingCalendar`, emit the month band and day-of-month header rows, shade `gantt_nonworking` columns, and place work-item and buffer bars on calendar columns; add the projected completion date to `critical_chain_page.rb`; add the calendar styles to `templates/css/main.css` |
| 4 | Tests | TEST | >[ADR-201] | 2 | 4 | Done | 20-06-2026 | 20-06-2026 | Unit tests for `WorkingCalendar`: `date_for` skips weekends and holidays; `columns` includes non-working days and maps working days to columns; the anchor defaults to today and honours config. E2E under `spec/e2e/decisions_spec.rb`: the Gantt renders month and day-of-month headers; weekend and holiday columns carry `gantt_nonworking`; a bar starting before a weekend spans the shaded weekend columns; the Critical Chain page shows a projected completion date consistent with the anchor and the projected duration; output is reproducible when `start_date` is set |
| 5 | Code | DEV |  | 1 | 1 | Done | 22-06-2026 | 22-06-2026 | Amendment: recolour the `gantt_nonworking` header to the Chart.js red (`rgb(255, 99, 132)`) tint and fill the holiday column with the buffer bar's 45° `repeating-linear-gradient` stripe in red, in `templates/css/main.css` |

# Out of Scope

- **Per-resource calendars.** A single project calendar applies to every owner; individual availability is a later refinement.
- **Working hours / half-days.** Durations remain whole working days; sub-day scheduling is out of scope.
- **A configurable work week.** Saturday and Sunday are the fixed weekend; alternative work weeks are deferred.
- **Per-group start dates.** All groups share one anchor; staggering group starts is a later refinement.
- **Changing the scheduling, resource levelling, chain, or buffer math.** These stay on the working-day axis; this record only projects them onto a calendar.
- **Re-deriving estimates as calendar days.** Estimates remain working days, consistent with [[adr-195-critical-chain-buffer]].

# Consequences

## Positive

- Turns the abstract working-day Gantt into a recognisable calendar with month and day labels and visible weekends, and gives each group a real projected completion date — the absolute-date placement deferred by [[adr-195-critical-chain-buffer]].
- Keeps the planning core untouched: because estimates are working days, the schedule, chain, and buffer need no change; the calendar is an additive projection that is independently testable.
- The holiday list lets a team reflect real non-working days without per-task date editing.

## Negative

- The Gantt grid is the largest change: an extra header row and a working-day-to-calendar-column remap, with bars spanning non-working columns. The existing Gantt e2e tests ([[adr-198-workitem-gantt-visualization]], [[adr-201-gantt-group-segmentation]]) assert numeric day headers and grid-column maths and will need updating.
- Defaulting the anchor to the render date makes the Gantt shift daily; reproducible output requires setting `planning.start_date`.

## Neutral

- Records with no estimates still render no bars; the calendar header renders only when the Gantt does.
- Re-rendering is required to compute and show the calendar; nothing is retroactive.

# Alternatives Considered

- **Relabel the existing working-day columns with dates (no weekend columns).** Lighter, but weekends would show only as a gap from Friday to Monday rather than as columns, which does not satisfy the request to highlight weekends. Rejected in favour of true calendar columns.
- **Move scheduling onto a calendar axis (schedule in calendar days).** Rejected: it would entangle weekends and holidays with the resource-levelling and buffer math that [[adr-195-critical-chain-buffer]] keeps in working days, for no benefit — the projection achieves the same view without disturbing the core.
- **Per-resource calendars now.** Rejected as premature; a single project calendar covers the common case and leaves per-resource availability as a clean later step.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall read an optional planning start date from the project configuration in DD-MM-YYYY form, using it as the calendar anchor for the first working day and falling back to the render date when the value is absent or unparseable. | >[SRS-149] |
| 2 | The software shall read an optional planning holidays list from the project configuration, each a DD-MM-YYYY date, and shall treat those dates, together with Saturdays and Sundays, as non-working days. | >[SRS-150] |
| 3 | The software shall map each working-day index of the planning schedule to a calendar date counted from the start date, skipping non-working days, so that the Nth working day falls on the Nth working date on or after the anchor. | >[SRS-151] |
| 4 | The Decision Records Overview Gantt shall render its day columns as consecutive calendar days, including non-working days, and shall mark each non-working column distinctly from working columns. | >[SRS-152] |
| 5 | The Decision Records Overview Gantt shall render a calendar header naming the month spanning its day columns and numbering the day of the month for each column. | >[SRS-153] |
| 6 | The software shall span each work-item bar and each buffer bar across the calendar columns from its first to its last working day inclusive, so that bars cover any intervening non-working columns without counting them as work. | >[SRS-154] |
| 7 | The Critical Chain page shall render, per decision-record group with an estimated critical chain, a projected completion date obtained by counting the projected duration in working days from the start date across non-working days. | >[SRS-155] |

# References

- [gfa.md](./../../specifications/gfa/gfa.md) — the planning/flow roadmap whose deferred calendar/date-placement step this record implements
- [[adr-201-gantt-group-segmentation]] — the group-segmented Gantt this record re-renders on calendar columns; this record's own prerequisite
- [[adr-206-working-day-columns]] — narrowed the non-working columns to weekday holidays only, the columns the amendment recolours
- [[adr-198-workitem-gantt-visualization]] — the original work-item swimlane Gantt and its abstract day axis
- [[adr-195-critical-chain-buffer]] — the working-day critical chain, buffer, and projected duration this record projects onto dates; the source of the "calendars deferred" out-of-scope note
- [[adr-196-buffer-fever-chart]] — also deferred calendars; its recent-Fridays trail is already calendar-based and unaffected
- [[enh-202-critical-chain-page]] — the dedicated Critical Chain page the projected completion date is added to
- [[adr-193-owner-wip-heatmap]] — the `planning:` configuration key extended here
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
