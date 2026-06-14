---
title: "ADR-194: Scope Phase Ordering, Dependencies, and Full-Kit Readiness"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 13-06-2026 | Proposed |
| * | 14-06-2026 | Analysis |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

This is the second implementation step of the planning/flow roadmap in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md). It follows [[adr-193-owner-wip-heatmap]] (step 1, which established the **Scope row / work-item phase** as the unit of planning and the bounded per-row `Status`) and implements the **Full Kit** rule from *Goldratt's Rules of Flow*: a piece of work should not be started until everything it needs to run to completion without interruption is ready. Starting half-kitted work is one of the main causes of the bad multitasking step 1 made visible — the worker stalls waiting for a missing input and opens a second task to stay busy.

There are **two** kit gates, at two altitudes:

1. **The primary gate is internal to a record** — its phases must run in order. You should not start coding before the analysis/requirements that feed it are finished. This is the gate that matters most day-to-day, and step 1 already gave us what it needs: per-row `Status` on a bounded vocabulary.
2. **The secondary gate is across records** — a record should not be worked while another record it depends on has not yet produced its deliverable.

Almirah already has the ingredients for both, without any external store:

- **Row order.** `MarkdownTable` preserves row order, and the framework already uses a numbered leading column to identify steps (the test-protocol "Test Step" column; the `#` column on the Affected Documents and overview tables). The same convention gives Scope rows an explicit step number.
- **A dependency vocabulary.** Records reference each other by ID, and `LinkRegistry.find_by_id` ([link_registry.rb](./../../../Almirah.Code/lib/almirah/link_registry.rb)) resolves any such ID to its managed document — so "this depends on that" needs no new notation.
- **A row-based readiness signal.** Per the project decision that the open record lifecycle status (`Analysis`, `Implemented`, `Deployment`, `Reopened`, …) is **not** a planning input ([[adr-193-owner-wip-heatmap]], [[adr-172-current-status-marker]]), readiness is read from the **bounded per-row `Status`** instead — a prerequisite's deliverable is ready when its delivery rows are `Done`.
- **A reporting channel.** The Check pass already prints unresolved cross-document links as **non-failing console warnings** (`report_broken_links` → `ConsoleReporter.warn`, [project.rb:100](./../../../Almirah.Code/lib/almirah/project.rb#L100)). A kit violation is the same shape of problem and joins the same family.

# Decision

Add a leading **`#` step column** and a **`Depends On`** column to the Scope table; enforce two non-failing kit gates — **phase ordering** within a record (primary) and **dependency readiness** across records (secondary) — both keyed on bounded per-row `Status`; and surface readiness on the overview.

## Phase ordering — the primary, intra-record gate

Add a leading **`#` column** to the `# Scope` table whose integer value is the **step number** of the row, mirroring the established numbered-first-column convention (test-protocol "Test Step"; the `#` column elsewhere). Rows with the **same** step number are concurrent; rows with a **higher** number are downstream. When the `#` column is absent, the intrinsic row order is used (each row its own step, in sequence), so existing records keep working.

The gate is **name-agnostic** — it constrains by number, never by whether a row is called `Analysis`, `Code`, or anything else:

> A row may not be `In-Progress` or `Done` while any **lower-numbered** step in the same record is not `Done`.

A **phase-order violation** is a started row (its `Status` is `In-Progress` or `Done`) that has an unfinished lower-numbered step. This is the within-record full-kit gate: it catches "started Code before Requirements was done."

## Dependency data source and extraction (the secondary gate)

Add an optional **`Depends On` column** to the Scope table, identified by header text (case-sensitive), not position. Cells hold record references in the existing link form, e.g. `>[ADR-193]` (comma-separated). Derive a record-level `depends_on` attribute: the **distinct, first-seen-ordered** list of references (aggregated across the record's rows) that resolve, via `LinkRegistry.find_by_id`, to a **Decision Record**. References that resolve to nothing or to a non-decision document are excluded (and reported — see *Reporting*). With no Scope table, no `Depends On` column, or an empty column, `depends_on` is empty.

This step handles **inter-record dependencies** only; specification-item and Testing-Data-Set prerequisites are deferred (specs carry no row-level doneness).

## Kit-readiness state

A prerequisite record **P** is **satisfied** — its deliverable is ready for a dependent to consume — by a **row-based, deliverable-targeted** rule:

- if **P** has any row whose item is `Code` or `Tests` (case-insensitive), then **all such rows that exist are `Done`** — "Code/Tests `Done` if they exist", so a record with only a `Code` row, or only a `Tests` row, is judged on whatever it has;
- otherwise (**P** has neither a `Code` nor a `Tests` row), **all of P's rows are `Done`**.

This deliberately targets the **deliverable** (implementation + verification) rather than the record's last step, so a later phase such as `Deployment` does not hold up a consumer that only needs the built-and-tested output. It reads only bounded per-row `Status`, never the open lifecycle vocabulary.

Derive `fully_kitted?` on each `Decision`: **kitted** when every record in `depends_on` is satisfied (including the trivial no-dependencies case); **blocked** otherwise. `fully_kitted?` describes inputs only and is independent of the record's own progress.

## Reporting (the gates)

After the existing link checks, report — all as **non-failing** console warnings in the `report_broken_links` family:

- **Cross-record kit violation:** a record with at least one **started** row (any row `In-Progress` or `Done`) while `fully_kitted?` is false; name the record and each unsatisfied prerequisite.
- **Phase-order violation:** a started row with an unfinished lower-numbered step; name the record, the row, and the blocking step.
- **Unresolved `Depends On` reference:** an ID resolving to nothing or to a non-decision document; name the referencing record and the bad target.

The build always completes; these are flow advisories, matching step 1's "warn, don't fail" stance.

## Overview rendering

Add a **`Kit`** column to the Decision Records Overview, reflecting cross-record readiness per record: empty when `depends_on` is empty; a "ready" marker when `fully_kitted?` and there is at least one prerequisite; a "blocked" marker (warning colour) otherwise. A record that is blocked while having a started row (a cross-record violation) is emphasised, matching the console.

# Scope

| # | Item | Owner | Depends On | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-193] | In-Progress | 14-06-2026 |  | This decision record: the two-gate model (phase ordering + dependency readiness), the step-column convention, and the row-based "Code/Tests Done if exist" satisfaction rule |
| 2 | Requirements | BA | >[ADR-193] | To Do |  |  | New SRS items SRS-113 through SRS-119 in `srs.md` (the "Planning" chapter from [[adr-193-owner-wip-heatmap]]): the leading `#` step column and row ordering; the phase-order gate and its violation; the `Depends On` column and `depends_on` attribute (resolved to Decision Records, header text not position, empty fallback); the row-based satisfaction rule (Code/Tests `Done` if exist, else all rows `Done`) and `fully_kitted?`; the cross-record violation; unresolved-reference reporting; and the overview `Kit` column |
| 3 | Code | DEV | >[ADR-193] | To Do |  |  | Read the `#` step column (fall back to row order) and add a phase-order check on `Decision`; add `depends_on` and a row-based `satisfied?`/`fully_kitted?` (matching `Code`/`Tests` item rows, else all rows) reusing `find_section_table` / `column_index` / `LinkRegistry.find_by_id`, from the `DocFabric.create_decision` path; add `report_kit_violations` (both gates) and unresolved-reference reporting to the Check pass alongside `report_broken_links`; add the `Kit` column to `decisions_overview.rb` |
| 4 | Tests | TEST | >[ADR-193] | To Do |  |  | E2E tests under `spec/e2e/decisions_spec.rb`: rows order by `#` (equal numbers concurrent; absent column → row order); a started row with an unfinished lower step is a phase-order violation; satisfaction requires all existing `Code`/`Tests` rows `Done`, judges a record with only one of them on that one, and falls back to all-rows-`Done` when neither exists; `fully_kitted?` over `depends_on`; a started-row + blocked record is a cross-record violation; unresolved `Depends On` reported; the `Kit` column renders empty / ready / blocked; everything reads row `Status`, never the lifecycle status |

# Out of Scope

- **Specification-item and Testing-Data-Set prerequisites.** Only inter-decision-record dependencies are handled; specs/TDS branches carry no row-level doneness yet.
- **Per-row cross-record dependencies in the gate.** `Depends On` is authored per row but the secondary gate is evaluated at the **record** level (a record's started rows vs. its aggregated prerequisites). Gating an individual downstream row on a specific prerequisite row is deferred.
- **Failing the build on any violation.** Both gates warn and (cross-record) emphasise an overview cell; the build still completes.
- **Cycle detection and full topological scheduling** of either the step graph or the dependency graph. This step evaluates immediate readiness and adjacent-step ordering only; transitive scheduling is the critical-chain step's groundwork (step 3).
- **Using the record lifecycle status as a readiness signal.** Per the project decision, readiness reads bounded per-row `Status`; the open lifecycle vocabulary (`Deployment`, `Reopened`, …) is never consulted ([[adr-193-owner-wip-heatmap]], [[adr-172-current-status-marker]]).
- **Estimates, buffers, and the fever chart** (roadmap steps 3–4).

# Consequences

## Positive

- Implements both kit gates with a small footprint and **no hardcoded phase names** for ordering: one numbered column, one dependency column, two derived checks, one overview column — reusing the existing link syntax, `LinkRegistry`, and the numbered-column convention.
- The primary (within-record) gate — the one teams hit most — is captured directly: "don't start Code before Requirements is Done" becomes a reported violation.
- The deliverable-targeted satisfaction rule ("Code/Tests `Done` if exist") is robust to records that have only some phases, and to later phases like `Deployment` that should not block consumers.
- Reads only bounded per-row `Status`, so adding a lifecycle status (`Deployment`, `Reopened`) never changes the gate logic.
- The `depends_on` graph and the step ordering are exactly the network the critical-chain step (3) consumes.

## Negative

- The renderer/check gain dependencies on the header text `Depends On` and on the `#`/item columns; renaming them changes behaviour. The test suite locks the headers, as for `Owner` ([[adr-193-owner-wip-heatmap]]) and `Target Date` ([[adr-191-overview-target-date]]).
- Readiness is only as accurate as per-row `Status`: a delivery row left un-`Done` keeps dependents blocked even when the work is really finished.
- The satisfaction rule does match the item names `Code`/`Tests` (only the *secondary* gate; the primary gate is name-agnostic). Renaming those rows changes which rows are treated as the deliverable.

## Neutral

- Records authored before this ADR work unchanged: no `#` column → row order; no `Depends On` → kitted; existing rows keep their statuses.
- A prerequisite with **no Scope rows at all** is vacuously satisfied (there is no tracked work to wait on); this is the consistent consequence of not consulting lifecycle status, and is acceptable for purely architectural records.
- Re-rendering is required to gain the columns and checks; nothing is retroactive.
- This record's own rows depend on [[adr-193-owner-wip-heatmap]], whose `Code`/`Tests` rows are not `Done`, so this record is intentionally **not fully kitted** — a live demonstration of the secondary gate, while its single started step (`Analysis`, step 1) has no lower step and so raises no phase-order violation.

# Alternatives Considered

- **Order phases by matching row names (`Analysis` → `Requirements` → `Code` → `Tests`).** Rejected: brittle and inflexible — a record may have only some phases, or extra ones, in a different order. A numbered column orders by intent and supports concurrent steps (equal numbers), with no name list to maintain.
- **Define cross-record satisfaction as "the prerequisite's last step is `Done`".** Rejected: a trailing phase such as `Deployment` would then block a consumer that only needs the built-and-tested deliverable. Targeting `Code`/`Tests` (with the "if exists / else all rows" fallback) matches what a dependent actually consumes.
- **Key readiness on the record lifecycle status (`current_status == Implemented`).** Rejected per the project decision: the lifecycle vocabulary is open and not a planning input; row `Status` is the bounded, authoritative work signal.
- **A record-level `depends_on:` frontmatter list.** Rejected: a second place to maintain relationships the Scope table already carries, prone to drift, as for frontmatter dates in [[adr-191-overview-target-date]] and a frontmatter owner in [[adr-193-owner-wip-heatmap]].
- **Failing the build on a violation.** Rejected for this step, consistent with step 1: make the problem visible before enforcing it.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support a leading step-number column that establishes the order of work-item rows, where rows sharing a step number are concurrent and, when the column is absent, the intrinsic row order applies. | >[SRS-113] |
| 2 | The software shall report, without failing the build, each Scope row that is In-Progress or Done while any lower-numbered step in the same Decision Record is not Done, naming the record, the row, and the blocking step. | >[SRS-114] |
| 3 | The Decision Record Scope table shall support a Depends On column of cross-references to other Decision Records, from which the software shall derive a distinct, first-seen-ordered Dependencies attribute resolving to Decision Records; the column shall be identified by header text, not position, and shall yield an empty attribute when absent or empty. | >[SRS-115] |
| 4 | The software shall consider a prerequisite Decision Record satisfied when all of its existing Code and Tests rows are Done, or — if it has neither a Code nor a Tests row — when all of its rows are Done; and shall consider a Decision Record fully kitted when every record in its Dependencies attribute is satisfied, including when it has none. | >[SRS-116] |
| 5 | The software shall report, without failing the build, each Decision Record that has at least one started row (In-Progress or Done) while it is not fully kitted, naming the record and each unsatisfied prerequisite. | >[SRS-117] |
| 6 | The software shall report, without failing the build, each Depends On reference that does not resolve to a managed Decision Record, naming the referencing Decision Record and the unresolved reference. | >[SRS-118] |
| 7 | The Decision Records Overview page shall render a Kit column indicating, for each Decision Record, whether it has no declared prerequisites, is fully kitted, or is blocked by an unsatisfied prerequisite. | >[SRS-119] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 2 of; implements the Full Kit rule
- [[adr-193-owner-wip-heatmap]] — step 1; established the Scope row as the planning unit, the bounded per-row `Status`, and the decision that the lifecycle status is not a planning input. It is also this record's own prerequisite
- [[adr-172-current-status-marker]] — the hand-marked, open-vocabulary record lifecycle status, kept independent of the per-row `Status` this gate reads
- [[adr-186-cross-document-links]] — the `LinkRegistry` resolution and broken-link reporting this check extends
- [[adr-191-overview-target-date]] — the header-text-not-position, derive-don't-duplicate discipline followed here
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, the overview page, and the numbered "Test Step" column convention this ADR mirrors for Scope steps

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
