---
title: "ADR-194: Scope Dependencies and Full-Kit Readiness Check"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 13-06-2026 | Proposed |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

This is the second implementation step of the planning/flow roadmap in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md). It follows [[adr-193-owner-wip-heatmap]] (step 1, the Work-In-Progress signal) and implements the **Full Kit** rule from *Goldratt's Rules of Flow*: a piece of work should not be started until everything it needs to run to completion without interruption is ready. Starting half-kitted work is one of the main causes of the bad multitasking that step 1 made visible — the worker stalls waiting for a missing input and opens a second task to stay busy.

Almirah already has the two ingredients needed to detect a kit violation without any external store:

- **A dependency vocabulary.** Decision records and specifications already reference each other by ID, and `LinkRegistry.find_by_id` ([link_registry.rb](./../../../Almirah.Code/lib/almirah/link_registry.rb)) resolves any such ID to its managed document. A "this depends on that" relationship is expressible with the link syntax already in use, with no new notation.
- **A readiness signal.** Every Decision Record carries a derived `current_status` (the `*`-marked Status row, [[adr-172-current-status-marker]]). The terminal state of a record is `Implemented`. So "is this prerequisite ready?" reduces to "is its `current_status` Implemented?".
- **A reporting channel.** The build's Check pass already collects unresolved cross-document links and prints them as **non-failing console warnings** at the end of the run (`report_broken_links` → `ConsoleReporter.warn`, [project.rb:100](./../../../Almirah.Code/lib/almirah/project.rb#L100)). A kit violation is the same shape of problem and belongs in the same family of checks.

What is missing is (a) a way to *declare* that a record depends on other records, and (b) a check that flags a record which has been **started while a prerequisite is not yet done**. This ADR adds both, at the Decision-Record level, reusing the derive-from-the-Scope-table discipline of [[adr-191-overview-target-date]] and [[adr-193-owner-wip-heatmap]].

# Decision

Introduce a **`Depends On`** column on the Scope table, derive a record-level dependency list and a **kit-readiness** state from it, report started-but-unkitted records as a build warning, and surface readiness on the overview.

## Dependency data source and extraction

Add an optional **`Depends On` column to the `# Scope` table**, identified by header text (case-sensitive, `Depends On`), not by position — consistent with every other Scope column.

Cells contain decision-record references in the existing link form, e.g. `>[ADR-193]` (one or more, comma-separated). Compute a `depends_on` attribute on each `Decision` during parsing: the **distinct, first-seen-ordered** list of references found in the `Depends On` column that **resolve, via `LinkRegistry.find_by_id`, to a Decision Record**. References that do not resolve, or that resolve to a non-decision document, are not added to `depends_on` (see *Reporting* below). When the Scope table is absent, has no `Depends On` column, or the column is empty, `depends_on` is the empty list.

This step is deliberately scoped to **inter-record dependencies**. Treating a specification item or a Testing-Data-Set branch as a "prerequisite" requires a doneness notion that specs do not currently carry, and is deferred.

## Kit-readiness state

A prerequisite is **satisfied** when the depended-on Decision Record's `current_status` is `Implemented`. Derive a `fully_kitted?` predicate on each `Decision`:

- **kitted** when every record in `depends_on` is satisfied, **including the trivial case of no dependencies**;
- **blocked** when at least one record in `depends_on` is not satisfied.

`fully_kitted?` is independent of the record's own status — it only describes whether its inputs are ready.

## Reporting (the full-kit gate)

A **kit violation** is a record whose own `current_status` is `In-Progress` while `fully_kitted?` is false — i.e. work was started before its inputs were ready. After the existing link checks, report each kit violation as a non-failing console warning in the `report_broken_links` family, naming the record and each unsatisfied prerequisite with its current status. The build still completes; this is a flow advisory, not a hard error, matching step 1's decision not to fail the build on a WIP overrun.

Additionally, an **unresolved `Depends On` reference** (an ID that resolves to nothing, or to a non-decision document) is reported exactly like an unresolved cross-document link, naming the referencing record and the bad target.

## Overview rendering

Add a **`Kit`** column to the Decision Records Overview table, rendered per row as:

- empty when `depends_on` is empty (no declared prerequisites);
- a "ready" marker when `fully_kitted?` is true and there is at least one prerequisite;
- a "blocked" marker, in the palette's warning colour, when `fully_kitted?` is false. A row that is both blocked and `In-Progress` (a kit violation) is emphasised so the overview shows the same violations the console warns about.

# Scope

| Item | Owner | Depends On | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|
| Requirements | A.I. | >[ADR-193] | To Do |  |  | New SRS items SRS-113 through SRS-119 in `srs.md`, in the "Planning" chapter introduced by [[adr-193-owner-wip-heatmap]], covering: the optional `Depends On` column on the Scope table; the `depends_on` attribute (distinct, first-seen-ordered, resolved to Decision Records via the link registry, read by header text); the empty-list fallback; the satisfied/kitted/blocked definitions keyed on the prerequisite's `Implemented` status; the kit-violation warning (In-Progress while not fully kitted); reporting of unresolved `Depends On` references; and the overview `Kit` column |
| Code | A.I. | >[ADR-193] | To Do |  |  | Add a `depends_on` accessor and `fully_kitted?` predicate on `Decision`; add `extract_dependencies` reusing `find_section_table` / `column_index` and `LinkRegistry.find_by_id`, called from the same `DocFabric.create_decision` path as the other extractors; add a `report_kit_violations` step to the Check pass alongside `report_broken_links`; report unresolved `Depends On` targets through the same warning channel; add the `Kit` column to `decisions_overview.rb` |
| Tests | A.I. | >[ADR-193] | To Do |  |  | End-to-end tests under `spec/e2e/decisions_spec.rb`: dependencies are extracted distinct and in first-seen order; only references resolving to Decision Records are kept; an absent/empty column yields no dependencies and a kitted record; a record is blocked when any prerequisite is not Implemented and kitted when all are; a kit violation (In-Progress + blocked) is reported as a non-failing warning naming the unmet prerequisites; an unresolved `Depends On` target is reported like a broken link; the `Kit` column renders empty / ready / blocked correctly |

# Out of Scope

- **Specification-item and Testing-Data-Set prerequisites.** A doneness notion for specs/TDS branches does not yet exist; only inter-decision-record dependencies are handled in this step.
- **Per-Scope-item dependencies and per-item readiness.** As with WIP in [[adr-193-owner-wip-heatmap]], this step operates at the Decision-Record level; finer per-row dependency tracking is deferred.
- **Failing the build on a kit violation.** Violations are surfaced as console warnings and an emphasised overview cell; the build still completes.
- **A configurable set of terminal/"done" statuses.** `Implemented` is the only status treated as satisfying a prerequisite; making this configurable is deferred.
- **Cycle detection and topological ordering / scheduling** of the dependency graph. This step only evaluates immediate prerequisite readiness, not transitive ordering — that is groundwork the critical-chain step (roadmap step 3) will build on.
- **Estimates, buffers, and the fever chart** (roadmap steps 3–4).

# Consequences

## Positive

- Implements the Full Kit rule with the smallest footprint: one optional column, one derived attribute, one predicate, one check, one overview column — and no new notation, reusing the existing link syntax and `LinkRegistry`.
- Half-kitted starts — a leading cause of the multitasking step 1 exposed — become visible both on the console and on the overview, closing the loop between the two steps.
- Single source of truth preserved: dependencies live in the Scope table; readiness is derived from the prerequisite's own Status table; nothing is duplicated, consistent with [[adr-178-overview-start-date]] / [[adr-191-overview-target-date]].
- The `depends_on` graph is reusable by the critical-chain step, which needs exactly this dependency network.

## Negative

- The renderer and check gain a dependency on the header text `Depends On`; renaming it silently disables dependency extraction. The test suite locks in the header, as for `Owner` ([[adr-193-owner-wip-heatmap]]) and `Target Date` ([[adr-191-overview-target-date]]).
- Readiness is only as accurate as prerequisite `current_status` markers: a prerequisite left un-promoted to `Implemented` will keep dependents showing as blocked even after its work is really done.
- The overview table gains a column, marginally widening it.

## Neutral

- Records authored before this ADR continue to work unchanged: extraction is additive and falls back to no dependencies and a kitted state.
- Re-rendering is required to gain the column and the check output; nothing is applied retroactively.
- Because a prerequisite is "satisfied" only at `Implemented`, this ADR's own Scope items depend on [[adr-193-owner-wip-heatmap]], which is still `Proposed`; this record is therefore intentionally **not yet fully kitted** — a live demonstration of the rule it introduces.

# Alternatives Considered

- **A record-level `depends_on:` frontmatter list.** Rejected: a second place to maintain relationships the Scope table can already carry per work item, and prone to drift — the same reasoning that rejected frontmatter dates in [[adr-191-overview-target-date]] and a frontmatter owner in [[adr-193-owner-wip-heatmap]].
- **Failing the build on a kit violation.** Rejected for this step, consistent with step 1: make the problem visible before enforcing it; a hard gate is premature before the team has lived with the signal.
- **Inferring dependencies from existing spec uplinks (`>[SRS-…]`) on the affected requirements.** Rejected: uplinks express requirement traceability, not work-readiness ordering, and conflating the two would make every record depend on its parent specs regardless of whether they gate the work.
- **Treating any non-Proposed prerequisite status as "ready".** Rejected: only `Implemented` means the input is actually finished; `In-Progress` or `Accepted` inputs are precisely the not-yet-ready kit the rule warns against.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support an optional Depends On column declaring, in cross-reference form, the other Decision Records a work item depends on. | >[SRS-113] |
| 2 | The software shall expose a Dependencies attribute on each Decision Record, computed as the distinct, first-seen-ordered list of references in the Depends On column of the Scope table that resolve to a Decision Record. The Depends On column shall be identified by its header text, case-sensitive, and not by column position. | >[SRS-114] |
| 3 | When a Decision Record has no Scope table, no Depends On column, or an empty Depends On column, the Dependencies attribute of that Decision Record shall be the empty list. | >[SRS-115] |
| 4 | The software shall consider a prerequisite Decision Record satisfied when its current status is Implemented, and shall consider a Decision Record fully kitted when every record in its Dependencies attribute is satisfied, including when it has no dependencies. | >[SRS-116] |
| 5 | The software shall report, without failing the build, each Decision Record whose current status is In-Progress while it is not fully kitted, naming the record and each unsatisfied prerequisite together with that prerequisite's current status. | >[SRS-117] |
| 6 | The software shall report, without failing the build, each Depends On reference that does not resolve to a managed Decision Record, naming the referencing Decision Record and the unresolved reference. | >[SRS-118] |
| 7 | The Decision Records Overview page shall render a Kit column indicating, for each Decision Record, whether it has no declared prerequisites, is fully kitted, or is blocked by an unsatisfied prerequisite. | >[SRS-119] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 2 of; this ADR implements the Full Kit rule
- [[adr-193-owner-wip-heatmap]] — step 1 (the WIP signal) this builds on; its derive-from-the-Scope-table approach and "warn, don't fail" stance are reused here, and it is this record's own prerequisite
- [[adr-172-current-status-marker]] — the `*`-marked current status that the satisfied/kitted definition is keyed on
- [[adr-186-cross-document-links]] — the cross-document link resolution (`LinkRegistry`) and broken-link reporting this check extends
- [[adr-191-overview-target-date]] — the header-text-not-position, derive-don't-duplicate discipline followed here
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
