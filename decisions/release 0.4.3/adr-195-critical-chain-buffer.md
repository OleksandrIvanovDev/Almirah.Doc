---
title: "ADR-195: Estimates, Critical Chain, and Project Buffer"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 13-06-2026 | Proposed |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

This is the third implementation step of the planning/flow roadmap in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md). It follows [[adr-193-owner-wip-heatmap]] (WIP signal) and [[adr-194-full-kit-readiness]] (dependencies + full-kit gate), and implements the Critical Chain core of *Goldratt's Rules of Flow*: **strip the safety out of individual task estimates and protect the chain as a whole with a single project buffer**, instead of padding every task's due date.

The reasoning the book makes is that per-task safety is wasted three ways — Student Syndrome (work starts late because the deadline feels far off), Parkinson's Law (work expands to fill the padded estimate), and non-passing of early finishes (a task that finishes early rarely lets the next one start early). Aggregating that safety into one buffer at the end of the chain protects the commitment with far less total time, and turns scattered per-task deadlines into one number to watch.

The substrate from the prior two steps is exactly what this needs:

- A **record-level dependency graph**: `depends_on` on each `Decision`, resolved to other Decision Records ([[adr-194-full-kit-readiness]]). This is the network a chain runs through.
- A **record-level owner**: `owners` on each `Decision` ([[adr-193-owner-wip-heatmap]]). This is what creates resource contention — two records sharing an owner cannot run at the same time, which is precisely what makes a *critical chain* differ from a plain critical path.
- The **derive-from-the-Scope-table** discipline and the per-target-release grouping the overview already uses.

What is missing is a notion of how long work takes. Today the Scope table carries hand-set `Start Date` / `Target Date` per row — the padded per-task dates the book argues against. This ADR replaces that thinking with **two estimates per work item** and derives the schedule and buffer from them.

## Altitude note

Like steps 1 and 2, this step operates at the **Decision-Record level**: the chain runs over the inter-record `depends_on` graph, and a record's duration is the aggregate of its Scope-item estimates. This is a deliberate refinement of the per-item sketch in the analysis note — per-item dependency chains are not expressible yet (`depends_on` resolves to records, not sibling rows), so a per-item critical chain is deferred, consistent with the per-item deferrals in [[adr-193-owner-wip-heatmap]] and [[adr-194-full-kit-readiness]].

# Decision

Add **two estimate columns** to the Scope table, derive per-record focused/safe durations, compute a **resource-levelled critical chain and a project buffer per target release**, and render them on the overview.

## Estimate data source and extraction

Add two optional columns to the `# Scope` table, identified by header text (case-sensitive), not position:

- **`Est (focused)`** — the aggressive, no-safety estimate: how long the item takes with full focus and everything going well. This is the number that drives the schedule.
- **`Est (safe)`** — the comfortable estimate the author would otherwise commit to. The difference `safe − focused` is the safety being aggregated into the buffer.

Estimates are a positive number of **working days** (decimals allowed). Empty or unparseable cells count as `0`. Derive two aggregates on each `Decision`, reusing `find_section_table` / `column_index`:

- `focused_estimate` = sum of the `Est (focused)` column across Scope rows;
- `safe_estimate` = sum of the `Est (safe)` column across Scope rows.

A record with neither column, or all-empty cells, has a `focused_estimate` of `0` and contributes no duration to a chain.

## Critical chain and project buffer (per target release)

For each `Target Release Version`, build the planning network over the records targeting that release that are **not yet `Implemented`** (remaining work):

1. **Nodes** are those records; each has duration `focused_estimate` and resource set `owners`.
2. **Dependency edges** come from `depends_on` (restricted to records in the same release set; cross-release prerequisites are treated as already-available inputs, not scheduled nodes).
3. **Resource levelling.** Schedule with a deterministic, greedy heuristic: process records in priority order (longest dependency-path duration first, ties broken by sequence number) and start each at the earliest time at which all its prerequisites have finished **and** all of its owners are free. Two records sharing an owner therefore serialise even when no dependency links them.
4. **The critical chain** is the sequence of records, traced back from the latest-finishing record, whose start-to-finish leaves no slack in that resource-levelled schedule — i.e. the chain of dependency *and* resource hand-offs that sets the release's completion.
5. **The project buffer** is appended after the chain: `buffer = ceil(buffer_ratio × Σ_chain(safe − focused))`, with `buffer_ratio` defaulting to `0.5` (the standard cut-the-aggregated-safety-in-half rule). Negative per-record contributions (where safe < focused) are clamped to 0.
6. The release's **projected duration** is the chain length plus the buffer.

The heuristic is explicitly an approximation (optimal resource-levelled scheduling is intractable in general); determinism is required so that rendering and tests are stable across runs.

## Configuration

Extend the `planning:` key in `project.yml` introduced by [[adr-193-owner-wip-heatmap]]:

```yaml
planning:
  wip_limit: 2
  buffer_ratio: 0.5
```

`buffer_ratio` is optional; absent or out of the range `(0, 1]` falls back to `0.5`.

## Rendering

- The new estimate columns render as ordinary Scope-table columns on each record page (no special handling — they are Markdown table cells).
- Add a **"Critical Chain & Project Buffer"** section to the Decision Records Overview, one block per target release: the ordered chain (record IDs and durations), the computed buffer size, and the projected duration. Records with no estimates anywhere in a release are noted as "unestimated" so a half-estimated plan is not silently reported as short.

# Scope

| Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|
| Requirements | A.I. | >[ADR-194] | 1 | 2 | To Do |  |  | New SRS items SRS-120 through SRS-127 in `srs.md` "Planning" chapter: the two estimate columns (working days, empty = 0, read by header text); `focused_estimate` / `safe_estimate` aggregates; per-release network of not-yet-Implemented records; the resource-levelling heuristic (dependency + shared-owner serialisation, deterministic ordering); the critical-chain definition; the project buffer formula and `buffer_ratio` config with its default and range; the projected duration; and the overview "Critical Chain & Project Buffer" section including the unestimated note |
| Code | A.I. | >[ADR-194] | 3 | 6 | To Do |  |  | Add `focused_estimate` / `safe_estimate` accessors and `extract_estimates` on `Decision`; add a per-release critical-chain/buffer computation (a new planning module consuming `depends_on` + `owners` + estimates); read `planning.buffer_ratio` in `project_configuration.rb` (default 0.5, range-checked); render the "Critical Chain & Project Buffer" section in `decisions_overview.rb` |
| Tests | A.I. | >[ADR-194] | 2 | 5 | To Do |  |  | End-to-end tests under `spec/e2e/decisions_spec.rb`: estimates aggregate per record with empty/unparseable cells as 0; a linear dependency chain yields the expected chain and buffer; two unlinked records sharing an owner serialise and extend the chain (resource levelling); records sharing no owner and no dependency run in parallel; the buffer is `ceil(ratio × Σ(safe−focused))` with negative contributions clamped; `buffer_ratio` default and out-of-range fallback; Implemented records and cross-release prerequisites are excluded as nodes; an all-unestimated release is reported as unestimated; the schedule is deterministic across runs |

# Out of Scope

- **Per-Scope-item critical chains.** The chain runs over the record-level `depends_on` graph; per-item dependencies and per-item scheduling are deferred, consistent with the prior steps.
- **Feeding buffers** on non-critical paths that merge into the chain. Only the single project buffer is computed in this step; feeding buffers are a later refinement.
- **Optimal resource-levelled scheduling.** A deterministic greedy heuristic is used; provably-minimal makespan scheduling is out of scope.
- **Calendars, working-hours, holidays, and absolute date placement.** Durations are in working days and the chain is reported as lengths/ordering, not calendar dates. Mapping the chain onto the calendar (and a Gantt view) is deferred.
- **Buffer consumption / fever chart.** Tracking actual progress against the buffer is roadmap step 4 ([goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md)); this step only computes the plan, not its consumption.
- **Configurable estimate units or the sum-of-squares buffer method.** Working days and the 50%-cut method are fixed here; alternatives are noted under Alternatives Considered.
- **Cross-release dependency scheduling.** A prerequisite in another release is treated as an available input, not scheduled.

# Consequences

## Positive

- Replaces scattered, padded per-task `Target Date`s with the CCPM model the book advocates: aggressive estimates plus one protective buffer, making the safety explicit and shared rather than hidden and wasted.
- Reuses the dependency graph (step 2) and owner data (step 1) directly — the resource-levelled chain is exactly what those two steps were quietly assembling.
- Single source of truth preserved: estimates live in the Scope table; the chain and buffer are derived, never hand-maintained, consistent with [[adr-191-overview-target-date]].
- Produces a per-release buffer number that step 4's fever chart can consume without further data.

## Negative

- This is the largest single step: it introduces a real scheduling computation (a new planning module) rather than a one-method extraction, so it carries more implementation and test surface than steps 1–2.
- The chain depends on the header text of two more Scope columns; renaming them silently zeroes estimates. The test suite locks the headers, as for `Owner` and `Depends On`.
- A resource-levelling heuristic can differ from an optimal schedule; the reported chain is a good-faith approximation, and over-optimistic focused estimates feed directly into a short buffer.

## Neutral

- Records authored before this ADR work unchanged: with no estimate columns they contribute 0 duration and simply do not lengthen any chain.
- Re-rendering is required to compute and show the chain; nothing is retroactive.
- This record's own Scope items carry sample estimates and depend on [[adr-194-full-kit-readiness]] (still `Proposed`), so it remains not-fully-kitted — continuing the live demonstration begun in step 2.

# Alternatives Considered

- **Keep padded per-task `Target Date`s and roll them up.** Rejected: this is precisely the per-task-safety model the book identifies as the source of Student Syndrome and Parkinson's Law; aggregating safety into a buffer is the whole point of the step.
- **A single estimate per item plus a flat percentage buffer.** Rejected: the focused/safe pair makes the safety being removed explicit per item and lets the buffer reflect where uncertainty actually is, rather than a uniform pad.
- **Sum-of-squares (SSQ) buffer sizing instead of the 50% cut.** Reasonable and arguably better for long chains; deferred. The 50% cut is the simplest defensible default and `buffer_ratio` leaves room to tune; an SSQ option can be added later without changing the data model.
- **Critical path (dependencies only), ignoring resource contention.** Rejected as the headline result: ignoring shared-owner serialisation is what makes a critical *path* understate reality. Resource levelling is included precisely because step 1 already gives us owners; the contention-free case still falls out naturally when owners do not overlap.
- **A dedicated new document type / Plan page rather than an overview section.** Deferred: a new doc type is heavier than warranted now; the overview already groups by release and is the established home for derived planning views.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support two optional estimate columns, a focused estimate and a safe estimate, each expressing a work item's effort as a non-negative number of working days. The columns shall be identified by their header text, case-sensitive, and not by column position. | >[SRS-120] |
| 2 | The software shall expose focused-estimate and safe-estimate aggregates on each Decision Record, each computed as the sum of the corresponding estimate column across the Scope table rows, treating empty or unparseable cells as zero. | >[SRS-121] |
| 3 | For each Target Release Version, the software shall construct a planning network over the Decision Records targeting that release whose current status is not Implemented, with dependency edges taken from the Dependencies attribute restricted to records in the same release. | >[SRS-122] |
| 4 | The software shall schedule the planning network with a deterministic resource-levelling rule in which a record starts only when all its prerequisites have finished and all of its owners are free, so that records sharing an owner do not run concurrently. | >[SRS-123] |
| 5 | The software shall identify the critical chain of a release as the sequence of records that determines the release's completion in the resource-levelled schedule. | >[SRS-124] |
| 6 | The software shall compute a project buffer for each release as the configured buffer ratio multiplied by the aggregated safety along the critical chain, where each record's safety contribution is its safe estimate minus its focused estimate clamped to zero, rounded up to a whole working day. | >[SRS-125] |
| 7 | The software shall read an optional planning buffer ratio from the project configuration, applying a default of 0.5 when the value is absent or outside the range greater than 0 and at most 1. | >[SRS-126] |
| 8 | The Decision Records Overview page shall render, per Target Release Version, the ordered critical chain, the project buffer size, and the projected duration, and shall indicate when a release has no estimated work. | >[SRS-127] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 3 of; implements the aggressive-estimate-plus-project-buffer (Critical Chain) rule
- [[adr-194-full-kit-readiness]] — supplies the `depends_on` dependency graph the chain runs through; this record's own prerequisite
- [[adr-193-owner-wip-heatmap]] — supplies the `owners` data that creates resource contention and the `planning:` config key extended here
- [[adr-191-overview-target-date]] — the header-text-not-position, derive-don't-duplicate discipline followed here
- [[adr-181-overview-release-version]] — the per-target-release grouping the chain/buffer view is organised by
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
