---
title: "ADR-193: Scope Owner Column and Work-In-Progress Heatmap"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 13-06-2026 | Proposed |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

This is the first implementation step of the planning/flow roadmap captured in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md), which reads Almirah as an ALM through *Goldratt's Rules of Flow* (Critical Chain / Theory of Constraints). The roadmap's first principle — and, per the book, the single highest-leverage one — is that **bad multitasking is the primary reason work does not flow**: the more items a person has open simultaneously, the longer every one of them takes to finish. The remedy is to make open-work-per-person *visible* and cap it, before any heavier estimation or buffer machinery is built.

Almirah already has most of the substrate:

- Each Decision Record carries a `# Scope` table whose rows are work items (`Requirements / Code / Tests / ...`) with `Status`, `Start Date`, `Target Date`, and `Description` columns. The table is located by heading text and read column-by-column via `find_section_table` / `column_index` ([decision.rb](./../../../Almirah.Code/lib/almirah/doc_types/decision.rb)).
- Each record has a derived `current_status` — the `*`-marked row of its `# Status` table ([adr-172-current-status-marker]).
- The Decision Records Overview ([decisions_overview.rb](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb)) already renders a three-cell chart grid (type pie, velocity, status distribution) via `render_charts_grid`, and already reserves an **`Owner` column that is rendered as an empty placeholder** for every row ([decisions_overview.rb:60](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L60)).

Two things are missing to surface multitasking. First, there is **no notion of who owns a piece of work** — the Owner column is dead space. Second, there is **no view of how many records are open per owner**, so an owner carrying five simultaneous In-Progress records looks identical to one carrying a single record.

Both gaps can be closed without introducing any external store or any hand-maintained field that can drift, following exactly the derive-from-the-existing-table discipline established for Start Date and Target Date in [[adr-178-overview-start-date]] and [[adr-191-overview-target-date]].

# Decision

Introduce a record **Owner** dimension sourced from the Scope table, render it in the overview's existing Owner column, and add a **Work-In-Progress by Owner** chart to the overview chart grid with a configurable freeze line.

## Owner data source and extraction

Add an optional **`Owner` column to the `# Scope` table**. As with every other Scope column, it is identified by header text (case-sensitive, `Owner`), not by position, so existing records and any future column additions are unaffected.

Compute an `owners` attribute on each `Decision` instance during parsing: the **distinct, first-seen-ordered** list of non-empty values in the Scope table's `Owner` column, obtained with the same `find_section_table` / `column_index` helpers `extract_start_date` relies on. When the Scope table is absent, has no `Owner` column, or that column is entirely empty, `owners` is the empty list.

The unit of "work in progress" is the **Decision Record**, not the individual Scope row: a record whose derived `current_status` is `In-Progress` represents one open task, attributed to each of its `owners`. This deliberately reuses the already-derived `current_status` and adds no per-row status semantics in this step.

## Overview Owner column

Populate the existing, currently-empty Owner cell ([decisions_overview.rb:60](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L60)) with the record's `owners`, joined by a comma when there is more than one. The cell remains empty when `owners` is empty.

## Work-In-Progress by Owner chart

Add a fourth cell to the overview chart grid (`render_charts_grid`): a horizontal bar chart titled **"Work In Progress by Owner"**.

- One bar per owner. The bar length is the number of Decision Records whose `current_status` is `In-Progress` and whose `owners` include that owner.
- Owners with zero In-Progress records are omitted; bars are sorted descending by count.
- A dashed reference line is drawn at the **WIP freeze limit**. Bars at or below the limit use the normal palette; bars above it are drawn in the warning colour already used elsewhere in the palette, marking owners who are over-committed.

## Configuration

Read an optional freeze limit from `project.yml`:

```yaml
planning:
  wip_limit: 2
```

When `planning.wip_limit` is absent, default to `2`. A non-positive or non-integer value falls back to the default. This is the only new configuration key; it is parsed in `project_configuration.rb` alongside the existing `specifications` and `repositories` keys.

# Scope

| Item | Owner | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|
| Requirements | A.I. | To Do |  |  | New SRS items SRS-107 through SRS-112 in `srs.md`, under a new "Planning" chapter, covering: the optional `Owner` column on the Decision Record Scope table; the `owners` attribute (distinct, first-seen-ordered, non-empty values, read by header text); the empty-list fallback; rendering of `owners` in the overview Owner column; the Work-In-Progress by Owner chart (one bar per owner = count of In-Progress records that owner is on); and the `planning.wip_limit` configuration with its default and over-limit indication |
| Code | A.I. | To Do |  |  | Add an `owners` accessor on `Decision`; add `extract_owners` reusing `find_section_table` / `column_index`; call it from the same `DocFabric.create_decision` path that invokes `extract_start_date` / `extract_target_date`; populate the Owner cell in `decisions_overview.rb`; add a `wip_by_owner_chart_data` method and a fourth chart cell in `render_charts_grid`; read `planning.wip_limit` in `project_configuration.rb` with a default of 2 |
| Tests | A.I. | To Do |  |  | End-to-end tests under `spec/e2e/decisions_spec.rb`: owners are extracted distinct and in first-seen order; an absent/empty Owner column yields no owners and an empty overview cell; the column is read by header text regardless of position; the WIP chart counts only In-Progress records and attributes a multi-owner record to each owner; the reference line reflects `planning.wip_limit`; the default of 2 applies when the key is absent or invalid |

# Out of Scope

- **Per-Scope-item ownership and per-item In-Progress counting.** This step counts open work at the Decision Record level using the already-derived `current_status`. Finer-grained per-row work-in-progress is deferred.
- **Estimates, the critical chain, and project buffers.** These are roadmap steps 3–4 in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) and get their own ADRs; this ADR adds no `Est` / `Actual` columns.
- **The full-kit readiness check and `Depends On` column** (roadmap step 2).
- **Constraint load / capacity-over-time charting.** The WIP chart here counts *current* open records, not forecast load across future weeks.
- **Enforcement.** Exceeding `wip_limit` is surfaced visually (warning-coloured bar past the reference line); the build is not failed and no record is rejected.
- **Validating or normalising owner identity** (aliases, initials vs. full names). Owner strings are compared verbatim after trimming.

# Consequences

## Positive

- The reserved Owner column on the overview stops being dead space and starts carrying information that already lives in the Scope table.
- Bad multitasking becomes visible at a glance — the change the book argues delivers most of the flow improvement — with the smallest possible footprint: one optional column, one derived attribute, one chart.
- Single source of truth preserved: owners live in the Scope table authors already maintain; no new hand-edited frontmatter field can drift, consistent with [[adr-178-overview-start-date]] and [[adr-191-overview-target-date]].
- The `owners` attribute and the WIP tally are available on the Decision instance for later roadmap steps (capacity charts, fever chart) without re-parsing.

## Negative

- The renderer gains a dependency on the **header text** `Owner` in the Scope table; renaming it would silently empty the column and the chart. The test suite locks in `Owner` as the expected header, mirroring the Target Date caveat in [[adr-191-overview-target-date]].
- WIP is only as accurate as the `current_status` markers; a record left at `In-Progress` after work stops inflates an owner's bar until the Status table is updated.

## Neutral

- Records authored before this ADR continue to work unchanged: extraction is additive and falls back to an empty owner list and an empty overview cell.
- Already-rendered projects must be re-rendered to gain the column and chart; nothing is applied retroactively.
- `planning.wip_limit` is optional; projects that omit it get the default of 2 and need no migration.

# Alternatives Considered

- **A record-level `owner:` frontmatter field.** Rejected: it is a second place to maintain an owner that the Scope table can already express per work item, and it would drift — the same reasoning that rejected a frontmatter date in [[adr-191-overview-target-date]].
- **Counting In-Progress Scope rows rather than In-Progress records.** Deferred rather than rejected: it is the more precise multitasking measure but requires per-row status semantics not yet defined. This ADR uses the record-level `current_status` that already exists, and the finer measure can layer on later.
- **Failing the build when an owner exceeds `wip_limit`.** Rejected for this step: the goal is to make multitasking visible, not to block authors; a hard gate is premature before the team has lived with the signal.
- **Deriving owners from git authorship/blame.** Rejected: couples planning ownership to commit history, is unavailable for not-yet-started work, and breaks the file-is-the-source-of-truth model.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support an optional Owner column identifying the owner of each work item. | >[SRS-107] |
| 2 | The software shall expose an Owners attribute on each Decision Record, computed as the distinct, first-seen-ordered list of non-empty values in the Owner column of the Decision Record's Scope table. The Owner column shall be identified by its header text, case-sensitive, and not by column position. | >[SRS-108] |
| 3 | When a Decision Record has no Scope table, no Owner column, or an empty Owner column, the Owners attribute of that Decision Record shall be the empty list. | >[SRS-109] |
| 4 | The Decision Records Overview page shall render the Owners attribute of each Decision Record in the existing Owner column, comma-separated when there is more than one owner, and empty when the Owners attribute is empty. | >[SRS-110] |
| 5 | The Decision Records Overview page shall render a Work-In-Progress by Owner chart with one bar per owner whose length is the number of Decision Records whose current status is In-Progress and whose Owners attribute includes that owner, and shall draw a reference line at the configured work-in-progress freeze limit. | >[SRS-111] |
| 6 | The software shall read an optional planning work-in-progress limit from the project configuration. When the limit is absent or invalid, the software shall apply a default of 2. | >[SRS-112] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 1 of; maps the five Rules of Flow onto Almirah features
- [[adr-191-overview-target-date]] — sibling derive-from-the-Scope-table change whose `find_section_table` / `column_index` machinery and header-text-not-position discipline this ADR reuses
- [[adr-178-overview-start-date]] — introduced the `collect_dates` / derived-attribute pattern
- [[adr-172-current-status-marker]] — established the `*`-marked current status this ADR's WIP count is keyed on
- [[adr-177-overview-pie-chart]], [[adr-182-overview-velocity-chart]], [[adr-185-overview-status-chart]] — the existing overview chart grid the WIP chart joins as a fourth cell
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
