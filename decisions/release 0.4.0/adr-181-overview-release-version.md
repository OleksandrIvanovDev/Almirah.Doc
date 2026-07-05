---
title: "ADR-181: Target Release Version Column on the Decision Records Overview"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 22-05-2026 | Proposed |
|   | 22-05-2026 | Accepted |
|   | 22-05-2026 | In-Progress |
| * | 22-05-2026 | Implemented |

# Context

The Decision Records Overview page currently shows, for each record, the columns: `#`, `Type`, `Status`, `Title`, `Start Date`, `Target Date`, `Owner`. The first three were introduced by [[adr-170-introduce-decision-records]] and [[adr-172-current-status-marker]]; Start Date was added by [[adr-178-overview-start-date]]; Target Date and Owner remain reserved placeholders.

A reader scanning the overview cannot tell from the table which release a given record is scheduled for. That information already exists in every decision record: the `Software Versions` section carries a two-column table whose `Software Version Category` column includes a `Target Release Version` row, and whose `Software Version ID` column holds the actual version string (e.g., `0.3.1`, `0.4.0`). Authoring a third, hand-maintained version field on the overview row would duplicate this information and drift out of sync. Deriving it during parsing keeps a single source of truth in the existing table.

# Decision

Add a new `Release` column to the Decision Records Overview page, positioned between the existing `Target Date` and `Owner` columns. Populate each row with the decision record's Target Release Version, extracted from the record's `Software Versions` section during parsing and exposed as a `target_release_version` attribute on the Decision instance.

## Header rendering

The column header renders as `<th title="Target Release Version">Release</th>`. The visible label is the short word `Release` so the overview table stays compact; hovering reveals the full meaning `Target Release Version` via the standard HTML `title` tooltip. This is the first overview header to carry a `title` attribute; future short-label columns may follow the same pattern.

## Extraction rule

For a Decision Record, `target_release_version` is the value found by the following lookup:

1. Locate the `Software Versions` section by its heading text (case-sensitive, any heading level), using the same locate-by-heading approach already used by `extract_current_status` and `extract_start_date` (the `find_section_table` helper).
2. Within that section, take the first markdown table.
3. In that table, identify the `Software Version Category` column and the `Software Version ID` column by their header text, case-sensitive, not by column position.
4. Find the row whose `Software Version Category` cell, after stripping surrounding whitespace, equals `Target Release Version` exactly.
5. Take the trimmed value of that row's `Software Version ID` cell as the attribute. An empty cell results in `nil`.

If the section is absent, if the table is absent, if either column header is not found, or if no row matches `Target Release Version`, the attribute is `nil`.

The version value is treated as an opaque string. No SemVer parsing, no validation, no normalisation: whatever the author wrote in the `Software Version ID` cell (e.g., `0.4.0`, `n/a`, `TBD`) is what appears in the column.

## Rendering

For each decision row, the new column cell carries `target_release_version` as plain text. When the attribute is `nil`, the cell renders empty.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  |  |  | Done | 22-05-2026 | 22-05-2026 | New SRS items in `srs.md` covering: the `target_release_version` attribute on a Decision Record; the extraction rule (Software Versions section, table lookup by column header, row lookup by exact match on `Target Release Version`); the empty-cell fallback when no value is found; rendering of the attribute in a new `Release` column on the Decision Records Overview; the `title="Target Release Version"` attribute on the column header |
| 2 | Code | DEV |  |  |  | Done | 22-05-2026 | 22-05-2026 | Add a `target_release_version` accessor on `Decision`; add an `extract_target_release_version` method that reuses `find_section_table` to locate the `Software Versions` table and reads the `Software Version ID` cell of the row whose `Software Version Category` is `Target Release Version`; wire the extractor into `DocFabric#create_decision` alongside `extract_current_status` and `extract_start_date`; insert the new `Release` column between `Target Date` and `Owner` in `decisions_overview.rb`, emitting the `<th>` with a `title` attribute and the `<td>` with the attribute value or empty |
| 3 | Tests | TEST |  |  |  | Done | 22-05-2026 | 22-05-2026 | End-to-end tests under `spec/e2e/decisions_spec.rb`: extraction reads the `Target Release Version` row from the `Software Versions` table; extraction works regardless of the column order in the Software Versions table (lookup by header text); extraction returns nil when the section is missing; extraction returns nil when the row is missing; extraction returns nil when the cell is empty; the rendered `Release` column carries the formatted value and is empty when undefined; the column header includes the `title="Target Release Version"` attribute; the column is placed between `Target Date` and `Owner` |

# Out of Scope

- Sorting the Decision Records Overview by `Release`. Row order continues to follow [[adr-170-introduce-decision-records]] / [[enh-175-overview-sort-id]] (sequence number then ID).
- Parsing or validating the version string. Values like `n/a`, `TBD`, `0.4.0-beta`, or anything else the author writes in the `Software Version ID` cell are passed through verbatim. SemVer-aware comparison, sorting, or grouping is left to a future ADR.
- The other rows of the `Software Versions` table (`Latest Released Version`, `Issue Found in Version`, etc.). They are not surfaced on the overview by this ADR.
- A configurable column label or tooltip. The visible text is fixed at `Release` and the tooltip at `Target Release Version`; theming or localisation is out of scope.
- Showing the same value on the rendered Decision Record page itself. The record already contains the `Software Versions` section in its body; this ADR only populates the overview's new column.
- Visualising version distribution in any chart on the overview page (the chart grid reserved by [[adr-177-overview-pie-chart]] remains unrelated to this change).

# Consequences

## Positive

- A reader can scan the overview and see at a glance which release each decision is targeted for, without opening individual records.
- A single source of truth: the version lives in the `Software Versions` table that authors already maintain. No new hand-edited frontmatter field can drift out of sync.
- The new `target_release_version` attribute is available on the Decision instance for any future feature (filtering, release-grouped views, milestone reports) without re-parsing.
- The `title`-attribute pattern on the column header is a small, clean way to keep the overview table compact while preserving the full meaning for users who need it.

## Negative

- The renderer now depends on the **header text** of the `Software Versions` table — renaming `Software Version Category` or `Software Version ID` would silently empty the column. The test suite locks in these as the expected headers; authors are expected to keep them stable.
- The renderer also depends on the exact spelling `Target Release Version` in the category column. A typo (`Target Release Ver.`, lowercase variant) would silently leave the cell empty. The test suite locks in this exact label.

## Neutral

- Decision records authored before this ADR continue to work unchanged: extraction is purely additive and falls back to empty when the section, row, or value is missing.
- No data migration required. Every existing decision record under `Almirah.Doc/decisions/` already carries the `Software Versions` section in the expected format.

# Alternatives Considered

- **Author the target release version as a frontmatter field (e.g., `target_release_version: 0.4.0`).** Rejected: duplicates information already present in the `Software Versions` table and creates a second place that can drift. Decision records already carry the Software Versions section by convention; deriving from it keeps maintenance cost zero.
- **Expose all three rows of the Software Versions table as separate columns (`Latest Released`, `Issue Found In`, `Target Release`).** Rejected as overreach for this ADR. The overview is meant to stay compact; adding three version columns would crowd out the existing ones. A future ADR can broaden the surface area if needed.
- **Use a long header text such as `Target Release Version` directly on the column.** Rejected: the overview already has seven columns and adding an eighth with a long header would force wrapping or horizontal scrolling. The short `Release` label with a `title` tooltip preserves the meaning without taking up width.
- **Look the row up by position (first row, or row index 2 of the body).** Rejected for the same reason ADR-178 rejected position-based column lookup: a future addition of, say, an `Investigation Started Version` row would silently change which row is read. Header-text and row-label lookup is robust to additions.
- **Parse the version string as SemVer to enable sorting and validation.** Rejected: out of scope here. The current authoring convention allows free-form strings (`n/a`, `TBD`); imposing SemVer would force a migration of existing records. SemVer support can be added in a separate enhancement.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall expose a Target Release Version attribute on each Decision Record, computed by locating the Decision Record's Software Versions section, finding the row of its table whose Software Version Category cell equals "Target Release Version", and taking the value of that row's Software Version ID cell. | >[SRS-066] |
| 2 | The software shall identify the Software Version Category column and the Software Version ID column of a Decision Record's Software Versions table by their header text, case-sensitive, and not by column position. | >[SRS-067] |
| 3 | The software shall identify the Target Release Version row of a Decision Record's Software Versions table by an exact, case-sensitive match against its Software Version Category cell after stripping surrounding whitespace. | >[SRS-068] |
| 4 | When a Decision Record's Software Versions section, the table within it, the Software Version Category or Software Version ID column, the Target Release Version row, or the cell value is missing or empty, the Target Release Version attribute of that Decision Record shall be undefined. | >[SRS-069] |
| 5 | The Decision Records Overview page shall include a "Version" column positioned between the "Target Date" and "Owner" columns, displaying each decision record's Target Release Version attribute as plain text. The column header shall carry an HTML title attribute with the value "Target Release Version". The cell shall be empty when the attribute is undefined. | >[SRS-070] |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records, the Software Versions section, and the Decision Records Overview page
- [ADR-172](./adr-172-current-status-marker.md) — established the parse-the-Status-table pattern reused for section-table lookups
- [ADR-174](./adr-174-dr-specification-links.md) — precedent for an ADR that adds software requirements via an "Affected Documents" table
- [ADR-178](./adr-178-overview-start-date.md) — sibling overview-column addition; reuses the same `find_section_table` helper and column-by-header lookup approach
- SRS-039 through SRS-065 in [srs.md](./../../specifications/srs/srs.md) — current requirements covering decision records; new SRS items added under this ADR extend that range

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47)
