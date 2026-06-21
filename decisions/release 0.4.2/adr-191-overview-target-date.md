---
title: "ADR-191: Target Date Extraction for the Decision Records Overview"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 11-06-2026 | Proposed |
|   | 11-06-2026 | Accepted |
|   | 11-06-2026 | In-Progress |
| * | 13-06-2026 | Implemented |

# Context

The Decision Records Overview page (`build/decisions/overview.html`) renders one row per decision record with columns for `#`, `Type`, `Status`, `Title`, `Start Date`, `Target Date`, `Release`, and `Owner`. The "Start Date" column was populated by [[adr-178-overview-start-date]], which derives a `start_date` attribute on each Decision instance as the **earliest** parseable date found in the Status table's `Date` column and the Scope table's `Start Date` column.

The adjacent "Target Date" column, however, is still rendered empty for every row — see the placeholder cell in [decisions_overview.rb:57](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L57). A reader can see when work on a decision was first picked up but not when it is expected to land, and there is no machine-readable "target completion" date on the Decision instance to drive future sorting, filtering, or charting.

The same two tables that supply the Start Date already carry the "latest" signal:

1. The `# Status` table — one row per lifecycle transition, with a `Date` column in `DD-MM-YYYY` format. The latest dated row corresponds to the most advanced (or planned) lifecycle state; records frequently pre-author future-dated rows (e.g. a planned `Implemented` row) as scheduling targets.
2. The `# Scope` table — one row per work item (Requirements / Code / Tests / ...), with a `Target Date` column in `DD-MM-YYYY` format. The latest cell corresponds to the work item expected to finish last.

The "true" target date of a decision record is the later of these two — whichever target is furthest out. As with the Start Date, authoring it as a third, hand-maintained frontmatter field would duplicate information already present in the document and drift out of sync. Deriving it during parsing keeps a single source of truth in the existing tables and mirrors the established Start Date approach exactly, but with `max` in place of `min`.

# Decision

Compute a `target_date` attribute on each Decision instance during parsing, and render it in the existing "Target Date" column of the Decision Records Overview page.

## Extraction rule

For a Decision Record, `target_date` is the **latest** parseable date found in either of:

- the `Date` column of the `# Status` table;
- the `Target Date` column of the `# Scope` table.

Both tables are already located by their section heading text (`Status` and `Scope`, case-sensitive, any heading level) using the existing `find_section_table` helper that `extract_start_date` relies on. Columns are identified by header text, not position, so that a future addition of columns to either table does not silently change which cell is read.

Dates are parsed in the `DD-MM-YYYY` format already used throughout the project. Cells that are empty, contain a non-date string, or do not match `DD-MM-YYYY` are skipped. If, after skipping unparseable cells, no dates remain in either table, `target_date` is `nil`.

Concretely, this is the symmetric counterpart of [`extract_start_date`](./../../../Almirah.Code/lib/almirah/doc_types/decision.rb#L47) — it reuses the same `collect_dates('Status', 'Date')` source and adds `collect_dates('Scope', 'Target Date')`, then takes the **maximum** instead of the minimum.

## Rendering

The existing empty "Target Date" cell in the Decision Records Overview ([decisions_overview.rb:57](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L57)) becomes populated with the `target_date` attribute, formatted as `DD-MM-YYYY`, exactly as the Start Date cell formats `start_date`. When the attribute is `nil`, the cell remains empty.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  |  |  | Done | 11-06-2026 | 11-06-2026 | New SRS items SRS-102 through SRS-106 in `srs.md` covering: the `target_date` attribute on a Decision Record; the extraction rule (maximum of Status `Date` column and Scope `Target Date` column); column lookup by header text; the `DD-MM-YYYY` parse format; the empty-cell fallback when no date is parseable; rendering of `target_date` in the existing "Target Date" column of the Decision Records Overview |
| 2 | Code | DEV |  |  |  | Done | 11-06-2026 | 11-06-2026 | Added a `target_date` accessor on `Decision`; added an `extract_target_date` method that collects dates from the Status `Date` column and the Scope `Target Date` column (reusing `collect_dates` / `find_section_table`) and takes the maximum; called it from the same `DocFabric.create_decision` path that invokes `extract_start_date`; populated the existing Target Date cell in `decisions_overview.rb` with the formatted value |
| 3 | Tests | TEST |  |  |  | Done | 11-06-2026 | 11-06-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: extraction picks the latest date when both tables have entries; extraction falls back to whichever table has a date when the other is empty or missing; extraction leaves the cell empty when no parseable date exists in either table; unparseable cells (e.g., "TBD", empty Target Date) are ignored without raising; the Target Date column is read by header text regardless of position |

# Out of Scope

- The "Owner" cell on the Decision Records Overview. It remains an empty placeholder and will be addressed in its own ADR.
- Sorting the Decision Records Overview by Target Date. Row order continues to follow [[adr-170-introduce-decision-records]] / [[enh-175-overview-sort-id]] (sequence number then ID).
- A configurable date format. `DD-MM-YYYY` is the only format recognised; alternate locale formats and ISO-8601 are not parsed.
- Validation that the Target Date is on or after the Start Date, or that Status `Date` cells and Scope `Target Date` cells are mutually consistent. Divergence between the two is expected during planning and is not an error.
- Visualising the Target Date in any chart on the overview page; the chart grid established by [[adr-177-overview-pie-chart]], [[adr-182-overview-velocity-chart]], and [[adr-185-overview-status-chart]] is untouched.
- Backfilling the Target Date attribute into rendered Decision Record pages themselves — this ADR only populates the overview's existing column.

# Consequences

## Positive

- The reserved "Target Date" column on the overview stops being dead space and starts carrying information that already exists in the source documents.
- A single source of truth: the dates live in the Status and Scope tables that authors already maintain; no new hand-edited frontmatter field can drift out of sync.
- Symmetry with Start Date: the same `collect_dates` / `find_section_table` machinery is reused with `max` instead of `min`, keeping the two attributes consistent and the change small.
- The extracted attribute is also available on the Decision instance for any future feature (sorting, timeline chart, overdue highlighting) without re-parsing.

## Negative

- The renderer now depends on the **header text** of the Scope table's `Target Date` column — renaming it would silently empty the column. Authors are expected to keep the header text stable; the test suite locks in `Target Date` as the expected header.
- Future-dated planning rows in the Status table (e.g. an unreached `Implemented` row) feed directly into the Target Date, so an over-optimistic planned date will be shown verbatim. This is intended — the target reflects what the record itself plans — but it is not validated against reality.

## Neutral

- Decision records authored before this ADR continue to work unchanged: extraction is purely additive and falls back to empty when neither table has parseable dates.
- The `DD-MM-YYYY` choice matches the format already used in every existing decision record; no migration of dates is required.

# Alternatives Considered

- **Author the target date as a frontmatter field (e.g., `target_date: 2026-06-11`).** Rejected: duplicates information already present in the Status and Scope tables and creates a third place that can drift, exactly as rejected for the Start Date in [[adr-178-overview-start-date]].
- **Use only the Scope table's latest Target Date.** Rejected: many decision records have no Scope table at all (purely architectural ADRs with no work breakdown) but always carry a Status table whose latest row is a meaningful target. Falling back to Status keeps the column populated for those.
- **Use only the Status table's latest Date.** Rejected: the Scope table's `Target Date` column can carry a later, finer-grained per-work-item target than the Status table's coarse lifecycle dates; the maximum-of-both rule captures the furthest-out target without forcing the author to pick.
- **Pick the earliest date in either table instead of the latest.** Rejected: that is precisely the Start Date, which already has its own column and attribute. "Target Date" by name and by reader expectation is the *last* / expected-completion moment associated with the record.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.1 |
| Issue Found in Version | 0.4.2 |
| Target Release Version | 0.4.2 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall expose a Target Date attribute on each Decision Record, computed as the latest calendar date found in the Date column of the Decision Record's Status table and in the Target Date column of the Decision Record's Scope table. | >[SRS-102] |
| 2 | The software shall recognise dates in the form "DD-MM-YYYY" when computing the Target Date attribute of a Decision Record. Cells whose contents do not match this form shall be skipped without raising an error. | >[SRS-103] |
| 3 | The software shall identify the Date column of a Decision Record's Status table and the Target Date column of a Decision Record's Scope table by their header text, case-sensitive, and not by column position. | >[SRS-104] |
| 4 | When neither the Status table's Date column nor the Scope table's Target Date column of a Decision Record contains a date matching the recognised form, the Target Date attribute of that Decision Record shall be undefined. | >[SRS-105] |
| 5 | The Decision Records Overview page shall render the Target Date attribute of each Decision Record in the existing Target Date column, formatted as "DD-MM-YYYY". The cell shall be empty when the Target Date attribute is undefined. | >[SRS-106] |

# References

- [ADR-178](./../release%200.4.0/adr-178-overview-start-date.md) — sibling change that introduced the Start Date attribute and the `extract_start_date` / `collect_dates` machinery this ADR mirrors with `max` instead of `min`
- [ADR-170](./../release%200.4.0/adr-170-introduce-decision-records.md) — introduces decision records, the Status table, and the Decision Records Overview page
- [ADR-172](./../release%200.4.0/adr-172-current-status-marker.md) — established the parse-the-Status-table pattern reused for section/column lookup
- SRS-061 through SRS-065 in [srs.md](./../../specifications/srs/srs.md) — the Start Date requirements these new Target Date items parallel

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
