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
2. **The secondary gate is across records** — a work item should not start while the matching activity it depends on, in another record, is not yet `Done`.

Almirah already has the ingredients for both, without any external store:

- **Row order.** `MarkdownTable` preserves row order, and the framework already uses a numbered leading column to identify steps (the test-protocol "Test Step" column; the `#` column on the Affected Documents and overview tables). The same convention gives Scope rows an explicit step number.
- **A linkage mechanism.** The framework already turns a Decision's tables into **controlled tables of work items** and links them like test protocols to specifications (`link_*_to_spec`, [doc_linker.rb](./../../../Almirah.Code/lib/almirah/project/doc_linker.rb)). The same native row-to-row linking represents work-item dependencies — a record id resolves via `LinkRegistry.find_by_id` ([link_registry.rb](./../../../Almirah.Code/lib/almirah/link_registry.rb)), then to a specific work item by activity type.
- **A row-based readiness signal.** Per the project decision that the open record lifecycle status (`Analysis`, `Implemented`, `Deployment`, `Reopened`, …) is **not** a planning input ([[adr-193-owner-wip-heatmap]], [[adr-172-current-status-marker]]), readiness reads the **bounded per-row `Status`**: a dependent work item is ready when the matching-activity predecessor it depends on is `Done`.
- **A reporting channel.** The Check pass already prints unresolved cross-document links as **non-failing console warnings** (`report_broken_links` → `ConsoleReporter.warn`, [project.rb:100](./../../../Almirah.Code/lib/almirah/project.rb#L100)). A kit violation is the same shape of problem and joins the same family.

# Decision

Add a leading **`#` step column** and a per-row **`Depends On`** column to the Scope table — internally a **controlled table of work items** linked row-to-row (like test protocols). Enforce two non-failing kit gates — **phase ordering** within a record (primary) and **activity-type-aligned dependency readiness** across records (secondary) — both keyed on bounded per-row `Status`; and surface readiness on the overview.

## Phase ordering — the primary, intra-record gate

Add a leading **`#` column** to the `# Scope` table whose integer value is the **step number** of the row, mirroring the established numbered-first-column convention (test-protocol "Test Step"; the `#` column elsewhere). Rows with the **same** step number are concurrent; rows with a **higher** number are downstream. When the `#` column is absent, the intrinsic row order is used (each row its own step, in sequence), so existing records keep working.

The gate is **name-agnostic** — it constrains by number, never by whether a row is called `Analysis`, `Code`, or anything else:

> A row may not be `In-Progress` or `Done` while any **lower-numbered** step in the same record is not `Done`.

A **phase-order violation** is a started row (its `Status` is `In-Progress` or `Done`) that has an unfinished lower-numbered step. This is the within-record full-kit gate: it catches "started Code before Requirements was done."

## Dependency data source and extraction (the secondary gate)

`Depends On` is a **per-row** column (header text, not position): the row it is written in is the **dependent work item**, and its value is **one or more other decision records** the dependent comes after — predecessors, in the link form `>[ADR-193]` (comma-separated). A record **never references its own rows**: order **within** a record is taken from row order (the `#` step column), not hand-authored.

**Activity-type-aligned resolution.** A record reference resolves to a *specific work item* of the prerequisite: the row whose **activity type** — its `Item` (`Analysis`, `Requirements`, `Code`, `Tests`, …) — **matches the dependent row's**, falling back to the prerequisite's nearest *earlier* activity type when an exact match is absent. So `Depends On: ADR-1` on ADR-2's `Analysis` row makes **ADR-2.Analysis depend on ADR-1.Analysis** — not on ADR-1's implementation. Alignment is by **activity type** (`Item`), *not* by the worker: the `Owner` (the resource — `BA1`, `BA2`, `DEV`, …) is a **separate** dimension used only for resource levelling, so two analysts give two parallel Analysis slots without changing the dependency graph.

Internally this is the framework's existing controlled-table linkage: each Scope row becomes a work item with an id (`<record>.<step>`), and a resolved dependency is a native row-to-row link. The record id resolves via `LinkRegistry.find_by_id`; the work item within it, by activity type. Unresolved references (no such record) are reported (see *Reporting*). Each row's **predecessors** are therefore its lower-numbered same-record steps **plus** its resolved cross-record work items; a row with neither has no predecessors.

This handles **inter-record dependencies** only; specification-item and Testing-Data-Set prerequisites are deferred. The per-row work-item network this produces is exactly what the critical-chain step ([[adr-195-critical-chain-buffer]]) schedules.

## Kit-readiness state

A work item's cross-record predecessor is **satisfied** when **that resolved (activity-type-aligned) prerequisite work item is `Done`**. This supersedes the earlier whole-record "deliverable (`Code`/`Tests`) `Done`" rule, which over-serialised: it would block a dependent's *analysis* until the prerequisite was fully built, defeating the cross-role parallelism resource levelling should achieve (a dependent's Analysis can proceed once the prerequisite's Analysis is done, in parallel with the prerequisite's Code). It reads only bounded per-row `Status`, never the open lifecycle vocabulary.

Derive `fully_kitted?` **per row**: a row is **kitted** when every one of its predecessors — its lower-numbered same-record steps **and** its resolved cross-record predecessor work items — is `Done` (trivially kitted when it has none); **blocked** otherwise. A record is kitted when all its rows are.

## Reporting (the gates)

After the existing link checks, report — all as **non-failing** console warnings in the `report_broken_links` family:

- **Cross-record kit violation:** a **started** work item (its `Status` is `In-Progress` or `Done`) whose resolved cross-record predecessor is not `Done`; name the record, the row, and the unsatisfied predecessor work item.
- **Phase-order violation:** a started row with an unfinished lower-numbered step; name the record, the row, and the blocking step.
- **Unresolved `Depends On` reference:** an ID resolving to nothing or to a non-decision document; name the referencing record and the bad target.

The build always completes; these are flow advisories, matching step 1's "warn, don't fail" stance.

## Overview rendering

Add a **`Kit`** column to the Decision Records Overview, reflecting cross-record readiness per record: empty when `depends_on` is empty; a "ready" marker when `fully_kitted?` and there is at least one prerequisite; a "blocked" marker (warning colour) otherwise. A record that is blocked while having a started row (a cross-record violation) is emphasised, matching the console.

# Scope

| # | Item | Owner | Depends On | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-193] | In-Progress | 14-06-2026 |  | This decision record: the two-gate model (phase ordering + activity-type-aligned dependency readiness), the controlled-table work-item linkage, and the activity-type vs. owner separation |
| 2 | Requirements | BA | >[ADR-193] | To Do |  |  | New SRS items SRS-113 through SRS-119 in `srs.md` (the "Planning" chapter from [[adr-193-owner-wip-heatmap]]): the leading `#` step column and row ordering; the phase-order gate and its violation; the per-row `Depends On` column referencing other Decision Records; activity-type-aligned resolution (match the dependent row's `Item`, fallback to the nearest earlier activity type) with the owner kept separate; a work item's predecessors and per-row `fully_kitted?` (predecessor work item `Done`); the cross-record violation; unresolved-reference reporting; and the overview `Kit` column |
| 3 | Code | DEV | >[ADR-193] | To Do |  |  | Parse the Scope table as a controlled table of work items (extend the existing `Decision`/`'Affected Documents'` controlled-table path to the `Scope` section), porting the date/owner/status extraction onto it; read the `#` step column (fall back to row order) for a phase-order check; resolve each per-row `Depends On` record reference to the activity-type-aligned predecessor work item (`LinkRegistry.find_by_id` for the record, `Item` match within it) and build the row-to-row links; add per-row `fully_kitted?`; add `report_kit_violations` (both gates) and unresolved-reference reporting alongside `report_broken_links`; add the `Kit` column to `decisions_overview.rb` |
| 4 | Tests | TEST | >[ADR-193] | To Do |  |  | E2E tests under `spec/e2e/decisions_spec.rb`: rows order by `#` (equal numbers concurrent; absent column → row order); a started row with an unfinished lower step is a phase-order violation; `Depends On: ADR-X` on a row resolves to ADR-X's same-`Item` work item (fallback to the nearest earlier activity type); the resolved predecessor must be `Done` for the dependent to be kitted; a dependent's Analysis is satisfied by the prerequisite's Analysis (not its Code), enabling parallelism; distinct owners (`BA1`/`BA2`) do not change the dependency; a started + blocked work item is a cross-record violation; unresolved `Depends On` reported; the `Kit` column renders empty / ready / blocked; everything reads row `Status`, never the lifecycle status |

# Out of Scope

- **Specification-item and Testing-Data-Set prerequisites.** Only inter-decision-record dependencies are handled; specs/TDS branches carry no row-level doneness yet.
- **Explicit per-row dependency *targets* and cross-activity dependencies.** A `Depends On` reference resolves to the prerequisite's *same-activity-type* work item; pointing a row at a *different* activity (e.g. an `Analysis` row depending on a prerequisite's `Code`) would need an explicit per-row target (`>[adr-1.3]`), deferred as a power-user escape hatch.
- **Failing the build on any violation.** Both gates warn and (cross-record) emphasise an overview cell; the build still completes.
- **Cycle detection and full topological scheduling** of either the step graph or the dependency graph. This step evaluates immediate readiness and adjacent-step ordering only; transitive scheduling is the critical-chain step's groundwork (step 3).
- **Using the record lifecycle status as a readiness signal.** Per the project decision, readiness reads bounded per-row `Status`; the open lifecycle vocabulary (`Deployment`, `Reopened`, …) is never consulted ([[adr-193-owner-wip-heatmap]], [[adr-172-current-status-marker]]).
- **Estimates, buffers, and the fever chart** (roadmap steps 3–4).

# Consequences

## Positive

- Implements both kit gates with **no hardcoded phase names** for ordering: one numbered column, one dependency column, two derived checks, one overview column — reusing the framework's existing controlled-table work-item linkage, `LinkRegistry`, and the numbered-column convention.
- The primary (within-record) gate — the one teams hit most — is captured directly: "don't start Code before Requirements is Done" becomes a reported violation.
- Activity-type-aligned resolution is what unlocks resource optimisation: a dependent's analysis waits only for the prerequisite's analysis, free to run in parallel with the prerequisite's code (different activity, different resource). A whole-record "deliverable done" rule would serialise them.
- Reads only bounded per-row `Status`, so adding a lifecycle status (`Deployment`, `Reopened`) never changes the gate logic.
- The per-row work-item network (intra-record step order + cross-record activity-aligned links) is exactly what the critical-chain step (3) schedules and resource-levels by `Owner`.

## Negative

- The renderer/check gain dependencies on the header text `Depends On` and on the `#`/item columns; renaming them changes behaviour. The test suite locks the headers, as for `Owner` ([[adr-193-owner-wip-heatmap]]) and `Target Date` ([[adr-191-overview-target-date]]).
- Readiness is only as accurate as per-row `Status`: a delivery row left un-`Done` keeps dependents blocked even when the work is really finished.
- Activity-type alignment matches the `Item` text across records; renaming an activity in one record but not another misaligns the cross-record dependency (the dependent then falls back to the nearest earlier activity). Consistent activity names across records matter for the secondary gate; the primary, ordering gate stays name-agnostic.

## Neutral

- Records authored before this ADR work unchanged: no `#` column → row order; no `Depends On` → kitted; existing rows keep their statuses.
- When a prerequisite has no matching activity (and no earlier activity) to resolve to — e.g. no Scope rows at all — the dependent's cross-record predecessor is empty and treated as satisfied; the consistent consequence of not consulting lifecycle status, acceptable for purely architectural records.
- Re-rendering is required to gain the columns and checks; nothing is retroactive.
- This record's `Analysis` row depends on [[adr-193-owner-wip-heatmap]] and so resolves (activity-type-aligned) to ADR-193's `Analysis`, which is `In-Progress`, not `Done`. Because this record's `Analysis` is itself started, it is intentionally a **cross-record violation** — a live demonstration of the secondary gate catching concurrent analysis — while its single started step (step 1) has no lower step and raises no phase-order violation.

# Alternatives Considered

- **Authoring per-row dependency targets (`>[adr-194.3]`).** The per-row, protocol-style linkage **is adopted internally** — the Scope table becomes a `ControlledTable` of work items with row IDs (`<record>.<step>`), linked row-to-row like protocols to specs ([doc_linker.rb](./../../../Almirah.Code/lib/almirah/project/doc_linker.rb)). What is **rejected for authoring** is making the user write per-row *targets*: they declare only record-to-record predecessors, and resolution picks the prerequisite work item by activity type. Hand-writing row IDs would be tedious, redundant with row order, and fragile under reordering. (An explicit per-row target stays a deferred escape hatch for cross-activity dependencies.)
- **Order phases by matching row names (`Analysis` → `Requirements` → `Code` → `Tests`).** Rejected: brittle and inflexible — a record may have only some phases, or extra ones, in a different order. A numbered column orders by intent and supports concurrent steps (equal numbers), with no name list to maintain.
- **Whole-record satisfaction (the prerequisite's deliverable, or last step, `Done`).** Rejected: it over-serialises — a dependent's *analysis* would wait for the prerequisite's *implementation*, blocking the cross-role parallelism that resource levelling should find. Activity-type alignment lets each activity wait only for its matching predecessor. (This reverses the "Code/Tests `Done` if exist" rule from an earlier draft of this ADR.)
- **Align dependencies by owner/role rather than activity type.** Rejected: a role may have several people (`BA1`, `BA2`). The dependency is a property of the work (analysis-needs-analysis), not the worker; aligning by activity type keeps the edge stable while extra staff in a role add parallel capacity during resource levelling.
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
| 3 | The Decision Record Scope table shall support a per-row Depends On column, identified by header text and not position, in which the row carrying a reference is the dependent work item; each reference to another Decision Record shall be resolved to that record's work item whose activity type (its Item) matches the dependent row's, falling back to the record's nearest earlier activity type when no exact match exists, and the row's Owner shall not affect this resolution. A Decision Record shall not reference its own rows. | >[SRS-115] |
| 4 | The software shall define a Scope row's predecessors as its lower-numbered same-record steps together with its resolved cross-record work items, and shall consider the row fully kitted when every predecessor's Status is Done, including when it has no predecessors. | >[SRS-116] |
| 5 | The software shall report, without failing the build, each started Scope row (Status In-Progress or Done) whose resolved cross-record predecessor work item is not Done, naming the record, the row, and the unsatisfied predecessor. | >[SRS-117] |
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
