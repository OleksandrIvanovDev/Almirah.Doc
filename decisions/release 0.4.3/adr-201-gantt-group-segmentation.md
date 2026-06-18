---
title: "ADR-201: Group-Segmented Gantt with Buffer Lane"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 18-06-2026 | Proposed |
| * | 18-06-2026 | Analysis |
|   |  | Accepted |
|   |  | In-Progress |
|   |  | Implemented |

# Context

Step 5 of the roadmap, [[adr-198-workitem-gantt-visualization]], put the per-row WorkItem network ([[adr-194-full-kit-readiness]]) on the Decision Records Overview as a **resource-swimlane Gantt**: one lane per owner, an abstract day-index axis, and a constant-duration bar per work item placed by `WorkItemScheduler` (forward pass + per-owner resource levelling). That chart schedules **every** work item across **every** record on **one global timeline** — the whole portfolio in a single strip.

But the planning unit the roadmap ([goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md)) actually commits to is a **group of records planned together**, not the whole portfolio. [[adr-197-decision-group-collection]] already collected exactly that unit — `@project_data.decision_groups`, an insertion-ordered list of `{ folder-name => [Decision, …] }` derived from the first-level sub-folders under `decisions/` — and left it inert, awaiting a consumer. The project buffer is likewise a **per-group** quantity: [[adr-195-critical-chain-buffer]] sizes one buffer for each delivery, not one for the portfolio.

So the single global Gantt mixes deliveries that are planned and buffered separately, and it has nowhere to hang a buffer. This step makes the Gantt **the first consumer of `decision_groups`**: it segments the chart into one block per group, laid left-to-right, and reserves a dedicated **Buffer lane** so each group's buffer has a home. It does **not** compute the buffer — that is [[adr-195-critical-chain-buffer]]'s work; this step builds the lane and leaves the same kind of single override hook ADR-198 left for durations.

Critical-chain *highlighting* is deliberately **not** part of this step (a balanced schedule highlights nearly every bar, so the signal is weak); the chain gets its own dedicated chart under [[adr-195-critical-chain-buffer]]. This ADR is purely the segmentation frame and the buffer lane.

# Decision

Replace ADR-198's single global timeline with a **group-segmented** Gantt: schedule each `decision_groups` group independently, lay the groups left-to-right as column blocks, add a **group band row** between the day-header and the lanes, and add a **Buffer lane** above the owner lanes carrying one placeholder buffer bar per group.

## Segmentation key — reuse `decision_groups`

The blocks are the groups of `@project_data.decision_groups` (ADR-197), in their existing insertion order (folder-encounter order). No new source of truth and nothing hand-listed, consistent with the derive-don't-duplicate discipline of [[adr-191-overview-target-date]]. Each group's work items are the work items of that group's `Decision` references; a `record_id => group-name` lookup built once from `decision_groups` maps each `WorkItem` (which carries `record_id`) to its block. Records under the `.` group (top-level, per ADR-197) form their own block like any other.

## Per-group sub-schedules laid side-by-side

`WorkItemScheduler` runs **once per group**, over that group's work-item subset only, each producing a local day axis starting at day 1 (its existing deterministic forward pass + per-owner resource levelling, unchanged). The groups are then placed left-to-right: group *g* occupies a contiguous column block offset by the summed widths of the prior groups plus a one-column gutter between blocks. This **replaces ADR-198's single global schedule** — resource levelling is now scoped to each "planned together" group, and the axis aligns with the per-group buffer.

Cross-group dependency edges (the in-group / cross-group tag from [[adr-194-full-kit-readiness]] / [[adr-197-decision-group-collection]]) are **not** scheduled across blocks: within a group's sub-schedule a predecessor in another group is treated as an **already-available external input** (start day 1), exactly as [[adr-195-critical-chain-buffer]] treats a prerequisite in another release as an available input rather than a scheduled node. So each block is self-contained and its day axis is honest.

Group order is `decision_groups` order and each per-group schedule is deterministic (ADR-198), so the whole segmented layout is stable across runs.

## The group band row — the horizontal group line

Between the day-header (`grid-row: 1`) and the lanes, emit a new **group band row** (`grid-row: 2`): one cell per group, each spanning its block's day columns (`grid-column: block-start / span block-width`), labelled with the group name. This is the conventional Gantt group line that sits over the day numbers and above the work items. Its leading corner cell over the Owner column is sticky-left like the existing `gantt_corner`.

## The Buffer lane

Directly above the owner lanes, emit a dedicated **Buffer lane** (`grid-row: 3`) whose sticky label is `Buffer`. Per group, a buffer bar sits in this lane starting at the group block's last work-item finish day, spanning the group's buffer length.

The buffer **length** is sourced from a single hook — a `buffer_for(group)` method, mirroring `WorkItemScheduler#duration_for` — that returns a **named-constant placeholder** until [[adr-195-critical-chain-buffer]] overrides it with the computed `ceil(buffer_ratio × Σ_chain(safe − focused))` for the group. This is the same staging discipline ADR-198 used for the constant 3-day duration: the **visual and the lane** are delivered now; the **accurate buffer size** is swapped in by ADR-195 through one override, with no rendering change.

## Grid and CSS

In `gantt_grid` (`decisions_overview.rb`) the row scheme shifts: day-header `grid-row: 1`, group band `grid-row: 2`, Buffer lane `grid-row: 3`, owner lanes `grid-row: i + 4` (was `i + 2`). The day-header numbers run **per block** (1 … block-width), reset at each block, since each group is its own plan. Add `.gantt_release_band` (the group band cell), `.gantt_buffer` (the sticky Buffer label), and `.gantt_buffer_bar` styles to the existing `.workitem_gantt` block in `templates/css/main.css`. The whole container is still omitted when there are no work items, and a group with no work items contributes no block.

# Scope

| # | Item | Owner | Depends On | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-197], >[ADR-198] | To Do |  |  | This decision record: segmenting the overview Gantt by `decision_groups`; per-group sub-schedules laid left-to-right with a gutter; cross-group predecessors treated as already-available external inputs; the group band row between header and lanes; the Buffer lane above the owners with a placeholder length swapped by [[adr-195-critical-chain-buffer]] via a single hook |
| 2 | Requirements | BA | >[ADR-197], >[ADR-198] | To Do |  |  | New SRS items SRS-141 through SRS-145 in the `srs.md` "Planning" chapter: the per-group block segmentation in folder-encounter order with per-block day axes; the group band row spanning each block; resource levelling scoped within a group with cross-group predecessors as external inputs; the Buffer lane above the owners with a per-group placeholder buffer bar until estimates exist; and the determinism of the segmented layout |
| 3 | Code | DEV | >[ADR-197], >[ADR-198] | To Do |  |  | In `decisions_overview.rb`: build a `record_id => group` map from `@project_data.decision_groups`; run `WorkItemScheduler` per group subset and lay blocks left-to-right with a gutter offset; treat cross-group predecessors as start-day-1 external inputs; shift the `gantt_grid` row scheme (header 1, band 2, buffer 3, owners `i + 4`) and emit per-block day headers; add a `gantt_release_band` row and a `gantt_buffer` lane with a `buffer_for(group)` placeholder hook; add `.gantt_release_band` / `.gantt_buffer` / `.gantt_buffer_bar` to `templates/css/main.css` |
| 4 | Tests | TEST | >[ADR-197], >[ADR-198] | To Do |  |  | E2E tests under `spec/e2e/decisions_spec.rb`: records in different folders land in distinct left-to-right blocks in folder-encounter order, each with its own day axis from 1; the group band spans exactly its block's columns and carries the folder name; a same-owner pair serialises within a block while the same owner in two different blocks does not interact; a cross-group predecessor is treated as available (does not push its dependent's block start); a Buffer lane renders above the owner lanes with one placeholder bar per group after that group's last work item; the segmented layout is identical across two runs |

# Out of Scope

- **Computing the buffer size.** The Buffer lane carries a placeholder length; the real per-group `ceil(buffer_ratio × Σ_chain(safe − focused))` is [[adr-195-critical-chain-buffer]]'s work, swapped in through the `buffer_for` hook.
- **Critical-chain highlighting on the Gantt.** Deliberately excluded (weak signal on balanced schedules); the chain gets its own dedicated chart under [[adr-195-critical-chain-buffer]].
- **Variable bar durations / real estimates.** Bars keep ADR-198's constant placeholder duration until [[adr-195-critical-chain-buffer]] supplies per-row estimates through `WorkItemScheduler#duration_for`.
- **A calendar / date axis.** The per-block axis stays an abstract working-day index; mapping onto real dates is deferred.
- **Cross-block dependency arrows.** Cross-group edges are treated as external inputs and not drawn as connectors between blocks.
- **Reconciling a group's folder with each record's `Target Release Version`.** The block key is the folder (per ADR-197); checking it against the records' declared release stays deferred, as in ADR-197.
- **Failing the build.** Like the rest of the flow series, this is an advisory view; the build always completes.

# Consequences

## Positive

- Makes `decision_groups` (ADR-197) its first real consumer, and aligns the Gantt's unit of segmentation with the unit the buffer is computed over — one block, one buffer.
- Gives the per-group buffer a permanent home (the Buffer lane) before the value exists, so [[adr-195-critical-chain-buffer]] adds only a computation behind one hook, not a new rendering — the same low-risk staging ADR-198 used for durations.
- Per-group resource levelling reads more truthfully than the global timeline: an owner shared by two separately-planned groups no longer artificially serialises across deliveries.
- No new source of truth: blocks derive from the folder layout the author already maintains.

## Negative

- Replaces ADR-198's single-pass global schedule with a per-group loop and a left-to-right offset layout — more rendering logic than ADR-198, and the row-scheme shift touches every lane's `grid-row`.
- The Buffer lane shows a placeholder until [[adr-195-critical-chain-buffer]] lands; a reader could mistake the placeholder for a real buffer. The lane label and tests mark it as not-yet-estimated.

## Neutral

- A project with all records in one folder renders a single block — visually close to ADR-198, plus the band and Buffer lane.
- Re-rendering is required; nothing is retroactive.

# Alternatives Considered

- **Keep one global timeline and only overlay a buffer at the far right.** Rejected: one portfolio-wide buffer contradicts the per-group buffer of [[adr-195-critical-chain-buffer]], and a global schedule serialises owners across deliveries that are planned independently.
- **Segment by each record's `Target Release Version` instead of the folder group.** Rejected for the same reason ADR-197 chose the folder: the folder is the author's explicit "planned together" grouping and the unit `decision_groups` already exposes; the declared release is a separate per-record assertion.
- **Render the Buffer lane only once ADR-195 computes a value (no placeholder).** Rejected: building the lane now fixes the chart frame and the grid-row scheme, so ADR-195 changes only a number; an empty lane would also leave the requested structure invisible until then.
- **Highlight the critical chain on these bars rather than a separate chart.** Rejected here (and deferred to [[adr-195-critical-chain-buffer]]): on a balanced schedule nearly every bar is critical, so the highlight carries little signal; a dedicated chain chart presents the constraint and its slack directly.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview work-item schedule shall be segmented into one block per decision-record group, the blocks laid left to right in the groups' folder-encounter order, with each block carrying its own day-index axis beginning at one. | >[SRS-141] |
| 2 | The software shall render a group band row between the day-header and the resource lanes, with one labelled cell per group spanning that group's day columns. | >[SRS-142] |
| 3 | The software shall schedule each group's work items independently with per-owner resource levelling scoped to the group, treating any predecessor belonging to another group as an already-available input rather than a scheduled work item. | >[SRS-143] |
| 4 | The Decision Records Overview shall render a Buffer lane above the resource lanes, drawing one buffer bar per group positioned after that group's last work item, with a placeholder length until the project buffer is available. | >[SRS-144] |
| 5 | The software shall produce the same segmented layout, block order, and per-group schedule on repeated runs over unchanged input. | >[SRS-145] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap; the per-group delivery view this segments the Gantt by
- [[adr-198-workitem-gantt-visualization]] — the resource-swimlane Gantt this segments and extends; its `duration_for` hook is the model for the `buffer_for` hook added here
- [[adr-197-decision-group-collection]] — supplies `@project_data.decision_groups`, the folder grouping this is the first consumer of
- [[adr-195-critical-chain-buffer]] — computes the per-group buffer that fills this Buffer lane via the `buffer_for` hook, and owns the separate critical-chain chart
- [[adr-194-full-kit-readiness]] — the per-row WorkItem network and the in-group / cross-group edge tag scheduled here
- [[adr-193-owner-wip-heatmap]] — the per-row owner roster (`ordered_owners`) reused for the lanes
- [[adr-191-overview-target-date]] — the derive-don't-duplicate discipline followed by reusing the folder grouping

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()