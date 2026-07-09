---
title: "ADR-198: Work-Item Network Visualization (Resource-Swimlane Gantt)"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-06-2026 | Proposed |
|   | 17-06-2026 | Analysis |
|   | 17-06-2026 | Accepted |
|   | 17-06-2026 | In-Progress |
|   | 17-06-2026 | Implemented |
| * | 09-07-2026 | Superseded |

# Context

Step 2 of the roadmap, [[adr-194-full-kit-readiness]], built the per-row **WorkItem network**: every Scope row is a node carrying its `Owner` and bounded per-row `Status`, with `predecessors` / `successors` edges that span both intra-record step order and cross-record `Depends On` links. That network is rich, but today it is only surfaced as **non-failing console warnings** (the two kit gates) and a one-word **`Kit`** column on the Decision Records Overview. There is **no visual** of the sequence or structure — you cannot see, at a glance, what runs after what, which role is loaded, or where work piles up.

Step 1, [[adr-193-owner-wip-heatmap]], added a WIP-by-owner bar, but that is a **point-in-time count** of `In-Progress` rows, not a time-ordered schedule. The roadmap in [gfa.md](./../../specifications/gfa/gfa.md) calls for a **load / staggering view keyed off `Owner`** (Rule 3 — *stagger against the constraint*): see each resource's queue laid out over time so release starts can be staggered against the most-loaded role. A resource-swimlane Gantt is exactly that view, and it costs almost nothing new in data — the network already exists.

The honest constraint is that **estimates do not exist yet**. Per-row estimates, the critical chain, and project buffers are [[adr-195-critical-chain-buffer]]'s work. So this step draws every work item with a **constant placeholder duration of 3 days** and lays the schedule on an **abstract day index** (1, 2, 3, …), not a calendar. The value delivered now is the *visual and the scheduling pass*; the value deferred is *accurate durations*.

This is deliberate sequencing rather than duplication. The **forward-schedule + resource-levelling pass** built here is the substrate [[adr-195-critical-chain-buffer]] will reuse: ADR-195's Scope already names a "row-as-task scheduling model," so this ADR builds that scheduler once, in a form ADR-195 extends by swapping the constant duration for per-row estimates and overlaying the buffer. ADR-198 therefore `Depends On` [[adr-194-full-kit-readiness]] only (the network), **not** ADR-195.

# Decision

Render the WorkItem network as a **resource-swimlane Gantt** on the Decision Records Overview, in a scrollable container placed **between the charts grid and the records table**. Each **owner is a lane** (row); the horizontal axis is an **abstract day index**; each work item is a **bar of constant 3-day duration** placed by a deterministic forward-schedule with **per-owner resource levelling** (same-owner bars never overlap). The schedule is a derived, server-side pass; the chart is static HTML/CSS.

## Placement and container

In `DecisionsOverview#to_html` (`decisions_overview.rb`), emit a new `div.workitem_gantt` **after `render_charts_grid` and before the `table.controlled.decisions_overview`**. The container scrolls **horizontally** (and vertically when lanes overflow). The leading **Owner column is sticky** (`position: sticky; left: 0`) so it stays fixed while the day columns scroll. The container is **omitted entirely** when there are no work items, exactly as an empty chart would be.

## Lanes — the owner roster

One lane per owner, reusing the **existing global roster** ordering already computed for the WIP chart (`ordered_owners` / first-seen across every record's Scope table in `decisions_overview.rb`). Every role named in any Scope table gets a lane, even an idle one, so the picture is the full resource roster — consistent with how [[adr-193-owner-wip-heatmap]] renders every owner.

The horizontal axis is **day indices 1, 2, 3 … N**, where N is the latest bar end. These are **abstract working-day offsets, not calendar dates** — an honest axis while no estimates or per-row start dates feed it. ([[adr-195-critical-chain-buffer]] and a later dating step may map this onto calendar time.)

## The schedule — a derived forward pass with resource levelling

A new server-side pass produces a `{ WorkItem => start_day }` map over **every** work item across **all** managed records (`@project_data.work_items`, populated by ADR-194's `link_work_items`). Constant duration `D = 3` is a **named constant** — the single hook [[adr-195-critical-chain-buffer]] replaces with a per-row estimate.

- **Forward pass over the dependency network.** `start(wi) = max(0, max(end(p)) for p in wi.predecessor_items)` and `end(wi) = start(wi) + D`. `WorkItem#predecessor_items` already spans intra-record step edges **and** cross-record `Depends On` edges, so no new graph walk is needed — the readiness network *is* the scheduling network.
- **Resource levelling (the chosen rule).** Within an owner lane, no two bars overlap: a single resource does one work item at a time. A **greedy list-scheduler** processes work items in a **deterministic priority order** — dependency-only earliest start, then `activity_rank`, then `record_id`, then `step` — assigning each `start = max(dependency_start, owner_free_cursor)` and advancing that owner's free cursor to the bar's end. Each lane is then a clean single row, and the layout *is* the staggering view Rule 3 asks for.
- **Deterministic across runs.** The priority order is a stable, total sort over fields that do not depend on hash iteration order, so the same input always yields the same schedule — matching [[adr-195-critical-chain-buffer]]'s determinism requirement.
- **Cycle-safe.** The network is a DAG by construction (intra-record edges run low→high step; cross-record edges resolve to existing records). The forward pass is still guarded by a visited / depth memo; on any unexpected cycle it breaks the back-edge deterministically and continues — **warn, don't fail**, matching the house stance from [[adr-194-full-kit-readiness]].

Owner is the **lane** dimension; activity type and the in-group / cross-group edge tag are **not** consumed here (the tag stays inert, as in [[adr-194-full-kit-readiness]] and [[adr-197-decision-group-collection]]).

## Rendering — a CSS grid, not a chart library

The chart is a **pure HTML/CSS grid** (Chart.js has no Gantt type and is not used here). The sticky Owner column sits left; day columns are sized by a fixed `--day-width` custom property; the total column count is the latest bar end. Each work item is a positioned cell spanning `D` columns from its start day, on its owner's row. Its label is `<record-id> <activity>` (e.g. `ADR-194 Code`); a tooltip lists its predecessors.

Bars are **coloured by `Status`**, reusing the existing decision-overview status palette — `Done` greyed (complete), `In-Progress` accented, `To Do` neutral. A **started-but-blocked** work item (`WorkItem#cross_record_violation?`) carries the **warning emphasis**, consistent with the `Kit` cell's treatment in `decisions_overview.rb`. `Done` work items are still drawn — the network is the network — styled as complete. The CSS lives in the existing decisions-overview stylesheet under `templates/`.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-194] |  |  | Done | 17-06-2026 | 17-06-2026 | This decision record: the resource-swimlane model over the [[adr-194-full-kit-readiness]] WorkItem network; the constant-duration **forward-schedule + per-owner resource-levelling** pass authored as the reusable substrate [[adr-195-critical-chain-buffer]] extends with estimates; the abstract day-index axis; status colouring with the cross-record-violation emphasis; and placement between the charts grid and the records table |
| 2 | Requirements | BA | >[ADR-194] |  |  | Done | 17-06-2026 | 17-06-2026 | New SRS items SRS-136 through SRS-140 in the "Planning" chapter of `srs.md`: the scrollable work-item schedule with a non-horizontally-scrolling Owner column and day-indexed columns; the constant 3-day bar; the predecessor-respecting start; per-owner non-overlap (deterministic resource levelling); and the per-bar Status indication with the blocked-item emphasis |
| 3 | Code | DEV | >[ADR-194] |  |  | Done | 17-06-2026 | 17-06-2026 | Add a deterministic forward-schedule + resource-levelling pass over `@project_data.work_items` (constant `D = 3` named constant; reuse `WorkItem#predecessor_items`, `activity_rank`; cycle-guarded) producing a `{ WorkItem => start_day }` map; add `render_workitem_gantt` to `decisions_overview.rb` emitting a sticky-first-column CSS grid between `render_charts_grid` and the records table, reusing `ordered_owners` for lanes and the status palette / `cross_record_violation?` for bar colour; add the `.workitem_gantt` styles to the overview stylesheet under `templates/`; omit the container when there are no work items |
| 4 | Tests | TEST | >[ADR-194] |  |  | Done | 17-06-2026 | 17-06-2026 | E2E tests under `spec/e2e/decisions_spec.rb`: the container renders between the charts and the table and is absent when no record has Scope owners; every owner across all records gets a lane; a bar spans 3 day-columns; a dependent's bar starts no earlier than its predecessor's finish (intra- and cross-record); two same-owner work items never overlap and serialise deterministically while different-owner unlinked items share day columns; a bar's class reflects its row Status and a started-but-blocked item carries the violation emphasis; the schedule is identical across two runs |

# Out of Scope

- **Real estimates and variable bar lengths.** Every bar is a constant 3 days; per-row estimates and the resulting variable durations are [[adr-195-critical-chain-buffer]]'s work, which replaces the single `D` constant.
- **Dependency arrows between bars.** Left-to-right placement implies sequence; drawing explicit SVG/CSS connectors between predecessor and successor bars is deferred.
- **A calendar / date axis.** The axis is an abstract working-day index; mapping it onto real dates needs per-row start dates and estimates and is deferred.
- **Buffers, critical-chain highlighting, and the fever chart** ([[adr-195-critical-chain-buffer]], [[adr-196-buffer-fever-chart]]).
- **Per-group scheduling or buffering.** The in-group / cross-group edge tag from [[adr-197-decision-group-collection]] stays inert here; scoping the schedule per group is [[adr-195-critical-chain-buffer]]'s concern.
- **Interactivity.** Drag, zoom, and lane collapsing are out; this is static server-rendered HTML.
- **Failing the build.** Like the rest of the flow series, this is an advisory view; the build always completes.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview page shall render a work-item schedule between the status charts and the records table, in a scrollable container whose leading Owner column does not scroll horizontally and whose remaining columns are indexed by day. | >[SRS-136] |
| 2 | The software shall draw each Scope work item across all Decision Records as a bar on its Owner's lane with a constant duration of three days, until per-row estimates are available. | >[SRS-137] |
| 3 | The software shall start each work item's bar no earlier than the latest finish among its predecessors, counting both lower-numbered same-record steps and resolved cross-record dependencies. | >[SRS-138] |
| 4 | The software shall place work items sharing an Owner so that their bars do not overlap on that Owner's lane, serialising the lane by resource levelling, and shall produce the same schedule on repeated runs. | >[SRS-139] |
| 5 | Each work-item bar shall indicate its row Status, and a started work item whose cross-record predecessor is not Done shall be visually emphasised, consistent with the Overview Kit cell. | >[SRS-140] |

# References

- [gfa.md](./../../specifications/gfa/gfa.md) — the roadmap this realises; the resource load / staggering view of Rule 3 (*stagger against the constraint*)
- [[adr-194-full-kit-readiness]] — the per-row WorkItem network (predecessors/successors, owners, bounded Status) this visualizes; this record's own prerequisite
- [[adr-193-owner-wip-heatmap]] — established the owner roster and the bounded per-row Status reused here; the WIP bar this complements with a time-ordered view
- [[adr-195-critical-chain-buffer]] — extends this scheduler, swapping the constant 3-day duration for per-row estimates and overlaying the project buffer
- [[adr-197-decision-group-collection]] — the decision-group folders whose edge tag stays inert in this step
- [[adr-191-overview-target-date]] — the derive-don't-duplicate discipline followed here (the schedule is derived, never hand-authored)

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/31)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/50)

