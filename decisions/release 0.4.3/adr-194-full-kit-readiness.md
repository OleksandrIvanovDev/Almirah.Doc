---
title: "ADR-194: Scope Phase Ordering, Dependencies, and Full-Kit Readiness"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 13-06-2026 | Proposed |
|   | 14-06-2026 | Analysis |
|   | 14-06-2026 | Accepted |
|   | 14-06-2026 | In-Progress |
| * | 17-06-2026 | Implemented |

# Context

This is the second implementation step of the planning/flow roadmap in [gfa.md](./../../specifications/gfa/gfa.md). It follows [[adr-193-owner-wip-heatmap]] (step 1, which established the **Scope row / work-item phase** as the unit of planning and the bounded per-row `Status`) and implements the **Full Kit** rule from *Goldratt's Rules of Flow*: a piece of work should not be started until everything it needs to run to completion without interruption is ready. Starting half-kitted work is one of the main causes of the bad multitasking step 1 made visible — the worker stalls waiting for a missing input and opens a second task to stay busy.

There are **two** kit gates, at two altitudes:

1. **The primary gate is internal to a record** — its phases must run in order. You should not start coding before the analysis/requirements that feed it are finished. This is the gate that matters most day-to-day, and step 1 already gave us what it needs: per-row `Status` on a bounded vocabulary.
2. **The secondary gate is across records** — a work item should not start while the matching activity it depends on, in another record, is not yet `Done`.

Almirah already has the basic ingredients for both.

This step also builds on [[adr-197-decision-group-collection]], which collected the first-level `decisions/` sub-folders into `@project_data.decision_groups` — each folder being a *group of records planned together* (a release). The per-row dependency network this ADR builds is **global** (an edge may cross from one group to another), but the group boundary is recorded on every cross-record edge so the later scheduling step ([[adr-195-critical-chain-buffer]]) can size estimates and buffers **per group**: a dependency that stays inside a group is part of that group's chain, while a dependency that crosses into another group is an external readiness constraint on the dependent group's *start*, not part of its internal buffer. Abstractly:

```
[ Group1: Record1 -> Record2 -> Buffer ]
[ Group2: Record3 (Depends On Group1.Record2) -> Buffer ]
```

Here `Record3` is blocked until `Group1.Record2` is `Done` (the gate this ADR enforces), but Group2's buffer is computed over Group2's own chain only — the cross-group edge gates Group2's start and is excluded from Group2's buffer sizing (deferred to [[adr-195-critical-chain-buffer]]).

# Decision

Add a leading **`#` step column** and a per-row **`Depends On`** column to the Scope table, parsed by a **purpose-built `ScopeTable`** that inherits from `ControlledTable` to add the new linking pattern. Each row becomes a **`WorkItem`** carrying **`predecessors`** and **`successors`** lists that span both intra-record (step-order) and cross-record edges, forming a per-row dependency network. On that network, enforce two non-failing kit gates keyed on bounded per-row `Status` — **phase ordering** within a record (primary) and **activity-type-aligned dependency readiness** across records (secondary) — and surface readiness on the overview.

Dependency resolution is **global**: a `Depends On` reference is resolved against *all* managed Decision Records (via `LinkRegistry`), regardless of which `decision_groups` folder either record lives in. Every cross-record edge additionally records whether it is **in-group or cross-group** (by comparing the two records' first-level `decisions/` folders, the same key [[adr-197-decision-group-collection]] uses). That flag is **inert in this step** — kit readiness counts every predecessor alike — and exists only so [[adr-195-critical-chain-buffer]] can scope buffers per group. There is therefore **no cross-group warning**: a cross-group dependency is a legitimate, fully honoured edge.

Implementation hooks: `doc_parser` swaps `MarkdownTable` for `ScopeTable` when `doc.instance_of?(Decision) && in_section?(doc, 'Scope')`; a new `link_work_items` method runs in `project.rb` after `link_all_decisions`. Because the new `ScopeTable` replaces the plain `MarkdownTable` previously returned for the Scope section, the [[adr-193-owner-wip-heatmap]] readers (`extract_owners`, `in_progress_owner_tally`, `effective_status_on`, and the shared `find_section_table` / `column_index`) are **ported onto `ScopeTable`** — which keeps a header-addressed cell-grid view alongside its `WorkItem` rows — so owner/WIP/velocity extraction is unaffected.

## Phase ordering — the primary, intra-record gate

Add a leading **`#` column** to the `# Scope` table whose integer value is the **step number** of the row, mirroring the established numbered-first-column convention (test-protocol "Test Step"; the `#` column elsewhere). Rows with the **same** step number are concurrent; rows with a **higher** number are downstream. When the `#` column is absent, the intrinsic row order is used (each row its own step, in sequence), so existing records keep working.

The gate is **name-agnostic** — it constrains by number, never by whether a row is called `Analysis`, `Code`, or anything else:

> A row may not be `In-Progress` or `Done` while any **lower-numbered** step in the same record is not `Done`.

A **phase-order violation** is a started row (its `Status` is `In-Progress` or `Done`) that has an unfinished lower-numbered step. This is the within-record full-kit gate: it catches "started Code before Requirements was done."

## Dependency data source and extraction (the secondary gate)

`Depends On` is a **per-row** column (header text, not position): the row it is written in is the **dependent work item**, and its value is **one or more other decision records** it comes after — predecessors, in the link form `>[ADR-193]` (comma-separated). A record **never references its own rows**: order **within** a record comes from the `#` step column, not hand-authored links. The gem fills the intra-record edges internally, so every work item ends up connected (no orphans).

**Activity-type-aligned resolution.** A record reference resolves to a *specific work item* of the prerequisite: the row whose **activity type** — its `Item` (`Analysis`, `Requirements`, `Code`, `Tests`, …) — **matches the dependent row's**, falling back to the prerequisite's nearest *earlier* activity type (by step order) when an exact match is absent. So `Depends On: ADR-1` on ADR-2's `Analysis` row makes **ADR-2.Analysis depend on ADR-1.Analysis** — not on ADR-1's implementation. Alignment is by **activity type** (`WorkItem`), *not* by the worker: the `Owner` (the resource — `BA1`, `BA2`, `DEV`, …) is a **separate** dimension used only for resource levelling, so two analysts give two parallel Analysis slots without changing the dependency graph.

The prerequisite record is found **globally** — `LinkRegistry.find_by_id` over every managed record — so the prerequisite may sit in any `decision_groups` folder. The activity-type match is then performed *within* that resolved record.

Internally each Scope row becomes a **`WorkItem`** with a canonical id `<record>.<step>.<activity>` (e.g. `ADR-194.1.Analysis`). The author writes only the **record-level** `>[adr-1]`; the resolver expands it to the activity-type-aligned row defined above (no step, no activity authored by hand). The project holds a dictionary of every work item, `{ "record.step.activity" => "record" }`, which is the source of record IDs for the `link_work_items` method.

Each `WorkItem` holds **`predecessors`** and **`successors`** lists, each entry mapping a readable `record.activity` label to the resolved `WorkItem`: e.g. `predecessors = [{ "ADR-1.1.Analysis" => WorkItem1 }, { "ADR-1.2.Code" => WorkItem2 }]`. A row's predecessors are its lower-numbered same-record steps **plus** its resolved cross-record work items; successors are the inverse, populated during linking. Unresolved references (no such record, or a target that is not a Decision Record) are reported (see *Reporting*).

Each **cross-record** edge also carries a boolean tag for whether the two records share a `decision_groups` folder (in-group) or not (cross-group). Intra-record edges are in-group by definition. The tag is written during linking and not consumed in this step; it is the hook [[adr-195-critical-chain-buffer]] uses to keep a group's buffer over its own chain while treating a cross-group predecessor as an external start constraint.

This handles **inter-record dependencies** only; specification-item and Testing-Data-Set prerequisites are deferred. The per-row work-item network this produces is exactly what the critical-chain step ([[adr-195-critical-chain-buffer]]) schedules.

## Kit-readiness state

A work item's cross-record predecessor is **satisfied** when that resolved (activity-type-aligned) prerequisite work item is `Done` — read from the bounded per-row `Status`, never the open lifecycle vocabulary at the Decision Record level. This per-activity rule supersedes the earlier whole-record "deliverable `Done`" rule, which over-serialised (see *Alternatives Considered*).

Derive `fully_kitted?` **per row**: a row is **kitted** when every one of its predecessors — its lower-numbered same-record steps **and** its resolved cross-record predecessor work items — is `Done` (trivially kitted when it has none); **blocked** otherwise. Readiness ignores the in-group/cross-group tag: a cross-group predecessor blocks the dependent exactly as an in-group one does (in the running example, `Group2.Record3` is blocked until `Group1.Record2` is `Done`). A record is kitted when all its rows are.

## Reporting (the gates)

After the existing link checks, report — all as **non-failing** console warnings in the `report_broken_links` family:

- **Cross-record kit violation:** a **started** work item (its `Status` is `In-Progress` or `Done`) whose resolved cross-record predecessor is not `Done`; name the record, the row, and the unsatisfied predecessor work item. This applies to in-group and cross-group predecessors alike — a cross-group edge is a real dependency, so starting against an unfinished one is a violation just the same.
- **Phase-order violation:** a started row with an unfinished lower-numbered step; name the record, the row, and the blocking step.
- **Unresolved `Depends On` reference:** an ID resolving to nothing or to a non-decision document; name the referencing record and the bad target. A reference that resolves to a real Decision Record in another group is **not** unresolved — it is a valid cross-group dependency and is never warned on.

The build always completes; these are flow advisories, matching step 1's "warn, don't fail" stance.

## Overview rendering

Add a **`Kit`** column to the Decision Records Overview, reflecting cross-record readiness per record, rendered as **text**: empty when `depends_on` is empty; the word **`Ready`** when `fully_kitted?` and there is at least one prerequisite; the word **`Blocked`** (warning colour) otherwise. A record that is blocked while having a started row (a cross-record violation) is emphasised, matching the console.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-193] |  |  | Done | 14-06-2026 | 17-06-2026 | This decision record: the two-gate model (phase ordering + activity-type-aligned dependency readiness), the purpose-built `ScopeTable` / `WorkItem` network (record-level authoring, internal `<record>.<step>` edges), the activity-type vs. owner separation, **global** dependency resolution with each cross-record edge tagged in-group / cross-group (over [[adr-197-decision-group-collection]]) so [[adr-195-critical-chain-buffer]] can scope buffers per group while cross-group edges still gate readiness, and porting the [[adr-193-owner-wip-heatmap]] Scope readers onto `ScopeTable` |
| 2 | Requirements | BA | >[ADR-193] |  |  | Done | 17-06-2026 | 17-06-2026 | New SRS items SRS-113 through SRS-119 in `srs.md` (the "Planning" chapter from [[adr-193-owner-wip-heatmap]]): the leading `#` step column and row ordering; the phase-order gate and its violation; the per-row `Depends On` column referencing other Decision Records; activity-type-aligned resolution (match the dependent row's `Item`, fallback to the nearest earlier activity type) with the owner kept separate; a work item's predecessors and per-row `fully_kitted?` (predecessor work item `Done`); the cross-record violation; unresolved-reference reporting; and the overview `Kit` column |
| 3 | Code | DEV | >[ADR-193] |  |  | Done | 17-06-2026 | 17-06-2026 | Introduce a purpose-built `ScopeTable` / `WorkItem` class family (modelled on `ControlledTable` but with header-addressed columns and no fixed link-column position, keeping a header-addressed cell-grid view); parse the `# Scope` section into it, port the date/owner/status extraction onto it, and broaden `find_section_table` to recognise it; assign each row a `<record>.<step>` id; read the `#` step column (fall back to row order) for a phase-order check; resolve each per-row `Depends On` record reference **globally** to the activity-type-aligned work item (`LinkRegistry.find_by_id` for the record, `Item` match within it) and populate per-row `predecessors`/`successors` lists spanning intra- and cross-record edges, tagging each cross-record edge in-group / cross-group via the [[adr-197-decision-group-collection]] folder key; add per-row `fully_kitted?` (counting every predecessor, tag-agnostic); add `report_kit_violations` (both gates) and unresolved-reference reporting alongside `report_broken_links`; render the Scope table like the protocol controlled table, with each `Depends On` reference rendered as a deep link to the resolved activity row of the target record (its `#` step anchor, falling back to the record page when the target has no step column) and an unresolved reference as a broken-link span; add the `Kit` column to `decisions_overview.rb` rendering `Ready` / `Blocked` / empty as text |
| 4 | Tests | TEST | >[ADR-193] |  |  | Done | 17-06-2026 | 17-06-2026 | E2E tests under `spec/e2e/decisions_spec.rb`: rows order by `#` (equal numbers concurrent; absent column → row order); a started row with an unfinished lower step is a phase-order violation; `Depends On: ADR-X` on a row resolves to ADR-X's same-`Item` work item (fallback to the nearest earlier activity type); the resolved predecessor must be `Done` for the dependent to be kitted; a dependent's Analysis is satisfied by the prerequisite's Analysis (not its Code), enabling parallelism; distinct owners (`BA1`/`BA2`) do not change the dependency; a started + blocked work item is a cross-record violation; a **cross-group** `Depends On` still creates the edge, blocks the dependent until the predecessor is `Done`, and is tagged cross-group (an in-group edge tagged in-group), with no cross-group warning; an only-truly-unresolved `Depends On` reported; the `Kit` column renders empty / `Ready` / `Blocked`; everything reads row `Status`, never the lifecycle status |

# Out of Scope

- **Specification-item and Testing-Data-Set prerequisites.** Only inter-decision-record dependencies are handled; specs/TDS branches carry no row-level doneness yet.
- **Explicit per-row dependency *targets* and cross-activity dependencies.** A `Depends On` reference resolves to the prerequisite's *same-activity-type* work item; pointing a row at a *different* activity (e.g. an `Analysis` row depending on a prerequisite's `Code`) would need an explicit per-row target (`>[adr-1.3]`), deferred as a power-user escape hatch.
- **Failing the build on any violation.** Both gates warn and (cross-record) emphasise an overview cell; the build still completes.
- **Cycle detection and full topological scheduling** of either the step graph or the dependency graph. This step evaluates immediate readiness and adjacent-step ordering only; transitive scheduling is the critical-chain step's groundwork (step 3).
- **Using the record lifecycle status as a readiness signal.** Per the project decision, readiness reads bounded per-row `Status`; the open lifecycle vocabulary (`Deployment`, `Reopened`, …) is never consulted ([[adr-193-owner-wip-heatmap]], [[adr-172-current-status-marker]]).
- **Estimates, buffers, and the fever chart** (roadmap steps 3–4).
- **Consuming the in-group / cross-group edge tag.** This step only *records* whether each cross-record edge stays within a `decision_groups` folder; scoping estimates and buffers per group — and treating a cross-group predecessor as an external start constraint rather than part of the dependent group's buffer — is [[adr-195-critical-chain-buffer]]'s work. Like [[adr-197-decision-group-collection]]'s collection, the tag arrives inert.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
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

- [gfa.md](./../../specifications/gfa/gfa.md) — the roadmap this ADR is step 2 of; implements the Full Kit rule
- [[adr-193-owner-wip-heatmap]] — step 1; established the Scope row as the planning unit, the bounded per-row `Status`, and the decision that the lifecycle status is not a planning input. It is also this record's own prerequisite
- [[adr-172-current-status-marker]] — the hand-marked, open-vocabulary record lifecycle status, kept independent of the per-row `Status` this gate reads
- [[adr-197-decision-group-collection]] — collected the `decision_groups` folders this ADR tags each cross-record edge against; the per-group buffer boundary those groups define is consumed by [[adr-195-critical-chain-buffer]]
- [[adr-186-cross-document-links]] — the `LinkRegistry` resolution and broken-link reporting this check extends
- [[adr-191-overview-target-date]] — the header-text-not-position, derive-don't-duplicate discipline followed here
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, the overview page, and the numbered "Test Step" column convention this ADR mirrors for Scope steps

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)

