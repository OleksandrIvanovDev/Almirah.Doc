---
title: "ADR-185: Current Status Distribution Chart on the Decision Records Overview"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 01-06-2026 | Proposed |
|   | 01-06-2026 | Accepted |
|   | 01-06-2026 | In-Progress |
| * | 04-06-2026 | Implemented |

# Context

The Decision Records Overview page reserves three chart cells inside `decisions_overview_charts`. The first holds the "Decision Records by Type" pie chart ([ADR-177](./../release%200.4.0/adr-177-overview-pie-chart.md)); the second holds the "Decision Records by Status Over Time" velocity chart ([ADR-182](./../release%200.4.0/adr-182-overview-velocity-chart.md)). The third cell is still an empty placeholder.

The velocity chart answers *"how has the workflow moved over the last six weeks"* by computing each record's date-derived status as of six Fridays. What it does **not** answer at a glance is the simplest backlog question: *"how many records are in each state right now?"* A reader wanting the current shape of the backlog — how much is still `Proposed`, how much is stuck `In-Progress`, how much is `Implemented` — has to read the rightmost velocity bar (which is date-computed, not the authoritative marker) or scan the Status column of the table row by row.

The authoritative "current status" already exists per record: it is the status of the `*`-marked Status-table row, exposed as `Decision#current_status` and already rendered in the overview's Status column ([ADR-172](./../release%200.4.0/adr-172-current-status-marker.md)). This ADR surfaces the distribution of that value as the third chart.

The hard part is **scale skew**. In a mature project the terminal state dominates: there may be hundreds of `Implemented` records against two `In-Progress` and one `Proposed`. Two tempting fixes were considered and rejected (see Alternatives):

- A **time window** (only records touched in the last week or two) — but that re-asks the velocity chart's question, and worse, it *hides* the records a current-state view most needs to show: a story that has sat `In-Progress` for a month is exactly what should be visible, yet a two-week window would drop it.
- A **logarithmic value axis** — but `log(1) = 0`, so a single `Proposed` record would render as a zero-height, invisible bar (the very value we want to see), and log-scaled bar lengths mislead readers about ratios.

The chosen approach keeps the chart honest (linear, every record counted) and solves legibility by **annotating counts in the labels** rather than distorting the axis.

# Decision

Add a third chart to the Decision Records Overview page — a Chart.js **horizontal bar chart** titled "Decision Records by Current Status" — placed in the third `chart_cell` of the existing `decisions_overview_charts` grid. This fills the last reserved cell.

## Data source and aggregation

The chart aggregates `Decision#current_status` — the status of the row marked with `*` in each record's Status table ([ADR-172](./../release%200.4.0/adr-172-current-status-marker.md)) — **not** the date-derived `effective_status_on` used by the velocity chart. "Currently" means what the author has declared current via the marker.

For each record, the gem increments a count keyed by its current-status text. Status text is compared by **exact, case-sensitive equality after stripping surrounding whitespace**, identical to the velocity chart's rule (so `Proposed` and `proposed` are distinct, `Implemented` and `Implemented ` are the same). This deliberately reuses the heterogeneous-vocabulary handling already established for ADR/ISSUE/ENH workflows — no canonical status list is imposed.

## The "Undefined" category

`Decision#current_status` is `nil` when a record's Status table has **zero** `*` markers or **more than one** (per [ADR-172](./../release%200.4.0/adr-172-current-status-marker.md)). Rather than silently dropping such records, the chart counts them under a single explicit **"Undefined"** category.

This is intentional: "Undefined" is a **data-quality indicator**. A non-empty Undefined bar tells the reader that one or more decision records have a missing or ambiguous current-status marker and need to be corrected. The bar is rendered in a distinct neutral grey (the grey entry of `CHART_PALETTE`) so it reads as an anomaly to fix rather than as a normal workflow state.

## Legibility under scale skew

The value axis is **linear** — bar length is honestly proportional to count. To keep small categories readable next to a dominant one, **each category's count is embedded in its axis (tick) label**, e.g. `Implemented (1000)`, `In-Progress (2)`, `Proposed (1)`. The number is therefore always visible regardless of bar length, with no dependency on a data-labels plugin (core Chart.js only, consistent with the existing charts).

Horizontal orientation (`indexAxis: 'y'`) is chosen so that long status names and a growing number of distinct statuses render as stacked rows without axis-label rotation.

## Category ordering

Categories are ordered by **first appearance in decision-record parse order** (the same deterministic, vocabulary-agnostic ordering the velocity chart uses for its stack segments). The **"Undefined" category, when present, is placed last**, after all real statuses.

## Rendering

The chart is rendered inline using Chart.js (already loaded for the existing charts). Configuration:

- `type: 'bar'` with `options.indexAxis: 'y'` (horizontal bars)
- `data.labels` — one string per category, each carrying its count, e.g. `"Implemented (1000)"`; the "Undefined" category, when present, is last
- `data.datasets` — a single dataset whose `data` array holds the per-category counts
- `options.scales.x.beginAtZero: true` (linear, not logarithmic)
- `options.plugins.title` — `"Decision Records by Current Status"`
- `options.plugins.legend.display: false` — the category is named on the axis, so a legend is redundant
- `backgroundColor` — `palette_rgba(i, 0.5)` per bar for parity with the pie and velocity charts, except the "Undefined" bar, which uses the neutral grey palette entry; `borderWidth: 0`

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Done | 01-06-2026 | 02-06-2026 | New SRS items (SRS-083 onward) covering: a horizontal bar chart of decision-record counts by current status in the third chart cell; aggregation by the `*`-marked current status with exact case-sensitive text match; the "Undefined" category for records with a missing or ambiguous current-status marker; a linear scale with the count shown in each bar label; first-seen category ordering with "Undefined" last |
| Code | Done | 01-06-2026 | 02-06-2026 | Add a private helper on `DecisionsOverview` that tallies `current_status` across decisions (counting `nil` under "Undefined"), derives categories in first-seen order with "Undefined" appended last, and emits a horizontal-bar Chart.js config into the third `chart_cell`; embed each count in its label; reuse `palette_rgba` for colours and give "Undefined" the neutral grey entry; keep the legend hidden |
| Tests | Done | 01-06-2026 | 02-06-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: the chart canvas is emitted in the third chart cell; the chart type is `bar` with `indexAxis: 'y'`; the value scale is linear (no logarithmic axis); labels carry the counts; records are tallied under their `*`-marked status; records with zero or multiple markers fall under "Undefined"; "Undefined" is ordered last; an all-defined project emits no "Undefined" category |

# Out of Scope

- A logarithmic or symlog value axis. The scale is linear; skew is handled by labelling counts, not by distorting bar lengths.
- A time-windowed snapshot (last one/two weeks). The chart counts every record's current status; movement over time remains the velocity chart's concern ([ADR-182](./../release%200.4.0/adr-182-overview-velocity-chart.md)).
- Proportional / 100%-stacked presentation. The chart shows absolute counts.
- Aliasing or normalisation of status text (`Done` → `Implemented`, `New` → `Proposed`). Statuses are matched exactly, as in the velocity chart; a configurable alias table remains a possible future enhancement.
- Per-status colour mapping as a general theming surface. Only the special "Undefined" category gets a fixed (neutral grey) colour; all real statuses take palette colours by index.
- Listing or linking the specific records that fall under "Undefined". The chart only signals that some exist; the reader finds them via the empty Status cells in the table below.
- Correcting the underlying records flagged as "Undefined". The chart surfaces the data-quality issue; fixing the markers is authoring work.
- Drill-down or hover-to-filter interactions. The chart is read-only.
- A configurable chart type, orientation, or sort order.

# Consequences

## Positive

- The current shape of the backlog is visible at a glance, sourced from the authoritative `*`-marked status that the Status column already shows — consistent with the rest of the overview.
- Small in-flight categories stay legible next to a dominant terminal category because the count is in the label; the linear axis keeps the visual ratio honest.
- The "Undefined" category converts a silent data-quality problem (missing or duplicated `*` markers) into a visible, actionable indicator on the overview.
- Minimal new code and no new dependency: it reuses `current_status` ([ADR-172](./../release%200.4.0/adr-172-current-status-marker.md)), the chart-grid scaffolding ([ADR-177](./../release%200.4.0/adr-177-overview-pie-chart.md)), and the `CHART_PALETTE` / `palette_rgba` helpers ([ADR-182](./../release%200.4.0/adr-182-overview-velocity-chart.md)).
- Fills the last reserved `chart_cell`, completing the overview chart grid.

## Negative

- The chart trusts the `*` marker. A record marked with the wrong status shows under that wrong status (the chart cannot detect a *plausible-but-incorrect* marker); only missing/duplicate markers are caught, via "Undefined".
- A linear scale compresses every non-dominant category to a thin sliver when one state vastly outnumbers the others. The count labels mitigate this for reading exact values, but the geometry still under-emphasises small categories — the accepted trade-off against a misleading log axis.
- A growing "Undefined" bar signals accumulating data-quality debt but does not name the offending records; the reader must scan the table's blank Status cells to find them.

## Neutral

- The computation is O(records) tallies per render — negligible for any realistic record count.
- Read-side only: no change to the Status-table format, the row listing, or the individual decision-record pages. No data migration.
- "Undefined" is a category specific to this chart; the velocity chart's segment set and the pie chart are unchanged.
- The chart's current-status snapshot can differ from the velocity chart's latest bar, because this chart uses the `*` marker while the velocity chart derives status from dates. This is intended — the two charts answer different questions.

# Alternatives Considered

- **Logarithmic / symlog value axis** (the user's "log scale" option). Rejected: `log(1) = 0` renders a single-record category as an invisible zero-height bar — exactly the value the chart exists to surface — and log-scaled bar lengths mislead readers about ratios and require a non-zero baseline. Labelling counts on a linear axis achieves compact legibility without lying about magnitude.
- **Time-windowed snapshot, e.g. records from the last one or two weeks** (the user's "recent records" option). Rejected: it changes the question from "current state of the whole backlog" to "recently touched records", duplicating the velocity chart, and it hides long-stalled records — the ones a current-state view most needs to show.
- **Pie / donut by current status.** Rejected: tiny slices (1 of 1000+) are even less visible than tiny bars, and a pie cannot carry a clear per-category count label as cleanly as an axis tick.
- **Vertical bars.** Rejected: long, heterogeneous status names need rotated labels and read poorly; horizontal bars scale gracefully as the number of distinct statuses grows.
- **`chartjs-plugin-datalabels` for in-bar numbers.** Rejected: a new dependency for what an axis-label suffix achieves with core Chart.js.
- **Silently excluding records with an undefined current status.** Rejected per the user's direction: the "Undefined" category is a deliberate indicator that a record needs its status marker fixed.
- **100%-stacked single bar of proportions.** Rejected: hides absolute volume, and the velocity-chart ADR already rejected proportional presentation for the same reason.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.1 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview page shall include a horizontal bar chart visualising the number of Decision Records grouped by their current status, placed in the third chart cell of the Decision Records Overview charts grid. | >[SRS-083] |
| 2 | The current-status distribution chart shall count each Decision Record under its current status — the status value of the row marked with "*" in the record's Status table — with status text compared by exact case-sensitive equality after stripping surrounding whitespace. | >[SRS-084] |
| 3 | The current-status distribution chart shall group every Decision Record whose current status is undefined under a single "Undefined" category, so that records with a missing or ambiguous current-status marker are surfaced for correction. | >[SRS-085] |
| 4 | The current-status distribution chart shall use a linear value scale and shall display each category's record count within its bar label, so that categories with small counts remain legible alongside categories with large counts. | >[SRS-086] |
| 5 | The categories of the current-status distribution chart shall be ordered by their first appearance in Decision Record parse order, with the "Undefined" category, when present, placed last. | >[SRS-087] |

# References

- [ADR-170](./../release%200.4.0/adr-170-introduce-decision-records.md) — introduces decision records and the Decision Records Overview page
- [ADR-172](./../release%200.4.0/adr-172-current-status-marker.md) — defines the Status table format and the `*`-marker, the source of `current_status` aggregated here
- [ADR-177](./../release%200.4.0/adr-177-overview-pie-chart.md) — introduces the chart grid scaffolding and the first chart (pie)
- [ADR-182](./../release%200.4.0/adr-182-overview-velocity-chart.md) — fills the second chart cell and defines the `CHART_PALETTE` / `palette_rgba` helpers and the status-dictionary approach reused here; this ADR fills the third cell
- SRS-039 through SRS-082 in [srs.md](./../../specifications/srs/srs.md) — current requirements covering decision records and the overview page; the new SRS items added under this ADR extend that range

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
