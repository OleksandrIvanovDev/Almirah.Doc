---
title: "ADR-178: Start Date Extraction for the Decision Records Overview"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-05-2026 | Proposed |
|   | 21-05-2026 | Accepted |
|   | 21-05-2026 | In-Progress |
| * | 21-05-2026 | Implemented |

# Context

The Decision Records Overview page (`build/decisions/overview.html`) already reserves a "Start Date" column in its header (introduced together with the page layout), but the cell is rendered empty for every row — see [decisions_overview.rb:53](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L53). A reader cannot tell at a glance when work on a decision was first picked up, and there is no machine-readable "earliest activity" date on the Decision instance to drive future sorting, filtering, or charting.

Every decision record already carries dates in two places:

1. The `# Status` table — one row per lifecycle transition, with a `Date` column in `DD-MM-YYYY` format. The earliest row corresponds to the record being first proposed.
2. The `# Scope` table — one row per work item (Requirements / Code / Tests / ...), with a `Start Date` column in `DD-MM-YYYY` format. The earliest cell corresponds to the first work item that started.

The "true" start date of a decision record is the earlier of these two — whichever signal arrived first. Authoring it as a third, hand-maintained field in the frontmatter would duplicate information already present in the document and drift out of sync. Deriving it during parsing keeps a single source of truth in the existing tables.

# Decision

Compute a `start_date` attribute on each Decision instance during parsing, and render it in the existing "Start Date" column of the Decision Records Overview page.

## Extraction rule

For a Decision Record, `start_date` is the **earliest** parseable date found in either of:

- the `Date` column of the `# Status` table;
- the `Start Date` column of the `# Scope` table.

Both tables are already located by their section heading text (`Status` and `Scope`, case-sensitive, any heading level). The Status section is identified by the existing `extract_current_status` machinery — the Scope section uses the same locate-by-heading approach.

Columns are identified by header text, not position, so that a future addition of columns to either table does not silently change which cell is read.

Dates are parsed in the `DD-MM-YYYY` format already used throughout the project. Cells that are empty, contain a non-date string, or do not match `DD-MM-YYYY` are skipped. If, after skipping unparseable cells, no dates remain in either table, `start_date` is `nil`.

## Rendering

The existing empty "Start Date" cell in the Decision Records Overview becomes populated with the `start_date` attribute, formatted as `DD-MM-YYYY`. When the attribute is `nil`, the cell remains empty.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  |  |  | Implemented | 21-05-2026 | 21-05-2026 | New SRS items in `srs.md` covering: the `start_date` attribute on a Decision Record; the extraction rule (minimum of Status `Date` column and Scope `Start Date` column); column lookup by header text; the `DD-MM-YYYY` parse format; the empty-cell fallback when no date is parseable; rendering of `start_date` in the existing "Start Date" column of the Decision Records Overview |
| 2 | Code | DEV |  |  |  | Implemented | 21-05-2026 | 21-05-2026 | Add a `start_date` accessor on `Decision`; extend the Decision parser to locate the Scope section and read the "Start Date" column; share heading-lookup logic with `extract_current_status`; compute the minimum across both tables; populate the existing Start Date cell in `decisions_overview.rb` |
| 3 | Tests | TEST |  |  |  | Implemented | 21-05-2026 | 21-05-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: extraction picks the earliest date when both tables have entries; extraction falls back to whichever table has a date when the other is empty or missing; extraction returns nil when no parseable date exists in either table; unparseable cells (e.g., empty Target Date, free-text Description) are ignored without raising; the rendered Start Date column carries the formatted value and is empty when undefined |

# Out of Scope

- The "Target Date" and "Owner" cells on the Decision Records Overview. They remain empty placeholders and will be addressed in their own ADRs.
- Sorting the Decision Records Overview by Start Date. Row order continues to follow [[adr-170-introduce-decision-records]] / [[enh-175-overview-sort-id]] (sequence number then ID).
- A configurable date format. `DD-MM-YYYY` is the only format recognised; alternate locale formats and ISO-8601 are not parsed.
- Validation that Status `Date` cells and Scope `Start Date` cells are consistent with each other (e.g., warning when Scope claims work started before the Status table's first row). Divergence between the two is expected during planning and is not an error.
- Visualising the Start Date in any chart on the overview page (the chart grid reserved by [[adr-177-overview-pie-chart]] remains unrelated to this change).
- Backfilling the Start Date attribute into rendered Decision Record pages themselves — this ADR only populates the overview's existing column.

# Consequences

## Positive

- The reserved "Start Date" column on the overview stops being dead space and starts carrying information that already exists in the source documents.
- A single source of truth: the dates live in the Status and Scope tables that authors already maintain; no new hand-edited frontmatter field can drift out of sync.
- The extracted attribute is also available on the Decision instance for any future feature (sorting, timeline chart, age-based highlighting) without re-parsing.

## Negative

- The renderer now depends on the **header text** of the Scope table — renaming "Start Date" to e.g. "Started" would silently empty the column. Authors are expected to keep the header text stable; the test suite locks in `Start Date` as the expected header.

## Neutral

- Decision records authored before this ADR continue to work unchanged: extraction is purely additive and falls back to empty when neither table has parseable dates.
- The `DD-MM-YYYY` choice matches the format already used in every existing decision record; no migration of dates is required.

# Alternatives Considered

- **Author the start date as a frontmatter field (e.g., `start_date: 2026-05-21`).** Rejected: duplicates information already present in the Status and Scope tables and creates a third place that can drift. Decision records are append-only documents; deriving from existing tables keeps maintenance cost zero.
- **Use only the Status table's earliest Date.** Rejected: the Status table can be authored with all rows dated to the day the record was *written up*, even when the underlying work started earlier (a common pattern for retroactive ADRs). The Scope table's `Start Date` column then carries the older "work-started" signal, and the minimum-of-both rule captures the earlier of the two without forcing the author to pick.
- **Use only the Scope table's earliest Start Date.** Rejected: many decision records have no Scope table at all (purely architectural ADRs with no work breakdown) but always carry a Status table. Falling back to Status keeps the column populated for those.
- **Pick the latest date in either table instead of the earliest.** Rejected: "Start Date" by name and by reader expectation is the *first* moment associated with the record; the latest date is closer to "current date" or "target date" and already has its own column.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall expose a Start Date attribute on each Decision Record, computed as the earliest calendar date found in the Date column of the Decision Record's Status table and in the Start Date column of the Decision Record's Scope table. | >[SRS-061] |
| 2 | The software shall recognise dates in the form "DD-MM-YYYY" when computing the Start Date attribute of a Decision Record. Cells whose contents do not match this form shall be skipped without raising an error. | >[SRS-062] |
| 3 | The software shall identify the Date column of a Decision Record's Status table and the Start Date column of a Decision Record's Scope table by their header text, case-sensitive, and not by column position. | >[SRS-063] |
| 4 | When neither the Status table's Date column nor the Scope table's Start Date column of a Decision Record contains a date matching the recognised form, the Start Date attribute of that Decision Record shall be undefined. | >[SRS-064] |
| 5 | The Decision Records Overview page shall render the Start Date attribute of each Decision Record in the existing Start Date column, formatted as "DD-MM-YYYY". The cell shall be empty when the Start Date attribute is undefined. | >[SRS-065] |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records, the Status table, and the Decision Records Overview page
- [ADR-172](./adr-172-current-status-marker.md) — established the parse-the-Status-table pattern (`extract_current_status`) reused here for the Scope table lookup
- [ADR-174](./adr-174-dr-specification-links.md) — precedent for an ADR that adds software requirements via an "Affected Documents" table
- [ADR-177](./adr-177-overview-pie-chart.md) — sibling change to the Decision Records Overview page; this ADR leaves the chart grid untouched
- SRS-039 through SRS-060 in [srs.md](./../../specifications/srs/srs.md) — current requirements covering decision records; new SRS items added under this ADR extend that range

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47)
