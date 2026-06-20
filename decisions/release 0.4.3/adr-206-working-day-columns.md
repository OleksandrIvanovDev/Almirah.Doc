---
title: "ADR-206: Compact Working-Day Gantt Columns"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-06-2026 | Proposed |
|   | 20-06-2026 | Analysis |
|   | 20-06-2026 | Accepted |
|   | 20-06-2026 | In-Progress |
| * | 20-06-2026 | Implemented |

# Context

The calendar Gantt of [[adr-205-calendar-gantt-view]] renders on **full calendar-day columns**: every day from the anchor — weekends and holidays included — gets a column, the non-working ones are shaded, and bars span across them. That faithfully shows the calendar, but it spends **two of every seven columns** (≈29% of the width) on weekends that carry no work. On any non-trivial plan the chart becomes wide and sparse, and horizontal scrolling hides the work.

The estimates and the schedule are already in **working days**, so the weekend columns add no planning information — only calendar realism. This record trades that realism for density: show only the days work can happen on, and keep just enough calendar cues (the weekly rhythm and the holidays that actually land on working days) to stay legible.

## Altitude note

As in [[adr-205-calendar-gantt-view]], this is a **projection-and-rendering change only**. The scheduler, resource levelling, critical chain, and buffer stay on the working-day axis; `WorkingCalendar#date_for` still counts the Nth working day by skipping weekends and holidays, so the **projected completion date is unchanged**. What changes is the Gantt's *column axis* — from calendar days to business days — and how bars are placed on it.

# Decision

Replace the calendar-day column axis of [[adr-205-calendar-gantt-view]] with a **compact business-day axis**: render a column only for each **working day** and for each **holiday that falls on a weekday**; **omit Saturdays and Sundays** entirely; **highlight Fridays** to preserve the five-day weekly rhythm; and keep weekday holidays **shaded with work-item bars drawn across them**, still counted as non-working in the projection. This supersedes the Model-B rendering of [[adr-205-calendar-gantt-view]]; the working-calendar math and the projected completion date are kept as-is.

## Business-day column axis

Extend `WorkingCalendar` with a business-day view (the working-day math is unchanged):

- `business_columns(working_day_count)` — the weekday dates from the anchor through the last working day, **excluding** Saturdays and Sundays but **including** weekday holidays.
- `business_index(working_day)` — the 0-based position of the Nth working day within those business columns.
- `friday?(date)` — true on Fridays, for the rhythm highlight.

Because weekends are excluded from the axis, the only non-working columns that remain are **weekday holidays**; a holiday that falls on a weekend simply never appears.

## Rendering

Re-render the group-segmented Gantt ([[adr-201-gantt-group-segmentation]]) on business-day columns:

- **Columns** come from `business_columns`: one per working day plus one per weekday holiday. A block's width is its working span (work days plus buffer) plus only the weekday holidays it crosses — never weekends.
- **Weekday-holiday columns** keep the shaded `gantt_nonworking` treatment from [[adr-205-calendar-gantt-view]]; bars are drawn across them and they remain non-working in the schedule (a bar covering a holiday does not count it as work).
- **Friday columns** are highlighted with a light marker (a right-edge rule and header tint) so the eye can group the five-day weeks that are no longer separated by weekend gaps.
- **Headers** keep the month band over the day-of-month row, now spanning the business-day columns.
- **Bars** (work items and the Buffer lane) span from the business-day column of their first working day to that of their last working day inclusive — so a three-working-day bar that previously stretched five columns across a weekend now occupies three, while one that crosses a weekday holiday still stretches across the shaded holiday column.

## Projected completion date

Unchanged from [[adr-205-calendar-gantt-view]]: the per-group projected completion date is still `date_for(projected_duration)`, a real calendar date counted across all non-working days. Only the on-screen column axis is compacted; the dates the plan resolves to do not move.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-205] | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | This decision record: the business-day axis superseding the calendar-day columns of [[adr-205-calendar-gantt-view]]; omitting weekends; keeping weekday holidays shaded with bars across; the Friday rhythm highlight; the unchanged projection and projected completion date |
| 2 | Requirements | BA | >[ADR-205] | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | Revise SRS-152 and SRS-154 (calendar columns -> business-day columns omitting weekends; bars span business-day columns across weekday holidays) and add SRS-158 (weekday holidays shaded with bars drawn across; weekends not rendered) and SRS-159 (Friday columns highlighted for the weekly rhythm) in the `srs.md` "Planning" chapter |
| 3 | Code | DEV | >[ADR-205] | 3 | 5 | Done | 20-06-2026 | 20-06-2026 | Add `business_columns`, `business_index`, and `friday?` to `WorkingCalendar`; in `decisions_overview.rb` build each Gantt block from `business_columns` (drop weekend columns), shade only weekday-holiday columns, add the Friday highlight overlay/class, and place work-item and buffer bars by `business_index`; add `gantt_friday` styles to `templates/css/main.css`; leave `date_for` and the projected completion date untouched |
| 4 | Tests | TEST | >[ADR-205] | 2 | 3 | Done | 20-06-2026 | 20-06-2026 | Unit tests for `WorkingCalendar`: `business_columns` excludes weekends and includes weekday holidays; `business_index` counts business days; `friday?`. E2E under `spec/e2e/decisions_spec.rb`: the Gantt renders no weekend columns; a bar spanning a weekend occupies only its working-day columns; a weekday holiday renders a shaded column the bar spans; Friday columns carry the highlight; the projected completion date is unchanged from ADR-205 |

# Out of Scope

- **A toggle between the compact and full-calendar views.** This record replaces the calendar-day axis rather than offering both; a configurable switch is a later refinement if a calendar-faithful view is wanted back.
- **Per-resource calendars, working hours / half-days, and a configurable work week.** Unchanged from [[adr-205-calendar-gantt-view]]: Saturday and Sunday remain the fixed weekend.
- **Changing the scheduling, chain, buffer, or the projected completion date.** This record only compacts the column axis.
- **Per-group start dates.** Still a single shared anchor, as in [[adr-205-calendar-gantt-view]].

# Consequences

## Positive

- Removes the weekend whitespace, making the Gantt substantially denser — more of a plan fits without horizontal scrolling, since ~29% of the former columns are gone.
- Keeps the information that matters: weekday holidays are still visible, shaded, spanned, and honoured in the projection, and the Friday highlight preserves the weekly rhythm that the weekend gaps used to provide.
- No change to the planning math or the projected completion dates, so the compaction is purely visual and independently testable.

## Negative

- Supersedes the just-shipped Model-B rendering of [[adr-205-calendar-gantt-view]]; its weekend-column e2e expectations change (no weekend columns; a weekend-crossing bar now occupies only its working-day columns), and those tests are rewritten to assert the compact axis and the weekday-holiday case instead.
- The calendar is no longer literally faithful: the horizontal gap between a Friday and the following Monday is one column, so absolute elapsed time is read from the dates, not from column distance.

## Neutral

- Records with no estimates still render no bars; the calendar header renders only when the Gantt does.
- Re-rendering is required; nothing is retroactive.

# Alternatives Considered

- **Keep the full calendar-day axis ([[adr-205-calendar-gantt-view]] Model B).** Rejected here: the weekend columns are the very whitespace this record removes.
- **Offer both views behind a configuration toggle.** Deferred: it doubles the rendering paths and their tests for a calendar-faithful view that is not currently wanted; the compact axis is made the single default instead.
- **Collapse weekends into a thin spacer column rather than removing them.** Rejected: still spends width on non-working time and complicates bar spans, for little gain over the Friday-rhythm highlight.
- **Drop the Friday highlight.** Rejected: without the weekend gaps the five-day rhythm is otherwise invisible, making it hard to read week boundaries.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview Gantt shall render a day column for each working day and for each holiday that falls on a weekday, and shall not render columns for Saturdays and Sundays. | >[SRS-152] |
| 2 | The software shall span each work-item bar and each buffer bar across the business-day columns from its first to its last working day inclusive, covering any weekday-holiday columns it crosses without counting them as work. | >[SRS-154] |
| 3 | The Decision Records Overview Gantt shall mark each weekday-holiday column as non-working, shading it distinctly and drawing work-item bars across it. | >[SRS-158] |
| 4 | The Decision Records Overview Gantt shall highlight Friday columns to indicate the boundaries of the five-day working week. | >[SRS-159] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the planning/flow roadmap whose calendar/date-placement step this record refines
- [[adr-205-calendar-gantt-view]] — the calendar Gantt whose calendar-day column axis this record supersedes with a compact business-day axis; this record's own prerequisite
- [[adr-201-gantt-group-segmentation]] — the group-segmented Gantt re-rendered on the business-day axis
- [[adr-198-workitem-gantt-visualization]] — the original work-item swimlane Gantt
- [[adr-195-critical-chain-buffer]] — the working-day chain, buffer, and projected duration the projection rests on, unchanged here
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
