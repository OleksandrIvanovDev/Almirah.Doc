---
title: "ADR-193: Scope Owner Column and Work-In-Progress Heatmap"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 13-06-2026 | Proposed |
|   | 14-06-2026 | Analysis |
|   | 15-06-2026 | Accepted |
|   | 15-06-2026 | In-Progress |
|   | 15-06-2026 | Implemented |
| * | 09-07-2026 | Superseded |

# Context

This is the first implementation step of the planning/flow roadmap captured in [gfa.md](./../../specifications/gfa/gfa.md), which reads Almirah as an ALM through *Goldratt's Rules of Flow* (Critical Chain / Theory of Constraints). The roadmap's first principle — and, per the book, the single highest-leverage one — is that **bad multitasking is the primary reason work does not flow**: the more items a person has open simultaneously, the longer every one of them takes to finish. The remedy is to make open-work-per-person *visible* and cap it, before any heavier estimation or buffer machinery is built.

Almirah already has most of the substrate:

- Each Decision Record carries a `# Scope` table whose rows are work items (`Requirements / Code / Tests / ...`) with `Status`, `Start Date`, `Target Date`, and `Description` columns. The table is located by heading text and read column-by-column via `find_section_table` / `column_index` ([decision.rb](./../../../Almirah.Code/lib/almirah/doc_types/decision.rb)).
- Each record has a derived `current_status` — the `*`-marked row of its `# Status` table ([adr-172-current-status-marker]).
- The Decision Records Overview ([decisions_overview.rb](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb)) already renders a three-cell chart grid (type pie, velocity, status distribution) via `render_charts_grid`, and already reserves an **`Owner` column that is rendered as an empty placeholder** for every row ([decisions_overview.rb:60](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L60)).

Three things are missing to surface multitasking. First, there is **no notion of who owns a piece of work** — the Owner column is dead space. Second, there is **no view of how much work is open per owner**. Third — the subtle one — **analysis is invisible work**: a record spends its `Analysis` lifecycle phase being actively worked (investigation, requirements authoring, the decision itself), yet keying "open" on the record-level `In-Progress` status would miss all of it, hiding exactly the multitasking the chart exists to expose, on the role (the analyst / BA) that is most often the constraint.

The fix is to count work in progress at the level of the **Scope row** — the work item — attributed to that row's owner, using the row's own `Status`. This keeps the planning signal on the **bounded** per-row status vocabulary (`To Do` / `In-Progress` / `Done`) and independent of the record's **hand-marked, open** lifecycle status, which carries decision states such as `Analysis`, `Deployment`, or `Reopened` that have no Scope-row equivalent — the two remain separate, complementary views (see [[adr-172-current-status-marker]]). None of this needs an external store or a drift-prone field, following the derive-from-the-existing-table discipline of [[adr-178-overview-start-date]] and [[adr-191-overview-target-date]].

# Decision

Establish the **Scope row (work-item phase) as the unit of planning** for this and the following roadmap steps, add an explicit **`Analysis`** phase so analytical work is tracked, give each row an **Owner**, count **work in progress per owner from the rows' own `Status`**, render owners in the overview's existing Owner column, and **replace the overview's left (pie) chart with a Work-In-Progress by Owner chart** with a configurable freeze line — keeping the pie-chart code in place but unrendered.

## Owner data source and extraction

Add an optional **`Owner` column to the `# Scope` table**. As with every other Scope column, it is identified by header text (case-sensitive, `Owner`), not by position, so existing records and any future column additions are unaffected.

Compute an `owners` attribute on each `Decision` instance during parsing: the **distinct, first-seen-ordered** list of non-empty values in the Scope table's `Owner` column, obtained with the same `find_section_table` / `column_index` helpers `extract_start_date` relies on. When the Scope table is absent, has no `Owner` column, or that column is entirely empty, `owners` is the empty list.

The unit of "work in progress" is the **Scope row**, not the whole Decision Record: a row whose own `Status` is `In-Progress` is one open task, attributed to that row's `Owner`. So a record in its `Analysis` phase lights up WIP for the analyst (its `Analysis` / `Requirements` rows), while its `Code` / `Tests` rows — still `To Do` — add nothing for DEV / TEST. WIP therefore rides on the **bounded** per-row `Status` (`To Do` / `In-Progress` / `Done`) and never on the record's open-vocabulary lifecycle status.

## Scope phases and the Analysis row

The standard Scope work items are the four artifacts the record already tracks in its **Review Evidences** section — **Analysis** (the investigation and the decision record itself), **Requirements**, **Code**, and **Tests**. This ADR adds the **`Analysis`** row so analytical work is a first-class, owned, trackable item rather than an invisible prelude. Each row carries a leading **`#`** step number, an `Owner`, and a `Status` drawn from the bounded set `To Do` / `In-Progress` / `Done`. WIP needs only the `Owner` and `Status`; the `#` column and its ordering semantics (the phase-ordering gate) are introduced in [[adr-194-full-kit-readiness]] and shown here only so every record shares one Scope-row shape.

These per-row statuses are deliberately **independent** of the record's hand-marked lifecycle status ([[adr-172-current-status-marker]]): the lifecycle status is an **open** vocabulary (e.g. `Analysis`, `Accepted`, `In-Progress`, `Implemented`, `Deployment`, `Reopened`) describing the *decision's* state, several of whose values have no Scope-row counterpart. Every planning signal in this roadmap — WIP here, the kit gate, the chain, completion — reads the bounded per-row statuses; the lifecycle status is never a planning input, so extending it (a new `Deployment` or `Reopened` state) never disturbs the planning math, and a `Reopened` record simply sends rows back to `In-Progress` and re-lights their owners' WIP with no special handling.

## Overview Owner column

Populate the existing, currently-empty Owner cell ([decisions_overview.rb:60](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L60)) with the record's `owners`, joined by a comma when there is more than one. The cell remains empty when `owners` is empty.

## Work-In-Progress by Owner chart

**Replace the left (pie) chart cell** of the overview chart grid (`render_charts_grid`) with a chart titled **"Work In Progress by Owner"**. The existing pie-chart data and `<script>` builder are **kept in the code but no longer emitted** into the grid, so the pie can be re-enabled later without rewriting it; Chart.js simply never instantiates it. The grid therefore remains three cells: WIP by Owner (left), velocity (centre), status distribution (right).

The chart is a **combined bar/line** chart on the bundled Chart.js (v4, loaded from CDN on the overview page only — [base_document.rb:55](./../../../Almirah.Code/lib/almirah/doc_types/base_document.rb#L55)), requiring **no additional plugin**:

- A **vertical bar** dataset: one bar for **every owner named in any Decision Record's Scope table**, height = the number of **Scope rows, across all Decision Records, whose row `Status` is `In-Progress`** and whose `Owner` is that owner. An owner with no in-progress rows renders as a **zero-height bar** rather than being omitted: showing the full roster of roles keeps the freeze line spanning the chart even when only one owner is currently busy (a single bar would otherwise leave the category-aligned threshold with a single point and nothing to draw). Bars are sorted descending by count, so idle owners — tied at zero — fall to the end in first-seen order. The chart is vertical (`indexAxis: 'x'`) rather than horizontal so that the freeze line renders as a clean horizontal threshold using core Chart.js alone.
- A **line** dataset drawing the **WIP freeze limit** as a dashed horizontal threshold: the constant `wip_limit` value repeated across every owner label, with points hidden (`pointRadius: 0`) and a dashed stroke (`borderDash`). This is the combined bar/line — a top-level `type: 'bar'` chart with one dataset declared `type: 'line'`.
- **Over-limit highlight:** bars at or below the limit use the normal palette colour; bars above it use the palette's warning colour. This is achieved by passing `backgroundColor` as a **per-bar array** computed in Ruby (each owner's count compared to `wip_limit`) — the same array-of-colours technique `status_distribution_chart_data` already uses, so the highlight is fully supported and needs no plugin.

The choice of a vertical orientation and a core line dataset (rather than a plugin-based annotation line) is what keeps the freeze line dependency-free; see *Out of Scope*.

## Configuration

Read an optional freeze limit from `project.yml`:

```yaml
planning:
  wip_limit: 2
```

When `planning.wip_limit` is absent, default to `2`. A non-positive or non-integer value falls back to the default. This is the only new configuration key; it is parsed in `project_configuration.rb` alongside the existing `specifications` and `repositories` keys.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA |  |  |  | Done | 14-06-2026 | 15-06-2026 | This decision record: the per-row / altitude analysis, the two-independent-status-systems decision, and the Chart.js feasibility for the WIP chart |
| 2 | Requirements | BA |  |  |  | Done | 15-06-2026 | 15-06-2026 | New SRS items SRS-107 through SRS-112 in `srs.md`, under a new "Planning" chapter, covering: Scope work-item rows including an `Analysis` row, each with an `Owner` and a bounded `Status` (`To Do` / `In-Progress` / `Done`); the per-row owner and the record's distinct owner list (read by header text); the empty-owner fallback; rendering owners in the overview Owner column; the Work-In-Progress by Owner chart (one bar per owner = count of `In-Progress` Scope rows that owner is on); and the `planning.wip_limit` configuration with its default and over-limit indication |
| 3 | Code | DEV |  |  |  | Done | 15-06-2026 | 15-06-2026 | Read each Scope row's `Owner` and `Status` and expose a distinct `owners` list on `Decision`, reusing `find_section_table` / `column_index`, from the same `DocFabric.create_decision` path that invokes `extract_start_date` / `extract_target_date`; populate the Owner cell in `decisions_overview.rb`; add a `wip_by_owner_chart_data` method that counts `In-Progress` rows by owner and **replace the pie chart cell in `render_charts_grid` with the WIP combined bar/line chart, keeping the pie-building code in place but no longer emitting its cell**; read `planning.wip_limit` in `project_configuration.rb` with a default of 2 |
| 4 | Tests | TEST |  |  |  | Done | 15-06-2026 | 15-06-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: rows expose owner and bounded status read by header text regardless of position; an absent/empty Owner column yields no owner and an empty overview cell; the WIP chart counts only `In-Progress` rows and attributes each to its row owner (a record in its `Analysis` phase contributes to the analyst, not to DEV / TEST); the reference line reflects `planning.wip_limit`; the default of 2 applies when the key is absent or invalid; the per-row count is independent of the record's lifecycle status |

# Out of Scope

- **Deriving the record lifecycle status from row statuses.** The two remain independent and the lifecycle status stays hand-marked ([[adr-172-current-status-marker]]); per the project decision, the open lifecycle vocabulary (`Deployment`, `Reopened`, …) is not constrained to row-status values, and no row-to-lifecycle inference is performed.
- **Estimates, the critical chain, and project buffers.** These are roadmap steps 3–4 in [gfa.md](./../../specifications/gfa/gfa.md) and get their own ADRs; this ADR adds no `Est` / `Actual` columns.
- **The full-kit readiness check and `Depends On` column** (roadmap step 2).
- **Constraint load / capacity-over-time charting.** The WIP chart here counts *current* open records, not forecast load across future weeks.
- **Enforcement.** Exceeding `wip_limit` is surfaced visually (warning-coloured bar past the reference line); the build is not failed and no record is rejected.
- **Validating or normalising owner identity** (aliases, initials vs. full names). Owner strings are compared verbatim after trimming.
- **A plugin-based annotation line** (`chartjs-plugin-annotation`) for the freeze threshold. Only Chart.js core is loaded from CDN; the threshold is drawn with a core line dataset instead, which keeps the WIP chart dependency-free at the cost of fixing it to a vertical orientation.

# Consequences

## Positive

- The reserved Owner column on the overview stops being dead space and starts carrying information that already lives in the Scope table.
- Bad multitasking becomes visible at a glance — the change the book argues delivers most of the flow improvement — with the smallest possible footprint: one optional column, one derived attribute, one chart.
- Single source of truth preserved: owners live in the Scope table authors already maintain; no new hand-edited frontmatter field can drift, consistent with [[adr-178-overview-start-date]] and [[adr-191-overview-target-date]].
- The `owners` attribute and the WIP tally are available on the Decision instance for later roadmap steps (capacity charts, fever chart) without re-parsing.

## Negative

- The renderer gains a dependency on the **header text** `Owner` in the Scope table; renaming it would silently empty the column and the chart. The test suite locks in `Owner` as the expected header, mirroring the Target Date caveat in [[adr-191-overview-target-date]].
- WIP is only as accurate as the per-row `Status` values; a row left at `In-Progress` after work stops inflates its owner's bar until the Scope table is updated.

## Neutral

- Records authored before this ADR continue to work unchanged: extraction is additive and falls back to an empty owner list and an empty overview cell.
- Already-rendered projects must be re-rendered to gain the column and chart; nothing is applied retroactively.
- `planning.wip_limit` is optional; projects that omit it get the default of 2 and need no migration.
- The "Decision Records by Type" pie chart is retained in code but no longer rendered; re-enabling it later is a one-line change to `render_charts_grid` (re-emit its cell) and requires no data work, since its data builder is left intact.

# Alternatives Considered

- **A record-level `owner:` frontmatter field.** Rejected: it is a second place to maintain an owner that the Scope table can already express per work item, and it would drift — the same reasoning that rejected a frontmatter date in [[adr-191-overview-target-date]].
- **Counting In-Progress records rather than In-Progress Scope rows.** Rejected: keying WIP on the record-level lifecycle status hides analytical work (which happens in the `Analysis` phase, before any `In-Progress` lifecycle state) and over-attributes a record to the owners of phases not yet active. Counting active rows by row owner is both more precise and the only way the analyst's load becomes visible. (This reverses the deferral in the first draft of this ADR; the per-row unit is now the foundation for steps 2–4.)
- **Failing the build when an owner exceeds `wip_limit`.** Rejected for this step: the goal is to make multitasking visible, not to block authors; a hard gate is premature before the team has lived with the signal.
- **Deriving owners from git authorship/blame.** Rejected: couples planning ownership to commit history, is unavailable for not-yet-started work, and breaks the file-is-the-source-of-truth model.
- **A horizontal bar chart (`indexAxis: 'y'`) with the freeze line as a vertical threshold.** Rejected: with core Chart.js the only dependency-free way to draw the threshold is a line dataset, which follows the category axis — clean as a horizontal line on a *vertical* bar chart, but awkward on a horizontal one (the line would track the bars). A vertical chart gives a tidy horizontal threshold with no plugin.
- **Adding a separate cell and keeping the pie (a four-cell grid).** Rejected per the request: the WIP chart is the more valuable left-hand view, and the pie is retained in code (re-enableable) rather than occupying grid space.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support work-item rows — including an Analysis row — each carrying an Owner and a Status whose value is one of To Do, In-Progress, or Done. These per-row statuses shall be independent of the Decision Record's hand-marked lifecycle status. | >[SRS-107] |
| 2 | The software shall associate each Scope row with the owner named in its Owner column, and shall expose on each Decision Record the distinct, first-seen-ordered list of its rows' owners. The Owner column shall be identified by its header text, case-sensitive, and not by column position. | >[SRS-108] |
| 3 | When a Scope row has no owner, that row shall contribute no owner; when a Decision Record has no Scope table, no Owner column, or an empty Owner column, its distinct owner list shall be empty. | >[SRS-109] |
| 4 | The Decision Records Overview page shall render the distinct owner list of each Decision Record in the existing Owner column, comma-separated when there is more than one owner, and empty when the list is empty. | >[SRS-110] |
| 5 | The Decision Records Overview page shall render a Work-In-Progress by Owner chart with one bar for every owner named in any Decision Record's Scope table, each bar's length being the number of Scope rows, across all Decision Records, whose row Status is In-Progress and whose Owner is that owner, taken as zero when the owner has none, and shall draw a reference line at the configured work-in-progress freeze limit. | >[SRS-111] |
| 6 | The software shall read an optional planning work-in-progress limit from the project configuration. When the limit is absent or invalid, the software shall apply a default of 2. | >[SRS-112] |

# References

- [gfa.md](./../../specifications/gfa/gfa.md) — the roadmap this ADR is step 1 of; maps the five Rules of Flow onto Almirah features
- [[adr-191-overview-target-date]] — sibling derive-from-the-Scope-table change whose `find_section_table` / `column_index` machinery and header-text-not-position discipline this ADR reuses
- [[adr-178-overview-start-date]] — introduced the `collect_dates` / derived-attribute pattern
- [[adr-172-current-status-marker]] — the hand-marked record lifecycle status, which this ADR keeps **independent** of the per-row `Status` that drives WIP, and whose vocabulary stays open (`Deployment`, `Reopened`, …)
- [[adr-177-overview-pie-chart]], [[adr-182-overview-velocity-chart]], [[adr-185-overview-status-chart]] — the existing overview chart grid; the WIP chart replaces the pie cell
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
