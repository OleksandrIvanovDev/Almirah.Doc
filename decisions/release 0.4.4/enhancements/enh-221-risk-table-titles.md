---
title: "ENH-221: Risk Table Titles and IDs"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 08-07-2026 | Proposed |
|   | 08-07-2026 | Accepted |
|   | 08-07-2026 | In-Progress |
| * | 08-07-2026 | Implemented |

# Context

The risk pages shipped by [ADR-216](./../adr-216-risk-register-table.md) and [ADR-219](./../adr-219-risks-menu-page.md) present three rough edges that the Testing Data Set examples made visible:

- The all-registries summary page names each registry by its folder (`security`), although the registry preface already carries a human title in its overview.md frontmatter ("Security Risk Register") — and the registry page itself already uses that title as its page heading.
- The register table's ID column shows the record ID in the canonical lowercase form (`secr-001`), while risk registers conventionally display capitalised IDs (SECR-001).
- The register table's Title column repeats the record's full frontmatter title, "SECR-001: SQL Injection in Search", so the ID appears twice in every row — once in the ID column and again inside the title.

# Decision

Polish the presentation of the two risk tables; no parsing, linking, or file-layout behaviour changes.

- **Registry titles on the summary page.** The Risk Registry cell on the all-registries page shows the registry preface's frontmatter title instead of the folder name, falling back to the folder name when the registry has no preface or the preface has no title. The link target and anchor are unchanged.
- **Capitalised IDs in the register table.** The ID column displays the record ID uppercased (SECR-001). The uppercasing is display-only: anchors, hrefs and the project-wide link space keep the canonical lowercase ID, so existing links keep resolving.
- **ID-free titles in the register table.** The Title column strips the record's own leading ID prefix — the record ID, case-insensitively, followed by a colon — from the frontmatter title, rendering "SQL Injection in Search" instead of "SECR-001: SQL Injection in Search". A title that does not start with the record's own ID renders unchanged, and the record page itself keeps the full frontmatter title.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 0.5 | 1 | Done | 08-07-2026 | 08-07-2026 | Amend SRS-168 (register table: uppercased ID column, ID-free Title column) and SRS-172 (summary page: registry preface title with folder-name fallback) |
| 2 | Code | DEV |  | 1 | 2 | Done | 08-07-2026 | 08-07-2026 | Pass the registry prefaces to the risks overview and render the preface title with fallback; uppercase the displayed ID and strip the record's own leading ID prefix from the title in the register table, keeping anchors and hrefs lowercase |
| 3 | Tests | TEST |  | 1 | 2 | Done | 08-07-2026 | 08-07-2026 | Extend the `Almirah.Code` e2e specs: summary row shows the preface title and falls back to the folder name; register ID cell shows the uppercased ID with the lowercase anchor kept; Title cell drops the ID prefix and leaves a non-prefixed title unchanged |

# Out of Scope

- **The record pages.** A record's own page keeps its full frontmatter title, ID included — the prefix is stripped only where an ID column already carries it.
- **Renaming files or IDs.** Record IDs stay lowercase in filenames, anchors, and links; only the rendered ID cell is capitalised.
- **Sorting or new columns.** Row order and the column set of both tables are untouched.

# Consequences

## Positive

- The summary page reads as a portfolio of named registers rather than a folder listing, reusing the title the registry page already displays.
- Register rows lose the duplicated ID, and the capitalised ID column matches common risk-register practice and the record titles themselves.

## Negative

- The Title cell no longer matches the record's frontmatter title verbatim; anyone text-searching a build for the full "ID: Title" string will find the record page but not the register row.

## Neutral

- Registries without a preface keep their folder name on the summary page, so minimal projects render exactly as before.

# Alternatives Considered

- **Uppercasing the record ID at parse time.** Rejected: the lowercase ID is the canonical key of the project-wide link space; changing it would ripple through anchors, hrefs and duplicate detection for a display concern.
- **Requiring titles without the ID prefix in the record frontmatter.** Rejected: the "ID: Title" frontmatter convention matches decision records and keeps the record page self-identifying; the register table is the only place the prefix is redundant.
- **Showing the preface's first heading instead of its frontmatter title.** Rejected: the frontmatter title is what the registry page itself already uses ([ADR-216](./../adr-216-risk-register-table.md)); reading a different source would let the two pages disagree.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This enhancement amends SRS-168 and SRS-172.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall render each risk registry to an overview page consisting of the registry's overview.md content followed by a register table with one row per risk record, whose leading columns are the record ID, displayed uppercased and linked to the record page, and the record title with the record's own leading ID prefix removed, and whose further columns are configured per registry and filled from each record's section whose heading matches the column name, the Status column being filled from the record's current lifecycle status. | >[SRS-168] |
| 2 | The software shall add a Risks entry to the top menu bar, present only when the project has at least one risk registry, leading to a summary page holding one row per registry with the columns Risk Registry, Total Risks, Open Risks, Highest RPN and Average RPN, where the Risk Registry cell shows the registry preface's frontmatter title, falling back to the registry folder name when no preface title exists, linked to its registry page, the open count excludes records whose current status is Closed, and the RPN aggregates are computed over the registry's leading RPN group ignoring blank values. | >[SRS-172] |

# References

- [ADR-216](./../adr-216-risk-register-table.md) — the register table whose ID and Title columns are polished
- [ADR-219](./../adr-219-risks-menu-page.md) — the summary page whose Risk Registry cell gains the preface title
- [ADR-215](./../adr-215-risk-record-collection.md) — the lowercase canonical ID the display capitalisation must not change

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 08-07-2026 | Requirements | BA | 0.5 | Initial Proposal |
| 08-07-2026 | Requirements | BA | 0.5 | SRS-168 and SRS-172 amended |
| 08-07-2026 | Code | DEV | 1 | Overview titles; register ID/Title display |
| 08-07-2026 | Tests | TEST | 1 | e2e specs extended, full suite green |
