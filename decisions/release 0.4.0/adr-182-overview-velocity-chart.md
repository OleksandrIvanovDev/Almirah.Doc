---
title: "ADR-182: Velocity Chart on the Decision Records Overview"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 23-05-2026 | Proposed |
|   | 23-05-2026 | Accepted |
|   | 23-05-2026 | In-Progress |
| * | 23-05-2026 | Implemented |

# Context

The Decision Records Overview page reserves three chart cells inside `decisions_overview_charts`. The first cell holds the "Decision Records by Type" pie chart introduced by [[adr-177-overview-pie-chart]]. The remaining two cells are placeholders.

A pie chart conveys the current distribution of records by type, but it says nothing about how the project is moving through the workflow. A reader scanning the overview cannot tell whether the team has been accumulating `Proposed` records faster than they reach `Implemented`, whether work has stalled in `In-Progress`, or whether the past six weeks were productive at all. That signal already exists in every decision record's `# Status` table — each row records the date a status was reached — but it is not surfaced anywhere on the overview.

This ADR adds a second chart that shows, week by week for the last six weeks, how many decision records were in each status as of that week's Friday. Friday is chosen as the snapshot date because it is the conventional Western end-of-week marker and aligns with how progress is typically reported.

Different decision records may use different status vocabularies (a strict ADR may use `Proposed → Accepted → In-Progress → Implemented`; an issue record may use `New → Investigating → Fixed`; an enhancement may use `Proposed → Rejected`). The chart cannot assume a single canonical workflow. The combined picture is built as a dictionary: every distinct status text encountered across all records becomes its own stack segment, and the bar for a given Friday simply counts how many records were in each status on that day.

# Decision

Add a second chart to the Decision Records Overview page — a Chart.js stacked bar chart titled "Decision Records by Status Over Time" — placed in the second `chart_cell` of the existing `decisions_overview_charts` grid (the third cell remains reserved).

## Snapshot dates

The chart shows **six bars** corresponding to the **six most recent Fridays** on or before the date of rendering, ordered left-to-right oldest-to-newest. The most recent Friday is the largest calendar date that is both a Friday and not later than the build date; the remaining five are the five Fridays before it at seven-day intervals.

For a build performed on 23-05-2026 (Saturday), the six Fridays are: 17-04-2026, 24-04-2026, 01-05-2026, 08-05-2026, 15-05-2026, 22-05-2026.

The X-axis labels render in the same `DD-MM-YYYY` format already used elsewhere in Almirah for dates.

## Status as of a given Friday

For each decision record and each of the six Fridays, the gem computes the record's effective status as of that Friday using this rule:

1. Take the record's Status table (found by locating the `Status` section heading, reusing the existing `find_section_table` helper).
2. From the table's rows, keep those whose `Date` column contains a value parseable as `DD-MM-YYYY` and whose parsed date is **on or before** the Friday.
3. Among the kept rows, pick the one with the **latest** parsed date.
4. If two or more kept rows share that latest date, pick the one that appears **later in the table** (document order).
5. The picked row's `Status` column value, after stripping whitespace, is the record's effective status as of that Friday.

If no row qualifies — i.e. every parseable date in the table is **after** the Friday — the record did not yet exist as of that Friday and **does not contribute** to that bar.

If the record has no `Status` section, no table in that section, or no rows with parseable dates, the record contributes to **no bar** of the chart (it is skipped entirely for the velocity chart, even though it still appears in the row listing below).

The future-dated `Implemented` row that authors often write up-front for planning is handled naturally by this rule: it is simply ignored on Fridays that precede its date.

## Combined workflow (status dictionary)

The chart does not assume a fixed status vocabulary. For each Friday, the gem builds a dictionary keyed by status text — incrementing the count for an existing key and adding a new key for an as-yet-unseen status. The set of stack segments in the chart is the union of all keys across all six Fridays' dictionaries.

Status values are compared by **exact, case-sensitive text equality after stripping surrounding whitespace**. `Proposed` and `proposed` are different segments; `Implemented` and `Implemented ` (trailing space) are the same.

Segment ordering: segments are stacked in **first-seen order**, computed by walking the six Fridays oldest-first and, within each Friday, walking decision records in parse order. This produces a stable, deterministic order without imposing a canonical workflow — which is what the user requested ("the order does not matter"). A future enhancement may revisit the ordering if a particular convention emerges.

## Rendering

The chart is rendered inline in the overview HTML using Chart.js (already loaded for the existing pie chart). Configuration:

- `type: 'bar'`
- `data.labels` — array of six `DD-MM-YYYY` strings
- `data.datasets` — one dataset per unique status, each with a six-element `data` array (zeros for Fridays where the count is absent)
- `options.scales.x.stacked: true` and `options.scales.y.stacked: true`
- `options.plugins.title` — `"Decision Records by Status Over Time"`
- `options.plugins.legend` — default placement; legend entries show the status text

Chart.js' default colour palette is used. The dataset's `backgroundColor` and `borderColor` are left unset so the Chart.js v4 Colors plugin auto-fills them: each dataset gets a translucent (`alpha 0.5`) fill paired with a fully opaque border. Because the bar element's default `borderWidth` is 0, only the soft pastel fill is visible.

## Visual parity with the existing pie chart

The pie chart introduced by [[adr-177-overview-pie-chart]] was rendered with the Chart.js defaults for a single-dataset categorical chart, which means each slice's `backgroundColor` defaulted to a **fully opaque** colour from the palette. Sitting next to the new velocity chart's soft pastel bars, the saturated pie slices stood out as visibly darker even though both charts use the same palette.

To bring the two charts into visual parity, this ADR also defines a small shared palette helper and applies it to the pie dataset:

- A `CHART_PALETTE` constant in `DecisionsOverview` holds the seven RGB triples that match the Chart.js v4 default palette (blue, red, orange, yellow, green, purple, grey), keeping the colour choices in one place for any future chart.
- A private `palette_rgba(index, alpha)` helper formats an RGB triple from `CHART_PALETTE` at a given alpha as a CSS `rgba(...)` string.
- The pie dataset is emitted with an explicit `backgroundColor` array built from `palette_rgba(i, 0.5)` for each slice, and with `borderWidth: 0` so no border line is drawn between slices.

The result: pie slices and stacked-bar segments use exactly the same colours at the same opacity, with no visible borders in either chart. The shared palette is private to `DecisionsOverview`; it is not a public theming surface.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  |  |  | Done | 23-05-2026 | 23-05-2026 | New SRS items in `srs.md` covering: the velocity chart placement in the second chart cell of the Decision Records Overview; the six-Friday window ending with the latest Friday on or before the build date; the status-as-of-Friday algorithm (latest parseable date ≤ Friday, document-order tie-break); the pre-existence rule (record skipped on Fridays before its earliest dated row); the missing-Status-table rule (record skipped for the chart); the combined-workflow dictionary built by exact case-sensitive text match |
| 2 | Code | DEV |  |  |  | Done | 23-05-2026 | 23-05-2026 | Add an `effective_status_on(date)` method on `Decision` that reuses `find_section_table('Status')` and applies the latest-date/document-order rule; add a private helper on `DecisionsOverview` that computes the six Fridays from `Date.today`, builds the per-Friday status dictionaries, derives the union of status segments in first-seen order, and emits the Chart.js stacked-bar config into the second `chart_cell`; ensure the third `chart_cell` remains an empty placeholder; add a `CHART_PALETTE` constant and a private `palette_rgba(index, alpha)` helper, and use them to emit the pie chart's dataset with an explicit half-opacity `backgroundColor` array and `borderWidth: 0` so the pie's colour treatment matches the stacked-bar chart |
| 3 | Tests | TEST |  |  |  | Done | 23-05-2026 | 23-05-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: the velocity chart canvas is emitted in the second chart cell; the chart type is `bar` and both axes are stacked; the labels array contains six `DD-MM-YYYY` strings; six bars are rendered; the latest-date tie-break by document order works; future-dated rows are ignored before their date; pre-existence (Friday before first row) leaves the record off the bar; missing Status table leaves the record off every bar; mixed-workflow records contribute to separate segments; unknown status text becomes its own segment without raising |

# Out of Scope

- A configurable snapshot weekday or week-end marker. Friday is fixed; Monday-, Sunday-, or month-end-snapshots can be addressed in a separate enhancement.
- A configurable window length. Six weeks (six Fridays) is fixed; arbitrary windows (4 weeks, 12 weeks, custom range) are not provided.
- User-configurable colours, accessibility-aware palettes, or per-status colour mapping. The shared `CHART_PALETTE` defined here is private and matches the Chart.js v4 default palette one-for-one; exposing it as a theming surface, mapping specific statuses to specific colours, or providing colour-blind-friendly variants is out of scope.
- A custom segment order or a canonical workflow. Segments are stacked in first-seen order; a fixed order such as `Proposed → Accepted → In-Progress → Implemented → Rejected` is not imposed.
- Aliasing or normalisation of status text. `Proposed` and `proposed` and `propsed` (typo) are distinct segments. No fuzzy matching, no case folding, no synonym tables.
- Drill-down interactions on bar segments (click-to-filter the table, hover-to-list contributing records). The chart is read-only.
- A rate-of-change or "velocity" metric in the traditional Agile sense (records-finished-per-week as a number). The chart shows distribution snapshots, not throughput.
- Cumulative totals, moving averages, or trend lines.
- Visualising the per-record workflow itself (timeline / Gantt of one record). This ADR aggregates across records; per-record timelines are a separate concern.
- Server-side caching of the per-Friday calculation. Almirah is a one-off build tool; the calculation runs once per `almirah please` invocation.

# Consequences

## Positive

- A reader of the overview immediately sees how the workflow has moved over the last six weeks, without opening individual records.
- The chart surfaces stalled work: a segment that grows week-over-week without other segments shrinking indicates accumulation rather than throughput.
- The dictionary-of-statuses approach accommodates heterogeneous workflows across ADR / ISSUE / ENH record types without forcing a single canonical vocabulary.
- The implementation reuses existing infrastructure: `find_section_table` from [[adr-178-overview-start-date]], the Status-table format from [[adr-172-current-status-marker]], the chart-grid scaffolding from [[adr-177-overview-pie-chart]], and the `DD-MM-YYYY` date parser already shared with the Start Date extractor.
- The third `chart_cell` remains reserved for a future chart (per [[adr-177-overview-pie-chart]]).

## Negative

- Status text matching is exact and case-sensitive. A single typo (`Implemnted` vs `Implemented`) creates a new segment in the legend that visually competes with the correct one. The test suite cannot catch this for arbitrary author content; the chart legend itself becomes the diagnostic.
- Records authored without a Status table or with non-`DD-MM-YYYY` dates are silently excluded from the chart. They still appear in the row listing below, but a reader cannot tell from the chart that they exist.
- The chart's value diminishes early in a project's life: if all records are recent, the four oldest Friday bars are empty or near-empty.
- A record whose earliest Status row is future-dated (e.g. authored today but `Proposed` listed for next week) contributes to **no** bars until the planning date arrives. This is consistent with the chosen algorithm but may surprise authors who expect the record to "exist now".

## Neutral

- The build time grows by O(records × 6) date comparisons per render, which is negligible for any realistic record count.
- The existing pie chart's dataset now carries an explicit `backgroundColor` and `borderWidth: 0`, but its data, labels, title, and behaviour are unchanged. The row listing and the individual Decision Record HTML pages are untouched.
- No data migration. The Status table format is unchanged; the new chart is purely a read-side computation.
- The legend on the chart depends on the calendar window. A status that has not appeared in any record in the last six weeks will not be shown, even if it is the current status of some record. This is by design — the chart is a six-week velocity snapshot, not a global vocabulary.

# Alternatives Considered

- **One line chart per status instead of a stacked bar chart.** Rejected: lines for many heterogeneous status names become visually noisy and force the reader to mentally sum totals. A stacked bar shows both per-status counts and the running total at a glance.
- **100%-stacked bars showing proportions instead of absolute counts.** Rejected: hides volume changes. A week where the team adds eight `Proposed` records looks identical to a week where the team adds one if both produce the same proportion.
- **Daily snapshots instead of weekly Fridays.** Rejected: 42 bars over six weeks is visually unreadable and most records do not change status on a daily cadence. Weekly Fridays are a stable, low-noise sampling.
- **Use Monday or Sunday as the week marker.** Rejected: Friday is the conventional Western end-of-week and matches how project status is most commonly reported. Snapshot-weekday can be revisited as a future enhancement if needed.
- **Walk Status rows in document order only, ignoring date sort.** Rejected (this was Option B in the user-facing question): assumes the author wrote rows in chronological order. The chosen algorithm reads the same answer for well-ordered tables but is robust to author reorderings or out-of-order rows.
- **Use the `*`-marked current status for all six bars.** Rejected (this was Option C in the user-facing question): degenerates the chart into six identical bars and discards the whole point of a velocity view.
- **Impose a fixed canonical status order** (`Proposed → Accepted → In-Progress → Implemented → Rejected`) **with unknowns appended at the end.** Rejected per the user's direction. The dictionary-of-unknowns approach keeps every workflow first-class and avoids hard-coding any single vocabulary.
- **Synonym/alias mapping** (`Done` → `Implemented`, `New` → `Proposed`). Rejected: out of scope. A future enhancement can introduce a configurable alias table once authoring conventions are established.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview page shall include a stacked bar chart visualising decision-record counts grouped by status across a trailing six-week window, placed in the second chart cell of the Decision Records Overview charts grid. | >[SRS-071] |
| 2 | The velocity chart shall display six bars, one per Friday, corresponding to the six most recent Fridays on or before the date the page is rendered, ordered left-to-right oldest-to-newest, with each X-axis label formatted as "DD-MM-YYYY". | >[SRS-072] |
| 3 | The status of a Decision Record as of a given calendar date shall be computed as the Status table row whose parseable date is the latest one that is on or before that calendar date; when multiple rows share that latest date, the row appearing later in the table shall be selected. | >[SRS-073] |
| 4 | When the earliest parseable Status table date of a Decision Record is later than the calendar date of a velocity chart bar, that Decision Record shall not contribute to that bar. | >[SRS-074] |
| 5 | Decision Records that have no Status section, no table in their Status section, or no Status table row whose Date cell is parseable as "DD-MM-YYYY" shall not contribute to any bar of the velocity chart. | >[SRS-075] |
| 6 | The set of stack segments in the velocity chart shall be the union of every distinct status text encountered across all Decision Records and all six bars, with status text compared by exact case-sensitive equality after stripping surrounding whitespace. | >[SRS-076] |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records and the Decision Records Overview page
- [ADR-172](./adr-172-current-status-marker.md) — defines the Status table format and the `*`-marker for current status
- [ADR-177](./adr-177-overview-pie-chart.md) — introduces the chart grid scaffolding and the first chart (pie); this ADR fills the second cell
- [ADR-178](./adr-178-overview-start-date.md) — establishes the `find_section_table` helper and `DD-MM-YYYY` date parsing reused here
- [ADR-181](./adr-181-overview-release-version.md) — sibling overview-data extraction; precedent for adding new attributes derived from existing tables
- SRS-039 through SRS-070 in [srs.md](./../specifications/srs/srs.md) — current requirements covering decision records and the overview page; new SRS items added under this ADR extend that range

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47)
