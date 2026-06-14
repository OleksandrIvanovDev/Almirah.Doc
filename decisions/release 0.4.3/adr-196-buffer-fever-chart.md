---
title: "ADR-196: Effort Logging and Buffer-Consumption Fever Chart"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 14-06-2026 | Proposed |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

This is the fourth and final implementation step of the planning/flow roadmap in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md). It follows [[adr-193-owner-wip-heatmap]], [[adr-194-full-kit-readiness]], and [[adr-195-critical-chain-buffer]], and implements the last Rule of Flow: **manage the project by buffer consumption.** Once safety is aggregated into a single project buffer (step 3), the only thing worth watching is whether that buffer is being eaten faster than the chain is being completed. That single signal — rendered as a *fever chart* — tells you when to act and, crucially, when to leave things alone, replacing the scramble over individual missed task dates that the book argues against.

The prior steps already produced everything the report needs except actuals:

- A per-release **critical chain** and **project buffer** ([[adr-195-critical-chain-buffer]]), with per-record `focused_estimate` / `safe_estimate`.
- A per-record **`current_status`** and the `effective_status_on(date)` time-machine ([decision.rb](./../../../Almirah.Code/lib/almirah/doc_types/decision.rb)) the overview already uses to reconstruct state on any past date for its velocity chart ([[adr-182-overview-velocity-chart]]).
- A `planning:` config key ([[adr-193-owner-wip-heatmap]], extended in step 3).

What is missing is **actual effort**. The Scope table records what work *exists* and how long it was *estimated* to take, but not how much has actually been spent. This ADR adds a dated, append-only effort log — modelled on the Status table, the pattern the project already trusts for time-series — and derives buffer consumption and chain completion from it.

# Decision

Add an append-only **`# Effort`** table to decision records, derive per-record actual effort, compute **buffer consumption** and **critical-chain completion** per release, and render a **fever chart** with a historical trail.

## Effort data source and extraction

Add an optional **`# Effort`** section to decision records containing an append-only table, located by heading text exactly like the Status and Scope tables:

```markdown
# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 14-06-2026 | Code | A.I. | 3 | parser scaffolding |
```

Columns are identified by header text. `Date` is `DD-MM-YYYY` (the project-wide format, parsed by the existing `collect_dates` machinery). `Hours` is a non-negative number. `Item` optionally ties a row to a Scope item; `Note` is free text. Rows are never edited or deleted — corrections are new rows — so the table is diffable and replays cleanly in git, exactly as the Status table does.

Derive on each `Decision`:

- `actual_hours` = sum of the `Hours` column;
- `actual_hours_on(date)` = sum of `Hours` for rows whose `Date` is on or before `date` (the effort-log counterpart of `effective_status_on`).

Actual days convert via `actual_hours / hours_per_day` (config below).

## Buffer consumption and chain completion (per release)

For each `Target Release Version`, over the records on that release's critical chain (from [[adr-195-critical-chain-buffer]]), as of a given date:

- **Chain completion %** = `100 × (Σ focused_estimate of chain records Implemented as of the date) / (Σ focused_estimate of all chain records)`, using `effective_status_on(date)` to decide "Implemented as of the date". This is status-based and unit-free.
- **Buffer consumption %** = `100 × (consumed_days / project_buffer)`, where `consumed_days = Σ_chain max(0, actual_days_on(date) − focused_estimate)`. In words: the buffer absorbs the aggregate amount by which chain records overran their aggressive (focused) estimates. Consumption may exceed 100% (over buffer).

These two numbers are a point `(completion%, consumption%)` — the fever-chart coordinate.

## Fever chart rendering

Add a **fever chart** to the Decision Records Overview, one per target release that has an estimated critical chain:

- x-axis = chain completion %, y-axis = buffer consumption %, both `0..100+`.
- Three zones by the conventional diagonal bands: **green** (consumption comfortably below completion — on track, do nothing), **yellow** (consumption tracking near completion — watch), **red** (consumption ahead of completion — act). Default zone boundaries are the standard one-third / two-thirds diagonals; making them configurable is out of scope.
- A **trail** of points over recent Fridays (reusing the `recent_fridays` bucketing behind the velocity chart) plotted from `effective_status_on` and `actual_hours_on`, so the chart shows the trajectory into the current zone, not just today's point.

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

| Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|
| Requirements | A.I. | >[ADR-195] | 1 | 2 | To Do |  |  | New SRS items SRS-128 through SRS-135 in `srs.md` "Planning" chapter: the `# Effort` append-only table (columns read by header text, `DD-MM-YYYY` dates, non-negative Hours); `actual_hours` and `actual_hours_on(date)` aggregates; the `hours_per_day` conversion and config default; per-release chain-completion % (status-based via `effective_status_on`); per-release buffer-consumption % (aggregate focused overrun over the buffer); the fever-chart point, three zones, and recent-Fridays trail |
| Code | A.I. | >[ADR-195] | 3 | 6 | To Do |  |  | Add `actual_hours` / `actual_hours_on` on `Decision` reusing `find_section_table` / `collect_dates`; read `planning.hours_per_day` in `project_configuration.rb` (default 8); compute per-release completion% and consumption% as of a date; render the fever chart with zones and a recent-Fridays trail in `decisions_overview.rb`, alongside the existing chart grid |
| Tests | A.I. | >[ADR-195] | 2 | 4 | To Do |  |  | End-to-end tests under `spec/e2e/decisions_spec.rb`: effort sums per record and `actual_hours_on` respects the as-of date; `hours_per_day` default and override; completion% is the focused-weighted fraction of Implemented chain records and tracks `effective_status_on` over time; consumption% is the aggregate focused overrun over the buffer and may exceed 100%; records under their focused estimate contribute zero consumption; the fever point lands in the expected zone; the trail has one point per recent Friday; a release with no estimated chain renders no fever chart |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
|  |  |  |  |  |

# Out of Scope

- **Per-Scope-item completion and per-item effort roll-up into feeding buffers.** Completion and consumption are computed at the record level over the record-level chain, consistent with steps 1–3; feeding buffers remain out of scope (as in [[adr-195-critical-chain-buffer]]).
- **Configurable fever-chart zone boundaries.** The conventional one-third / two-thirds diagonals are fixed here.
- **Partial-credit completion of in-progress chain records.** Completion is status-based (Implemented or not); crediting a fraction of an In-Progress record's focused duration is deferred to avoid coupling the x-axis to effort units.
- **Calendar/working-hours/holiday modelling and absolute target dates.** Consumption is derived from logged effort versus estimates, not from elapsed calendar time against baseline dates.
- **Automatic effort capture** (timers, git-commit inference, IDE integration). Effort is hand-logged in the `# Effort` table, keeping the file the single source of truth.
- **Editing/deleting effort rows.** The log is append-only; corrections are new rows.

# Consequences

## Positive

- Completes the roadmap: the static plan from step 3 becomes a live, single-signal health report — the buffer-management discipline the book ends on.
- Reuses the project's most trusted patterns: an append-only dated table (like Status) and the `effective_status_on` time-machine (like the velocity chart), so the historical fever trail comes almost for free.
- Single source of truth preserved: actuals live in the `# Effort` table; completion and consumption are derived, never hand-maintained, consistent with [[adr-191-overview-target-date]].
- Gives the team the "when *not* to act" signal (green zone) that scattered task dates never provided.

## Negative

- Requires discipline to log effort; an unmaintained `# Effort` table makes consumption read artificially low (the inverse failure mode of stale `current_status` noted in earlier steps).
- Adds another header-text dependency (`# Effort`, `Hours`, `Date`); the test suite locks these in, as for the columns introduced in steps 1–3.
- Status-based completion means a long In-Progress chain record shows no x-axis progress until it flips to Implemented, which can make the trail step rather than glide.

## Neutral

- Records authored before this ADR work unchanged: with no `# Effort` table, `actual_hours` is 0, consumption is 0, and the record simply does not move the fever point.
- Re-rendering is required to compute and show the fever chart; nothing is retroactive.
- This record's own `# Effort` table is present but empty (no effort logged yet) and it depends on [[adr-195-critical-chain-buffer]] (still `Proposed`), so it remains not-fully-kitted — closing the live demonstration that ran across steps 2–4.

# Alternatives Considered

- **Measure buffer consumption from elapsed calendar time against baseline dates.** Rejected for this step: it requires a calendar/working-day model and baseline date placement that step 3 explicitly deferred. Deriving consumption from logged effort versus the aggressive estimate needs only data we already have.
- **Effort-based (rather than status-based) chain completion.** Rejected as the default: it couples the completion axis to the hours/days conversion and to honest in-flight logging; status-based completion is robust and unit-free. Partial credit is left as a future refinement.
- **A burndown or burn-up chart instead of a fever chart.** Rejected: burn charts track scope/time but not buffer health, which is the specific thing CCPM manages; the fever chart is the report the rule calls for.
- **Auto-capturing effort from git history.** Rejected: breaks the file-is-the-source-of-truth model and attributes commits, not focused effort; hand-logging keeps the data deliberate and auditable.
- **A separate timesheet document type.** Deferred: a per-record append-only table keeps effort next to the work it describes and reuses existing table parsing; a dedicated type is heavier than warranted now.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | A Decision Record shall support an optional Effort section containing an append-only table whose columns are a date in DD-MM-YYYY form, an optional Scope item, an optional owner, a non-negative number of hours, and an optional note. The columns shall be identified by header text, not by position. | >[SRS-128] |
| 2 | The software shall expose an actual-effort attribute on each Decision Record equal to the sum of the Hours column of its Effort table, and an as-of-date variant equal to the sum of Hours for rows whose date is on or before a given date. | >[SRS-129] |
| 3 | The software shall read an optional planning hours-per-day value from the project configuration, applying a default of 8 when the value is absent or non-positive, and shall use it to convert logged hours to working days. | >[SRS-130] |
| 4 | For each Target Release Version, the software shall compute the critical-chain completion as the percentage, weighted by focused estimate, of the release's critical-chain records whose effective status as of a given date is Implemented. | >[SRS-131] |
| 5 | For each Target Release Version, the software shall compute the buffer consumption as the percentage of the project buffer consumed, where consumed days equal the sum over the critical-chain records of the amount by which each record's actual days as of a given date exceed its focused estimate, clamped to zero, and where the consumption may exceed one hundred percent. | >[SRS-132] |
| 6 | The Decision Records Overview page shall render, per Target Release Version with an estimated critical chain, a fever chart plotting buffer consumption against critical-chain completion. | >[SRS-133] |
| 7 | The fever chart shall divide its area into green, yellow, and red zones by the conventional one-third and two-thirds diagonal boundaries, indicating respectively on-track, watch, and act conditions. | >[SRS-134] |
| 8 | The fever chart shall render a trail of points, one per recent Friday, computed from the effective status and the as-of-date actual effort at each of those dates, showing the trajectory of the release toward its current zone. | >[SRS-135] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 4 (the final step) of; implements the manage-by-buffer-consumption rule
- [[adr-195-critical-chain-buffer]] — supplies the critical chain, project buffer, and estimates this report tracks against; this record's own prerequisite
- [[adr-182-overview-velocity-chart]] — the `effective_status_on` time-machine and `recent_fridays` bucketing reused for the fever-chart trail
- [[adr-193-owner-wip-heatmap]] — introduced the `planning:` config key extended here
- [[adr-172-current-status-marker]] — the current-status marker the completion axis is keyed on
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
