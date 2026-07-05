---
title: "ENH-209: Buffer Utilization Visibility in Planning Views"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-06-2026 | Proposed |
|   | 20-06-2026 | Accepted |
|   | 20-06-2026 | In-Progress |
| * | 21-06-2026 | Implemented |

# Context

The buffer-consumption machinery from [ADR-196](./../adr-196-buffer-fever-chart.md) (refined by [issue-207](./../issues/issue-207-fever-completed-overrun.md)) computes, per decision-record group, how far the critical chain's overruns have eaten into the project buffer (SRS-132). The fever chart plots that consumption against chain completion (SRS-133), and the calendar projection from [ADR-205](./../adr-205-calendar-gantt-view.md) anchors the work-item Gantt ([ADR-198](./../adr-198-workitem-gantt-visualization.md)) on real dates with a Buffer lane (SRS-144).

What the rendered views lacked was a direct, legible read of *buffer utilization* — the single number a reader most wants from a critical-chain plan. Three gaps surfaced while dogfooding the planning views on this repo:

- The Critical Chain page reported the project buffer only in working days. It never said how much of that buffer is already consumed, so "how much margin is left?" could not be answered from the page.
- The fever chart's trail points showed *where* the group has been but not *when* each reading was taken, so the trajectory could not be tied to a date.
- The Gantt drew the Buffer lane at the end of a group's work with nothing relating it to the current date, so a reader could not see whether the project had reached, or already eaten into, its buffer.

This enhancement closes all three with presentation-only additions over the existing computations. It changes no consumption maths, schedule, chain, or buffer sizing; it makes the already-computed utilization visible across the Critical Chain page and the Gantt.

# Decision

Add three buffer-utilization readouts, scoped to `critical_chain_page.rb`, `fever_chart.rb`, and `decisions_overview.rb` plus the planning CSS.

**1. Buffer-consumed line on the Critical Chain page.** Beneath each group's "Project buffer: N working days" figure, render a line stating the consumption as a percentage and as days of the baseline buffer:

```
Project buffer: 3 working days
Buffer consumed: 75% (3 of 4 baseline days)
```

The percentage and the day count both come from the fever chart's live point, so the figure and the chart agree exactly. The denominator is made explicit as the *baseline* buffer (the original plan-time buffer, completed rows included, per issue-207), which is intentionally not the same quantity as the remaining-work "Project buffer" figure above it — stating "of 4 baseline days" prevents reading the 75% as a fraction of the 3-day line. `FeverChart` gains a public `consumed_days` so the page can report the numerator without recomputing it. The line is omitted when the group has no baseline buffer to consume.

**2. Dated fever-chart points.** Each fever point carries the calendar date it was sampled on — a recent Friday for each trail point, the render date for the live point — and the point tooltip presents that date followed by the completion and consumption values (for example `Jun 20: (57.14, 75)`), in place of the bare dataset label. This ties each reading on the trajectory to a date.

**3. Today marker on the Gantt.** A thin full-height vertical rule per group block marks the current date, so buffer utilization can be read against the Buffer lane on the timeline:

- Because each block carries its own day-index axis beginning at one (SRS-141), the rule is emitted per block at the column of the first working day on or after today, projected through the block's business-day calendar (SRS-151 / SRS-152); a weekend or holiday today therefore snaps forward to the next working column.
- When today precedes a block it pins to the block's leading edge; when today is past the block — the project has overrun its projected end including buffer — it hugs the trailing edge of the last column, so "beyond the plan" still reads.
- The rule is drawn after the bars so it paints on top and is inert to the pointer, so it never intercepts a bar's dependency tooltip ([ENH-208](./enh-208-gantt-dependency-tooltip.md)).

```
            Jun 22
Owner   ... | work bars |
DEV     ...   ████████
Buffer  ...           ▒│▒▒     │ = today, one day into the buffer
```

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  | 2 | 4 | Done | 20-06-2026 | 21-06-2026 | In `critical_chain_page.rb`, add a `cc_consumed` line under the buffer figure showing the live consumption percentage and the consumed-of-baseline working days; expose `consumed_days` on `FeverChart`; in `decisions_overview.rb` emit a per-block today rule (`gantt_today_lines` / `gantt_today_column`) with left/right edge clamping; date each fever point and show the date in its tooltip in `critical_chain_page.rb`; add the `cc_consumed` and `gantt_today` CSS |
| 2 | Tests | TEST |  | 2 | 4 | Done | 20-06-2026 | 21-06-2026 | In `spec/e2e/decisions_spec.rb`, assert the buffer-consumed line text for a no-overrun and an overrun group, the per-point date label and tooltip callback, and the today rule clamping to the first column (future anchor) and to the last column's trailing edge (past anchor) |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 21-06-2026 | Code | DEV | 1 | dated fever points and tooltip in `critical_chain_page.rb` |
| 21-06-2026 | Code | DEV | 1 | `cc_consumed` line and `FeverChart#consumed_days` |
| 21-06-2026 | Code | DEV | 1 | `gantt_today_lines` / `gantt_today_column` and the `gantt_today` CSS |
| 21-06-2026 | Tests | TEST | 1 | buffer-consumed, dated-point, and today-rule assertions in `decisions_spec.rb` |

# Out of Scope

- **The consumption, completion, chain, and buffer maths.** Every figure is read from the existing ADR-196 / issue-207 computations (SRS-131, SRS-132); nothing about how they are calculated changes.
- **The schedule and calendar projection.** Bar placement, the Buffer lane, the projected completion date, and the calendar axis (SRS-151) are untouched; the today rule only reads the same calendar.
- **Showing buffer erosion inside the Gantt buffer bar** (a consumed/remaining fill of the bar itself). Considered and not adopted: it would force the bar to represent the baseline pool rather than the remaining-work slack it currently shows. The today marker gives the timeline reference without redefining the bar.
- **A date caption or hover panel on the today rule.** The marker is a plain click-through line; a "Today" label or interactive hit-area is deferred.

# Consequences

## Positive

- "How much buffer is left?" is answerable directly from the Critical Chain page, and "of N baseline days" makes the denominator unambiguous.
- The fever chart's trajectory is now tied to dates, so a reading can be placed in time.
- The Gantt relates the Buffer lane to the current date, showing at a glance whether a group has entered its buffer; the right-edge clamp keeps an overrunning project legible.
- All three are additive overlays — bars, lanes, tooltips, and the consumption maths are unchanged.

## Negative

- The consumed line and the today rule are computed against the render date, so a rendered page is a snapshot; re-rendering moves both. This matches the moving-anchor behaviour when no start date is configured (SRS-149).
- "Buffer consumed" is a percentage of the baseline buffer while "Project buffer" above it is the remaining-work buffer, so the two day counts differ by design; the "of N baseline days" wording is what keeps that from misreading.
- On a weekend or holiday the today rule snaps to the next working column rather than sitting between columns.

## Neutral

- Like [ENH-199](./enh-199-gantt-separator-lines.md) and [ENH-200](./enh-200-gantt-status-styling.md), these are presentation refinements of the ADR-196 / ADR-198 / ADR-205 planning views; unlike purely cosmetic ones the readouts are deterministic and assertable, so regression tests are in scope and new requirements (SRS-160, SRS-161, SRS-162) record the behaviours.
- Already-rendered projects must be re-rendered to show the readouts; nothing is retroactive.

# Alternatives Considered

- **Show buffer consumed as "P% remaining" only.** Rejected in favour of "P% (D of N baseline days)": the bare percentage invited reading it against the 3-day "Project buffer" line; the explicit baseline day count removes the ambiguity.
- **A single today rule across the whole grid.** Rejected: with per-block day-index axes (SRS-141) there is no one column for today across blocks, so the rule is per block.
- **Omit the today rule when today is outside a block's range.** Rejected for the past case: an overrun is exactly when the reference is most useful, so the rule clamps to the trailing edge instead of vanishing.
- **Fill the Gantt buffer bar to show erosion.** Deferred: it conflates the remaining-work buffer the bar draws with the baseline pool consumption is measured against; the dated marker and the page line cover utilization without that conflation.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-196](./../adr-196-buffer-fever-chart.md) — the buffer-consumption computation and fever chart these readouts surface
- [issue-207](./../issues/issue-207-fever-completed-overrun.md) — the baseline-buffer denominator the consumed line reports against
- [ADR-205](./../adr-205-calendar-gantt-view.md) — the calendar projection the today marker reuses to locate today
- [ADR-198](./../adr-198-workitem-gantt-visualization.md) — the work-item Gantt the marker overlays
- [ENH-208](./enh-208-gantt-dependency-tooltip.md) — the bar dependency tooltip the click-through rule must not obstruct

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
