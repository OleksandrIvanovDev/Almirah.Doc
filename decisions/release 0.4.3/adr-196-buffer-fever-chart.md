---
title: "ADR-196: Effort Logging and Buffer-Consumption Fever Chart"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 14-06-2026 | Proposed |
|   | 14-06-2026 | Analysis |
|   | 20-06-2026 | Accepted |
|   | 20-06-2026 | In-Progress |
| * | 20-06-2026 | Implemented |

# Context

This is the fourth and final implementation step of the planning/flow roadmap in [gfa.md](./../../specifications/gfa/gfa.md). It follows [[adr-193-owner-wip-heatmap]], [[adr-194-full-kit-readiness]], and [[adr-195-critical-chain-buffer]], and implements the last Rule of Flow: **manage the project by buffer consumption.** Once safety is aggregated into a single project buffer (step 3), the only thing worth watching is whether that buffer is being eaten faster than the chain is being completed. That single signal — a *fever chart* — tells you when to act and, crucially, when to leave things alone, replacing the scramble over individual missed task dates the book argues against.

The prior steps produced everything the report needs except actuals:

- A per-group **critical chain of role-phase rows** and a **project buffer** ([[adr-195-critical-chain-buffer]]), with per-row focused estimates.
- The bounded per-row `Status` and the per-row `Owner` ([[adr-193-owner-wip-heatmap]]); and the decision that the open record lifecycle status is **not** a planning input ([[adr-172-current-status-marker]]).
- The `recent_fridays` bucketing behind the velocity chart ([[adr-182-overview-velocity-chart]]) and a `planning:` config key ([[adr-193-owner-wip-heatmap]], extended in step 3).

What is missing is **actual effort**: the Scope table records what work exists and how long each row was estimated to take, but not how much has been spent. This ADR adds a dated, append-only effort log — modelled on the Status table, the pattern the project already trusts for time-series — and derives buffer consumption and chain completion from it.

A note on the time dimension: the record lifecycle Status table has *dated* transition rows, which is why `effective_status_on` can reconstruct it historically — but the per-row Scope `Status` is a single current value with **no dated history**. So the only dated, replayable progress signal at the row level is the effort log itself. This ADR therefore derives the fever chart from logged effort (and, for the live point, the current row `Status`), and never from the lifecycle status — which also keeps it consistent with the project decision that the lifecycle vocabulary is not a planning input.

# Decision

Add an append-only **`# Effort`** table to decision records, derive per-row actual effort, compute **critical-chain completion** and **buffer consumption** per decision group from it, and render a **fever chart** with a historical trail on the Critical Chain page ([[enh-202-critical-chain-page]]), beside each group's chain block.

## Effort data source and extraction

Add an optional **`# Effort`** section (heading text `Effort`) containing an append-only table, located by heading text like the Status and Scope tables. The table columns are:

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 14-06-2026 | Code | DEV | 3 | parser scaffolding |

(Shown without its `# Effort` heading on purpose: Almirah's parser does not treat fenced code blocks as opaque, so a literal `# Effort` heading inside a fenced example would be parsed as a *second*, real Effort section and shadow the record's actual one — see the parser-limitation note in the Consequences.)

Columns are read by header text. `Date` is `DD-MM-YYYY` (parsed by the existing `collect_dates` machinery). `Hours` is non-negative. `Item` ties the entry to a Scope **row** (matched by item name, case-insensitive); `Owner` and `Note` are optional. Rows are never edited or deleted — corrections are new rows — so the log is diffable and replays cleanly in git, like the Status table.

Derive on each `Decision`:

- `actual_hours` = sum of `Hours` (record total, for display);
- `actual_hours_on(date)` = sum of `Hours` for entries dated on or before `date`;
- per row, `row_actual_hours_on(item, date)` = sum of `Hours` for entries whose `Item` matches that row and are dated on or before `date`.

Hours convert to working days via `hours_per_day` (config below). Effort entries with no `Item` count toward `actual_hours` but credit no specific chain row.

## Critical-chain completion and buffer consumption (per decision group)

For each **decision group** (the folder grouping from [[adr-197-decision-group-collection]] — the first-level sub-folder under `decisions/`), over its critical-chain **rows** (from [[adr-195-critical-chain-buffer]]) with a positive focused estimate, as of a date `d`:

- **Chain completion %** = `100 × Σ_chainrows(credit(row, d) × focused(row)) / Σ_chainrows(focused(row))`, where `credit(row, d)` is the row's fractional progress. For the **live point** (`d` = render date), a row whose `Status` is `Done` credits `1`; otherwise `credit = clamp(row_actual_days_on(d) / focused, 0, 1)`. For **historical** trail points, only logged effort is available, so `credit = clamp(row_actual_days_on(d) / focused, 0, 1)`. This is unit-free in aggregate and reads only bounded per-row `Status` and the effort log — never the lifecycle status.
- **Buffer consumption %** = `100 × consumed_days / project_buffer`, where `consumed_days = Σ_chainrows max(0, row_actual_days_on(d) − focused(row))` — the aggregate amount by which chain rows overran their aggressive (focused) estimate. Consumption may exceed 100% (over buffer).

These two numbers form the point `(completion%, consumption%)` — the fever-chart coordinate.

## Fever chart rendering

Render the **fever chart on the Critical Chain page** (`build/decisions/critical-chain.html`, [[enh-202-critical-chain-page]]) — the page that already renders each group's chain, buffer, and projected duration — one chart per decision group that has an estimated critical chain. Within each group's `critical_chain_block`, lay the existing chain table, buffer, and projected-duration text on the **left** and that group's fever chart on the **right**, side by side; a group with no estimates keeps its "not sized" note and renders no chart.

- x-axis = chain completion %, y-axis = buffer consumption %, both `0..100+`.
- Three zones by the conventional diagonal bands: **green** (consumption comfortably below completion — on track, do nothing), **yellow** (tracking near — watch), **red** (consumption ahead of completion — act). Default boundaries are the standard one-third / two-thirds diagonals; configurability is out of scope.
- A **trail** of points over recent Fridays (reusing `recent_fridays`), each computed from `row_actual_hours_on(friday)` on both axes, so the chart shows the trajectory into the current zone, not just today's point.

## Configuration

Extend `planning:` with the working-day conversion used to compare logged hours against day-denominated estimates:

```yaml
planning:
  wip_limit: 2
  buffer_ratio: 0.5
  hours_per_day: 8
```

`hours_per_day` is optional; absent or non-positive falls back to `8`.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-195] | 1 | 2 | Done | 14-06-2026 | 20-06-2026 | This decision record: the append-only `# Effort` table, per-row actuals, and the effort-based fever-chart definition |
| 2 | Requirements | BA | >[ADR-195] | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | New SRS items SRS-128 through SRS-135 in `srs.md` "Planning" chapter: the `# Effort` append-only table (header-text columns, `DD-MM-YYYY` dates, non-negative Hours, `Item` tying an entry to a Scope row); `actual_hours`, `actual_hours_on`, and per-row `row_actual_hours_on`; the `hours_per_day` conversion and config default; per-group chain-completion % (effort-based credit, with row `Status` `Done` crediting the live point); per-group buffer-consumption % (aggregate focused overrun over the buffer); and the fever-chart point, three zones, and recent-Fridays trail |
| 3 | Code | DEV | >[ADR-195] | 3 | 6 | Done | 20-06-2026 | 20-06-2026 | Add `actual_hours` / `actual_hours_on` / per-row `row_actual_hours_on` on `Decision` reusing `find_section_table` / `collect_dates`; read `planning.hours_per_day` in `project_configuration.rb` (default 8); compute per-group completion% and consumption% over the chain rows (`CriticalChain#chain`) as of a date; in `critical_chain_page.rb` extend `critical_chain_block` to wrap the existing chain table/buffer/projected text and the new fever chart in a side-by-side layout (a `.cc_group_body` flex wrapper with `.cc_plan` / `.cc_fever` columns added to `templates/css/main.css`), emitting the chart only in the `plan.estimated?` branch with Chart.js zones and a recent-Fridays trail; load Chart.js on the page by extending the `{{STYLES_AND_SCRIPTS}}` branch in `base_document.rb` to include `CriticalChainPage` |
| 4 | Tests | TEST | >[ADR-195] | 2 | 4 | Done | 20-06-2026 | 20-06-2026 | E2E tests under `spec/e2e/decisions_spec.rb`: effort sums per record and per row, and `row_actual_hours_on` respects the as-of date and `Item` match; `hours_per_day` default and override; completion% is the focused-weighted effort credit over chain rows, with a `Done` row crediting the live point; consumption% is the aggregate focused overrun over the buffer and may exceed 100%; a row under its focused estimate contributes zero consumption; the fever chart renders on `critical-chain.html` beside its group's chain table and its point lands in the expected zone; the trail has one point per recent Friday from as-of-date effort; a decision group with no estimated chain keeps its "not sized" note and renders no fever chart; nothing reads the lifecycle status |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
|  |  |  |  |  |

# Out of Scope

- **Using the record lifecycle status in the fever chart.** Completion reads per-row `Status` and the effort log; the open lifecycle vocabulary is never consulted, per [[adr-193-owner-wip-heatmap]] / [[adr-172-current-status-marker]].
- **Feeding-buffer consumption.** Only the single project buffer is tracked; feeding buffers remain out of scope (as in [[adr-195-critical-chain-buffer]]).
- **Configurable fever-chart zone boundaries.** The conventional one-third / two-thirds diagonals are fixed here.
- **Dated history of per-row `Status`.** The Scope `Status` is a current value; historical completion is reconstructed from the dated effort log, not from past row states.
- **Calendar/working-hours/holiday modelling and absolute target dates.** Consumption is derived from logged effort versus estimates, not from elapsed calendar time against baseline dates.
- **Automatic effort capture** (timers, git-commit inference, IDE integration). Effort is hand-logged, keeping the file the single source of truth.
- **Editing/deleting effort rows.** The log is append-only; corrections are new rows.

# Consequences

## Positive

- Completes the roadmap: the static plan from step 3 becomes a live, single-signal health report — the buffer-management discipline the book ends on.
- Reuses the project's most trusted patterns: an append-only dated table (like Status) and the `recent_fridays` bucketing (like the velocity chart), so the historical trail comes almost for free.
- Effort-based completion gives **partial credit** per row, so the trail glides as analysis, code, and tests finish, rather than stepping only when a whole record flips — and it is reconstructable historically, which a per-row `Status` (no dated history) is not.
- Reads only bounded per-row `Status` and the effort log, fixing the earlier draft's reliance on the lifecycle status and keeping the chart consistent with the project's two-status decision.
- Gives the team the "when *not* to act" signal (green zone) that scattered task dates never provided.

## Negative

- Requires discipline to log effort; an unmaintained `# Effort` table makes both completion and consumption read low. A row marked `Done` with little logged effort under-reports completion on the historical trail (the live point credits it via `Status`).
- Adds header-text dependencies (`# Effort`, `Hours`, `Date`, `Item`); the test suite locks these in, as for the columns in steps 1–3.
- Effort entries with no `Item` cannot be attributed to a chain row and so do not move the fever point, only the record total.

## Neutral

- Records authored before this ADR work unchanged: with no `# Effort` table, actuals are 0, consumption is 0, and the record does not move the fever point.
- Re-rendering is required to compute and show the fever chart; nothing is retroactive.
- The Critical Chain page now loads Chart.js (previously only the Decision Records Overview did); the `{{STYLES_AND_SCRIPTS}}` branch in `base_document.rb` gains `CriticalChainPage`.
- Implementation surfaced a pre-existing parser limitation: `doc_parser.rb` does not treat fenced code blocks as opaque, so a `# Effort` (or `# Scope` / `# Status`) heading shown inside a fenced example is parsed as a real section. Because a record's real `# Effort` section sits near the bottom, an example heading above it would shadow it via `find_section_table`. This record's example therefore omits the literal heading; a proper fix (making fences opaque) is a separate Almirah.Code decision, since it would change the rendering of every existing fenced example.
- This record's own `# Effort` table is present but empty (no effort logged yet). Its prerequisite [[adr-195-critical-chain-buffer]] is now `Implemented` (all rows `Done`), so this record is fully kitted and — being the only group member with estimated, not-`Done` rows — currently *is* the live critical chain for the `release 0.4.3` group; with no effort logged, its fever point sits at the origin (0% complete, 0% consumed).

# Alternatives Considered

- **Effort-based completion is now the chosen default (reversing the earlier draft).** In the per-row model it is the right choice: it gives partial credit and, unlike per-row `Status`, is reconstructable historically from the dated effort log. The live point still credits a `Done` row fully via its `Status`.
- **Record-status-based completion (`effective_status_on == Implemented`).** Rejected: it reads the open lifecycle vocabulary the project decided is not a planning input, and a per-row `Status` has no dated history to drive a trail.
- **Measure buffer consumption from elapsed calendar time against baseline dates.** Rejected: requires the calendar/baseline model step 3 deferred; logged effort versus the aggressive estimate needs only data we already have.
- **A burndown / burn-up chart instead of a fever chart.** Rejected: burn charts track scope/time, not buffer health — the specific thing CCPM manages.
- **Auto-capturing effort from git history.** Rejected: breaks the file-is-the-source-of-truth model and attributes commits, not focused effort.
- **A separate timesheet document type.** Deferred: a per-record append-only table keeps effort next to the work it describes and reuses existing table parsing.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | A Decision Record shall support an optional Effort section containing an append-only table whose columns are a date in DD-MM-YYYY form, an Item tying the entry to a Scope row, an optional owner, a non-negative number of hours, and an optional note. The columns shall be identified by header text, not by position. | >[SRS-128] |
| 2 | The software shall expose on each Decision Record the total actual effort, an as-of-date total, and a per-Scope-row as-of-date effort summing the Hours of entries whose Item matches the row and whose date is on or before the given date. | >[SRS-129] |
| 3 | The software shall read an optional planning hours-per-day value from the project configuration, applying a default of 8 when absent or non-positive, and shall use it to convert logged hours to working days. | >[SRS-130] |
| 4 | For each decision-record group, the software shall compute the critical-chain completion as the focused-estimate-weighted fractional progress of the group's critical-chain rows, where a row's progress is its as-of-date actual days over its focused estimate clamped to one, and a row whose Status is Done credits full progress at the current date. | >[SRS-131] |
| 5 | For each decision-record group, the software shall compute the buffer consumption as the percentage of the project buffer consumed, where consumed days equal the sum over the critical-chain rows of the amount by which each row's as-of-date actual days exceed its focused estimate, clamped to zero, and where the consumption may exceed one hundred percent. | >[SRS-132] |
| 6 | The Critical Chain page shall render, per decision-record group with an estimated critical chain, a fever chart plotting buffer consumption against critical-chain completion, placed beside that group's critical-chain table. | >[SRS-133] |
| 7 | The fever chart shall divide its area into green, yellow, and red zones by the conventional one-third and two-thirds diagonal boundaries, indicating respectively on-track, watch, and act conditions. | >[SRS-134] |
| 8 | The fever chart shall render a trail of points, one per recent Friday, each computed from the as-of-date actual effort of the critical-chain rows on both axes, showing the trajectory of the decision group toward its current zone. | >[SRS-135] |

# References

- [gfa.md](./../../specifications/gfa/gfa.md) — the roadmap this ADR is step 4 (the final step) of; implements the manage-by-buffer-consumption rule
- [[adr-195-critical-chain-buffer]] — supplies the critical chain of role-phase rows, the project buffer, and the per-row focused estimates this report tracks against; this record's own prerequisite
- [[adr-197-decision-group-collection]] — supplies the decision-group folder grouping the completion and buffer consumption are computed per (replacing the earlier per-target-release grouping)
- [[enh-202-critical-chain-page]] — the dedicated Critical Chain page that already renders each group's chain, buffer, and projected duration; this ADR adds the fever chart beside that per-group block (the rendering moved here from the overview)
- [[adr-182-overview-velocity-chart]] — the `recent_fridays` bucketing reused for the fever-chart trail
- [[adr-193-owner-wip-heatmap]] — the per-row `Status` the live completion point reads, and the `planning:` config key extended here
- [[adr-172-current-status-marker]] — the open-vocabulary lifecycle status that this chart deliberately does **not** consult
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
