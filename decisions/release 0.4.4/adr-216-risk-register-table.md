---
title: "ADR-216: Risk Register Table"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-07-2026 | Proposed |
|   | 07-07-2026 | Analysis |
|   | 07-07-2026 | In-Progress |
| * | 08-07-2026 | Implemented |

# Context

With risk records collected per registry ([ADR-215](./adr-215-risk-record-collection.md)), each registry needs its register view: the one-row-per-risk table that PMBOK registers, FMEA worksheets, and ISO 27005 registers are read through. The columns legitimately differ per risk type — a project register wants Probability, Impact, and Response; an FMEA wants Severity, Occurrence, Detection, and Mitigation; a security register wants Threat, Vulnerability, and a score — so the engine must not hard-code a schema.

Decision records already prove the extraction pattern: a section is located by its heading text and a table column is addressed by its header name, not position ([[adr-172-current-status-marker]]). The same pattern, inverted, turns a record's sections into a register row.

# Decision

Render each registry to a page assembled from its preface and a configurable register table.

- **The registry page.** Each registry renders to `build/risks/<registry>/overview.html`: the rendered content of the registry's `overview.md` first, followed by the register table of every risk record in the registry.
- **Columns configured per registry.** A new `risks:` root in `project.yml` carries one entry per registry folder with the ordered list of register columns:

```yaml
risks:
  - folder: product
    columns: [Severity, Occurrence, Detection, Mitigation, Status]
  - folder: project
    columns: [Probability, Impact, Response, Status]
```

- **Sections become cells.** For each configured column, a record's cell is the rendered content of the record section whose heading text equals the column name. A record may carry more sections than the registry surfaces — analysis prose stays on the record page; only configured columns appear in the table. A record missing a configured section gets an empty cell.
- **Implicit leading columns.** Every register table begins with the record ID (linked to the record page) and the frontmatter Title, before the configured columns; they are not listed in `columns:`.
- **Status is read from the lifecycle.** A configured column named `Status` is filled from the record's current Status-table state ([ADR-215](./adr-215-risk-record-collection.md)), not from a section, so the lifecycle is written once.
- **An unconfigured registry still renders.** A registry folder with no `risks:` entry gets the implicit columns plus Status only — the collection stays visible before it is configured.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA | >[ADR-215] | 1 | 2 | Done | 07-07-2026 | 08-07-2026 | State SRS-168: each risk registry shall render an overview page holding the registry preface followed by a register table whose columns are configured per registry and filled from the records' sections matched by heading text |
| 2 | Code | DEV | >[ADR-215] | 3 | 5 | Done | 07-07-2026 | 08-07-2026 | Read the risks root from project.yml; render the registry page from overview.md plus the register table; fill cells from sections matched by heading, Status from the lifecycle marker; link the ID column to the record pages; default to implicit columns for unconfigured registries |
| 3 | Tests | TEST |  | 1 | 2 | Done | 07-07-2026 | 08-07-2026 | Add an `Almirah.Code` e2e asserting the registry page order, the configured columns, empty cells for missing sections, and the unconfigured-registry default, with inline fixtures |

# Out of Scope

- **Computed columns.** RPN groups and their thresholds are ADR-217; this table renders only authored sections.
- **Linked-ID rendering of Affected Documents cells.** ADR-218.
- **Sorting, filtering, or pagination** of the register table in the browser. The table renders in file-system collection order, like the decisions overview before its charts.
- **Cell-length policing.** Long prose in a surfaced section renders as written; keeping register-surfaced sections short is an authoring convention for the registry preface to state.

# Consequences

## Positive

- Each risk type gets exactly the register its discipline expects, from one mechanism — the column list is the whole schema.
- Records stay documents: everything the table does not surface remains readable on the record page, so the register never truncates the analysis.
- The section-equals-column rule is the existing heading-text convention authors already know from Status, Scope, and Effort tables.

## Negative

- Section-per-column is verbose for short attributes: a record with ten surfaced columns needs ten headings, several holding a single number. A compact attributes shorthand is deliberately deferred until real authoring friction is observed.
- A typo in a section heading silently yields an empty cell; the register makes the gap visible but the build does not flag it.

## Neutral

- Column order in the table is the configured order; two registries may surface the same section name at different positions.
- The `Status` column name is reserved by the lifecycle rule; a registry wanting a prose status must pick another heading.

# Alternatives Considered

- **A hard-coded register schema.** Rejected: no single column set serves FMEA, project, process, and security registers at once; the per-registry list is the smallest configuration that covers them all.
- **Declaring columns in each registry's `overview.md` frontmatter.** Rejected: `project.yml` already owns per-collection configuration (specifications, repositories, planning), and the RPN groups (ADR-217) need the same home; splitting the schema across files would scatter it.
- **Frontmatter attributes instead of sections as the cell source.** Rejected here for the same reason as in ADR-215: sections keep the record a readable document and reuse the heading-text extraction; frontmatter would duplicate every surfaced value out of the visible body.
- **Flagging missing sections as build warnings.** Deferred: a fresh registry would drown the console before its records are filled in; the empty cell is the honest signal meanwhile.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This decision adds SRS-168.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall render each risk registry to an overview page consisting of the registry's overview.md content followed by a register table with one row per risk record, whose leading columns are the linked record ID and title and whose further columns are configured per registry and filled from each record's section whose heading matches the column name, the Status column being filled from the record's current lifecycle status. | >[SRS-168] |

# References

- [ADR-215](./adr-215-risk-record-collection.md) — the record collection and lifecycle this table renders
- [ADR-217](./adr-217-rpn-computed-columns.md) — the computed columns appended to this table
- [[adr-172-current-status-marker]] — the heading-text and header-name addressing convention inverted here
- [[enh-173-decisions-table-view]] — the overview-as-table precedent for a record collection

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 07-07-2026 | Requirements | DEV | 0.25 | Analysis |
