---
title: "ADR-215: Risk Record Collection"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-07-2026 | Proposed |
|   | 07-07-2026 | Analysis |
| * | 07-07-2026 | In-Progress |

# Context

Almirah manages four document collections today: specifications, test protocols, source files, and decision records ([[adr-170-introduce-decision-records]]). Risk management is the register-shaped discipline still missing: on real projects the risk register lives in a spreadsheet outside the traceability graph, so a risk's mitigation cannot point at the requirement that implements it, and the requirement cannot show which risk it exists to control.

The register-style techniques common across domains — a PMBOK project risk register, an FMEA worksheet (IEC 60812), an ISO 14971 hazard analysis, an ISO 27005 security register — share one shape: a set of individually identified risks, each carrying a handful of named attributes, reviewed over a lifecycle and summarised in a table. That shape maps directly onto the machinery decision records already proved: one Markdown file per record, sections located by heading text, a Status table with a current-state marker ([[adr-172-current-status-marker]]), and a per-collection overview page.

This is the foundation decision of the risk-records feature; the register table (ADR-216), the RPN computed columns (ADR-217), the specification links (ADR-218), and the top-menu registry page (ADR-219) build on the collection introduced here.

# Decision

Introduce a third record collection, *risk records*, parsed from a `risks/` folder at the project root, sibling to `decisions/`.

- **One registry per first-level subfolder.** Each first-level subfolder of `risks/` is a *risk registry* grouping one risk type; the folder names are free (typically `project`, `product`, `process`, `security`). Parsing derives everything from the file system — no configuration is required to collect the records.
- **A registry preface.** Each registry folder may carry an `overview.md`: free Markdown describing what risks live there, the scales in use, and any registry-specific rules. It is a preface, not a risk record, and is rendered at the top of the registry page (ADR-216).
- **One file per risk.** Risk record filenames follow the decision-record convention `<letters>-<digits>-<slug>.md` with the slug kept to three words at most (for example `prjr-001-expertise-loss.md`). The record ID is derived from the `<letters>-<digits>` prefix. Each registry uses its own distinct letter prefix so an ID names its registry on sight and stays unique in the project-wide link space; a duplicate ID is reported at build time like any other duplicate controlled ID.
- **The Status lifecycle is reused verbatim.** Each record starts with a `# Status` section holding the decision-record status table: the leading column carries `*` in exactly one row marking the current state. Status texts are free; the suggested lifecycle is Identified, Analysed, Mitigating, Accepted, Closed. The current state feeds the register table (ADR-216) and the open-risk counts (ADR-219).
- **Frontmatter title.** Every record carries a YAML `title:` field, displayed on the record page and in the register table; without it the title falls back to the file name.
- **Standalone record pages.** Each record renders to its own HTML page under `build/risks/<registry>/`, with the navigation pane, exactly as decision records do. All other sections of a record are free-form Markdown; which sections surface in the register table is decided per registry (ADR-216).

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 2 | Done | 05-07-2026 | 07-07-2026 | State SRS-166 and SRS-167: the software shall collect risk records from first-level subfolders of a risks folder and shall derive each record's current status from its Status table marker |
| 2 | Code | DEV |  | 2 | 3 | Done | 07-07-2026 | 07-07-2026 | Add a RiskRecord document type and the risks folder scan mirroring the decisions scan; derive IDs from filenames; report duplicate IDs; treat overview.md as the registry preface; reuse the Status-table current-state extraction; render each record to its own page |
| 3 | Tests | TEST |  | 2 | 3 | Done | 07-07-2026 | 07-07-2026 | Add an `Almirah.Code` e2e building a fixture project with two registries and asserting record pages, IDs, and current statuses, with inline fixtures |

# Out of Scope

- **The register table and its column configuration.** Rendering the registry page from `overview.md` plus the collected records is ADR-216.
- **RPN computation and thresholds.** ADR-217.
- **Affected Documents links from risk records.** ADR-218.
- **The top-menu Risks button and the all-registries page.** ADR-219.
- **Graphical and quantitative techniques.** Bowtie diagrams, fault and event trees, and Monte Carlo aggregation are not register-shaped and are out of the feature's scope entirely.

# Consequences

## Positive

- Risks become first-class, individually addressable documents in the traceability graph, with git history per risk — the audit trail regulated domains (ISO 14971, ISO 27001) expect from a register and spreadsheets cannot give.
- The record format is the decision-record format, so authors learn nothing new: frontmatter title, Status table with a marker, free sections.
- Per-registry letter prefixes keep every risk ID legible and collision-free across registries without any central sequence.

## Negative

- A third scanned collection adds a parse pass and a new document type to maintain.
- Distinct prefixes per registry are a convention the build can only check for collisions, not invent; a project that reuses one prefix across registries gets duplicate-ID errors it must resolve by renaming files.

## Neutral

- The suggested lifecycle statuses are advisory; the engine only reads the marked row's text, as it does for decision records.
- A registry without an `overview.md` is valid; its page (ADR-216) simply starts at the table.

# Alternatives Considered

- **A single flat `risks/` folder with a type column.** Rejected: per-type subfolders give each risk type its own scales, columns (ADR-216), and preface, which is how FMEA, project, and security registers actually differ; a single folder would force one schema onto all of them.
- **One global `risk-NNN` sequence across all registries.** Rejected: `risk-001` in two registries would collide in the link space, and a global sequence hides the risk type that a per-registry prefix makes legible in every cross-reference.
- **Risks as a decision-record subtype under `decisions/`.** Rejected: decisions and risks have different lifecycles, different overview needs (register columns and RPN versus planning columns and Gantt), and mixing them would overload the decisions overview that ADR-193 through ADR-213 built around planning.
- **Frontmatter-only risk attributes.** Rejected for the record body: sections located by heading text keep the record readable as a document and reuse the proven extraction machinery; a compact attributes shorthand may be revisited later if authoring friction warrants it.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This decision adds SRS-166 and SRS-167.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall collect risk records from the first-level subfolders of a risks folder at the project root, each subfolder forming a risk registry, each record identified by the letters-digits prefix of its file name, and shall render each record to its own HTML page. | >[SRS-166] |
| 2 | The software shall derive a risk record's current status from the row of its Status table marked with an asterisk in the leading column, in the same way as for decision records. | >[SRS-167] |

# References

- [[adr-170-introduce-decision-records]] — the record convention (frontmatter title, Status table, folder collection) this collection reuses
- [[adr-172-current-status-marker]] — the current-state marker extraction reused for risk records
- [[adr-197-decision-group-collection]] — the folder-as-group precedent the registries follow
- [ADR-216](./adr-216-risk-register-table.md) — the registry page and register table built on this collection

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 07-07-2026 | Requirements | DEV | 0.25 | Analysis |
| 07-07-2026 | Code | DEV | 0.5 | Implementation |
| 07-07-2026 | Tests | TEST | 0.5 | Verification |
