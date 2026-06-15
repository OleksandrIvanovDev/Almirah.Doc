---
title: "ADR-195: Estimates, Critical Chain, and Project Buffer"
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

This is the third implementation step of the planning/flow roadmap in [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md). It follows [[adr-193-owner-wip-heatmap]] (WIP signal) and [[adr-194-full-kit-readiness]] (phase ordering + dependencies), and implements the Critical Chain core of *Goldratt's Rules of Flow*: **strip the safety out of individual task estimates and protect the chain as a whole with a single project buffer**, instead of padding every task's due date.

The book's reasoning: per-task safety is wasted three ways — Student Syndrome (work starts late because the deadline feels far off), Parkinson's Law (work expands to fill the padded estimate), and non-passing of early finishes (an early finish rarely lets the next task start early). Aggregating that safety into one buffer at the end of the chain protects the commitment with far less total time, and turns scattered per-task deadlines into one number to watch.

The prior two steps assembled exactly the network this needs, at the **role-phase row** altitude:

- **Within-record ordering**: the `#` step column and the rule that a row follows its lower-numbered steps ([[adr-194-full-kit-readiness]]). These are the intra-record edges.
- **A cross-record dependency graph**: each row's `Depends On` resolves to an **activity-type-aligned** predecessor work item in the referenced record ([[adr-194-full-kit-readiness]]). These are the inter-record edges.
- **A single owner per row**: each Scope row carries one `Owner` — a *resource* (an individual such as `BA1` / `BA2`, not merely a role) ([[adr-193-owner-wip-heatmap]]). This creates resource contention — rows sharing the *same* owner cannot run at once, while two people in one role (`BA1`, `BA2`) are distinct owners that run in parallel — which is precisely what makes a *critical chain* differ from a plain critical path.

What is missing is a notion of how long each row takes. Today the Scope table carries hand-set `Start Date` / `Target Date` per row — the padded per-task dates the book argues against. This ADR replaces that with **two estimates per row** and derives the schedule and buffer from them.

## Altitude note

The planning unit is the **role-phase Scope row**, established in [[adr-193-owner-wip-heatmap]] and [[adr-194-full-kit-readiness]]. The chain runs over **rows** (each a single-owner task with a duration and step/dependency edges), **not** whole records. This is the decisive choice for this step: a record carries several owners (BA, DEV, TEST), so a record-level node would be a multi-owner task whose resource-levelling degenerates (almost every pair of records shares some owner and would serialise) — and that record-level scheduler would have to be discarded when moving to rows. Building it at the row level from the start is both correct and avoids that throwaway work.

# Decision

Add **two estimate columns** to the Scope table, take each **row** as a scheduling task with a duration and a single owner, compute a **resource-levelled critical chain and a project buffer per target release** over those rows, and render them on the overview.

## Estimate data source and extraction

Add two optional columns to the `# Scope` table, identified by header text (case-sensitive), not position:

- **`Est (focused)`** — the aggressive, no-safety estimate: how long the row takes with full focus and everything going well. This drives the schedule.
- **`Est (safe)`** — the comfortable estimate the author would otherwise commit to. The difference `safe − focused` is the safety aggregated into the buffer.

Estimates are a non-negative number of **working days** (decimals allowed); empty or unparseable cells count as `0`. Each row's own focused estimate is its scheduling **duration**. For display, also expose per-record aggregates (`focused_estimate` / `safe_estimate` = the column sums), reusing `find_section_table` / `column_index`.

## Critical chain and project buffer (per target release)

For each `Target Release Version`, build the planning network whose **nodes are the not-`Done` Scope rows** of the records targeting that release (a `Done` row is finished work and is excluded; an `In-Progress` or `To Do` row contributes its full focused estimate as remaining work):

1. **Node** = a Scope row; **duration** = its `Est (focused)`; **resource** = its single `Owner`.
2. **Intra-record edges**: a row follows every lower-numbered step in its record (the `#`-column order from [[adr-194-full-kit-readiness]]); rows sharing a step number are concurrent.
3. **Cross-record edges**: each row carrying a `Depends On` reference follows its **activity-type-aligned predecessor work item** in the referenced record — the prerequisite row whose `Item` matches, falling back to the nearest earlier activity (the resolution defined in [[adr-194-full-kit-readiness]]). So a dependent's `Analysis` follows the prerequisite's `Analysis`, leaving it free to run in parallel with the prerequisite's `Code`. Prerequisites in another release are treated as already-available inputs, not scheduled nodes.
4. **Resource levelling.** Schedule with a deterministic greedy heuristic: process rows in priority order (longest downstream-path duration first; ties broken by record sequence number, then step number) and start each at the earliest time at which all its predecessor rows have finished **and** its owner is free. Rows sharing the **same owner** therefore serialise even with no edge between them; rows with **different owners** (including two people in one role, `BA1` and `BA2`) run concurrently, so capacity scales with staffing.
5. **The critical chain** is the sequence of rows, traced back from the latest-finishing row, with no slack in that resource-levelled schedule — a chain of dependency *and* resource hand-offs across role-phases that sets the release's completion.
6. **The project buffer** is appended after the chain: `buffer = ceil(buffer_ratio × Σ_chain(safe − focused))`, `buffer_ratio` defaulting to `0.5` (the standard cut-the-aggregated-safety-in-half rule). Per-row contributions where `safe < focused` are clamped to 0.
7. The release's **projected duration** is the chain length plus the buffer.

The heuristic is explicitly an approximation (optimal resource-levelled scheduling is intractable in general); determinism is required so rendering and tests are stable across runs.

## Configuration

Extend the `planning:` key from [[adr-193-owner-wip-heatmap]]:

```yaml
planning:
  wip_limit: 2
  buffer_ratio: 0.5
```

`buffer_ratio` is optional; absent or outside `(0, 1]` falls back to `0.5`.

## Rendering

- The estimate columns render as ordinary Scope-table cells on each record page (no special handling).
- Add a **"Critical Chain & Project Buffer"** section to the Decision Records Overview, one block per target release: the ordered chain as **role-phase rows** (record ID · row item · owner · duration), the buffer size, and the projected duration. A release whose rows carry no estimates is marked "unestimated" so a half-estimated plan is not silently reported as short.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-194] | 2 | 4 | In-Progress | 14-06-2026 |  | This decision record: the row-as-task scheduling model, the intra/cross-record edges, and the per-row buffer formula |
| 2 | Requirements | BA | >[ADR-194] | 1 | 2 | To Do |  |  | New SRS items SRS-120 through SRS-127 in `srs.md` "Planning" chapter: the two per-row estimate columns (working days, empty = 0, header text); per-row durations and per-record aggregates; the per-release network whose nodes are not-`Done` rows with intra-record step edges and cross-record activity-type-aligned edges; the deterministic resource-levelling rule keyed on the per-row `Owner` (distinct owners run in parallel); the critical-chain definition over rows; the buffer formula and `buffer_ratio` config; the projected duration; and the overview "Critical Chain & Project Buffer" section with the unestimated note |
| 3 | Code | DEV | >[ADR-194] | 3 | 6 | To Do |  |  | Add per-row `Est (focused)` / `Est (safe)` reading and per-record aggregates on `Decision`; add a planning module that builds the per-release row network (consuming the `#` order, the activity-type-aligned cross-record links from [[adr-194-full-kit-readiness]], per-row owners and durations), resource-levels it deterministically, traces the critical chain, and sizes the buffer; read `planning.buffer_ratio` in `project_configuration.rb` (default 0.5, range-checked); render the "Critical Chain & Project Buffer" section in `decisions_overview.rb` |
| 4 | Tests | TEST | >[ADR-194] | 2 | 5 | To Do |  |  | E2E tests under `spec/e2e/decisions_spec.rb`: per-row estimates with empty/unparseable = 0; a linear step chain within a record yields the expected ordering; two unlinked rows sharing an owner serialise (resource levelling) while rows with different owners and no edge run in parallel; a cross-record dependency places a dependent row after its activity-type-aligned predecessor in the prerequisite (a dependent's `Analysis` after the prerequisite's `Analysis`, free to run in parallel with the prerequisite's `Code`); the chain and buffer = `ceil(ratio × Σ(safe−focused))` with negatives clamped; `buffer_ratio` default and out-of-range fallback; `Done` rows and cross-release prerequisites are excluded as nodes; an all-unestimated release is reported unestimated; the schedule is deterministic across runs |

# Out of Scope

- **Feeding buffers** on non-critical paths merging into the chain. Only the single project buffer is computed; feeding buffers are a later refinement.
- **Optimal resource-levelled scheduling.** A deterministic greedy heuristic is used; provably-minimal makespan scheduling is out of scope.
- **Partial credit for In-Progress rows.** An In-Progress row contributes its full focused estimate as remaining work; consuming logged effort to discount it is roadmap step 4 ([[adr-196-buffer-fever-chart]]).
- **Calendars, working-hours, holidays, and absolute date placement.** Durations are working days; the chain is reported as ordering/lengths, not calendar dates. A Gantt/calendar view is deferred.
- **Buffer consumption / fever chart.** Tracking actual progress against the buffer is roadmap step 4; this step computes only the plan.
- **Configurable estimate units or the sum-of-squares buffer method.** Working days and the 50%-cut method are fixed here; alternatives are under Alternatives Considered.
- **Cross-activity per-row dependency targets.** A cross-record dependency resolves to the prerequisite's activity-type-aligned work item (the rule defined in [[adr-194-full-kit-readiness]]); pointing a row at a *different* activity (e.g. an `Analysis` row depending on a prerequisite's `Code`) remains the deferred per-row-target escape hatch from that ADR.

# Consequences

## Positive

- Replaces scattered, padded per-task `Target Date`s with the CCPM model the book advocates: aggressive estimates plus one protective buffer, making the safety explicit and shared rather than hidden and wasted.
- Consumes the role-phase network the prior steps built: single-owner rows make resource levelling **meaningful** (BA's analysis rows contend with each other, DEV's code rows with each other, but analysis does not contend with code), which a multi-owner record node could not express.
- Avoids throwaway work: scheduling at the row level from the start means no record-level scheduler (and its test suite) has to be built and then discarded when the per-row model lands.
- Single source of truth preserved: estimates live in the Scope table; the chain and buffer are derived, never hand-maintained, consistent with [[adr-191-overview-target-date]].
- Produces a per-release buffer that step 4's fever chart consumes without further data.

## Negative

- This is the largest single step: a real scheduling computation (a new planning module) rather than a one-method extraction, with correspondingly more implementation and test surface.
- The chain depends on the header text of two more Scope columns; renaming them silently zeroes estimates. The test suite locks the headers, as for `Owner` and `Depends On`.
- A greedy resource-levelling heuristic can differ from an optimal schedule; the reported chain is a good-faith approximation, and over-optimistic focused estimates feed directly into a short buffer.

## Neutral

- Records authored before this ADR work unchanged: rows with no estimate contribute 0 duration and do not lengthen any chain.
- Re-rendering is required to compute and show the chain; nothing is retroactive.
- This record's own rows carry sample estimates and depend on [[adr-194-full-kit-readiness]] (still in `Analysis`), so it remains not-fully-kitted — continuing the live demonstration from step 2.

# Alternatives Considered

- **A record-level critical chain (whole record as one node).** Rejected: a record carries several owners, so its node would be a multi-owner task whose resource-levelling degenerates (nearly every pair of records shares an owner and serialises), and the result would be discarded on moving to rows. Single-owner row nodes are both correct and reusable — this is the rework the per-row decision was made to avoid.
- **Keep padded per-task `Target Date`s and roll them up.** Rejected: precisely the per-task-safety model the book blames for Student Syndrome and Parkinson's Law.
- **A single estimate per row plus a flat percentage buffer.** Rejected: the focused/safe pair makes the removed safety explicit per row and lets the buffer reflect where uncertainty actually is.
- **Sum-of-squares (SSQ) buffer sizing instead of the 50% cut.** Reasonable for long chains; deferred. The 50% cut is the simplest defensible default and `buffer_ratio` allows tuning; an SSQ option can be added later without a data-model change.
- **Critical path (dependencies only), ignoring resource contention.** Rejected as the headline result: ignoring shared-owner serialisation makes a critical *path* understate reality. The contention-free case still falls out when owners do not overlap.
- **A dedicated new document type / Plan page rather than an overview section.** Deferred: heavier than warranted now; the overview already groups by release and is the established home for derived planning views.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support two optional estimate columns, a focused estimate and a safe estimate, each expressing a work-item row's effort as a non-negative number of working days. The columns shall be identified by their header text, case-sensitive, and not by column position. | >[SRS-120] |
| 2 | The software shall treat each Scope row's focused estimate as its scheduling duration, treating empty or unparseable estimate cells as zero, and shall additionally expose per-Decision-Record focused-estimate and safe-estimate aggregates equal to the column sums. | >[SRS-121] |
| 3 | For each Target Release Version, the software shall construct a planning network whose nodes are the not-Done Scope rows of the records targeting that release, with intra-record edges following the step-number order and cross-record edges placing each row that carries a Depends On reference after its activity-type-aligned predecessor work item in the referenced record. | >[SRS-122] |
| 4 | The software shall schedule the planning network with a deterministic resource-levelling rule in which each row starts only when all its predecessor rows have finished and its owner is free, so that rows sharing the same owner do not run concurrently while rows with different owners, including multiple people in one role, may. | >[SRS-123] |
| 5 | The software shall identify the critical chain of a release as the sequence of Scope rows that determines the release's completion in the resource-levelled schedule. | >[SRS-124] |
| 6 | The software shall compute a project buffer for each release as the configured buffer ratio multiplied by the aggregated safety along the critical chain, where each row's safety contribution is its safe estimate minus its focused estimate clamped to zero, rounded up to a whole working day. | >[SRS-125] |
| 7 | The software shall read an optional planning buffer ratio from the project configuration, applying a default of 0.5 when the value is absent or outside the range greater than 0 and at most 1. | >[SRS-126] |
| 8 | The Decision Records Overview page shall render, per Target Release Version, the ordered critical chain as role-phase rows, the project buffer size, and the projected duration, and shall indicate when a release has no estimated work. | >[SRS-127] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap this ADR is step 3 of; implements the aggressive-estimate-plus-project-buffer (Critical Chain) rule
- [[adr-194-full-kit-readiness]] — supplies the `#` step ordering and the activity-type-aligned cross-record dependency graph the chain runs through; this record's own prerequisite
- [[adr-193-owner-wip-heatmap]] — supplies the per-row single `Owner` that creates resource contention and the `planning:` config key extended here
- [[adr-191-overview-target-date]] — the header-text-not-position, derive-don't-duplicate discipline followed here
- [[adr-181-overview-release-version]] — the per-target-release grouping the chain/buffer view is organised by
- [[adr-170-introduce-decision-records]] — introduced decision records, the Status/Scope tables, and the overview page

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
