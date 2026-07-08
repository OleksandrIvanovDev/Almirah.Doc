---
title: "ISSUE-220: Scheduler Queues Rows Behind Idle Lane Gaps"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-07-2026 | Proposed |
|   | 05-07-2026 | Accepted |
|   | 05-07-2026 | In Progress |
| * | 05-07-2026 | Implemented |

# Context

The work-item scheduler introduced by [[adr-198-workitem-gantt-visualization]] and reused by the critical-chain computation ([[adr-195-critical-chain-buffer]]) levels each owner's lane with a single forward-moving cursor: every placed row advances the owner's next-free day, and the next row of that owner can start no earlier than that cursor. Once the cursor has moved past a day, the lane can never be filled there again — the scheduler cannot backfill an idle gap left between rows placed earlier.

The critical-chain priority rule ([[adr-195-critical-chain-buffer]]) makes this visible: rows are placed in order of the longest downstream chain of focused estimates. A short row near the end of the network (typically a small Tests row, which is a sink whose downstream length is just its own estimate) is placed after every longer row, so it queues behind the owner's whole lane even when an idle gap that fits it exists much earlier.

The `release 0.4.4` planning group reproduces the defect. The TEST lane was scheduled in the order ADR-215.Tests, ADR-217.Tests, ADR-219.Tests, ADR-216.Tests, ADR-218.Tests. ADR-216.Tests (1 focused day) was ready on day 7, immediately after ADR-216.Code finished, and the TEST owner was idle on days 6 to 8 — yet the row was parked at day 13 because the three 2-day Tests rows had already advanced the TEST cursor past it. Two faults follow:

- **A phantom critical-chain member.** The parked row finished exactly when the next TEST row started, so the chain trace walked back through it as a resource hand-off and reported ADR-216.Tests on the critical chain. Its true float was six days; it does not bind the group's completion at all.
- **An inflated makespan and buffer.** The group's schedule came out one day longer than the same network allows with the gap filled (14 day columns instead of 13), and the project buffer aggregated the safety of a row that does not belong on the chain.

The root cause is in `lib/almirah/project/work_item_scheduler.rb` in `Almirah.Code`: resource levelling remembered only the owner's next-free day (a cursor) instead of which day spans the owner's lane actually occupies, so the information needed to reuse an idle gap was discarded at placement time.

# Decision

Replace the per-owner cursor with per-owner occupied-interval tracking, so resource levelling backfills idle gaps.

- The scheduler keeps, per owner, the list of already-placed rows of that owner's lane. A new row starts at the earliest day at or after its dependency finish where the lane stays clear for the row's whole duration — which may be a gap between rows placed earlier, not only the end of the lane.
- The binding predecessor used by the critical-chain trace is derived from the same search: it is the lane row whose finish the start actually had to wait behind, or the dependency whose finish coincides with the start. A backfilled row that merely touches another row's start does not bind it.
- The fix lives in the base `WorkItemScheduler`; `ChainScheduler` and `GanttScheduler` ([[adr-195-critical-chain-buffer]], [[adr-201-gantt-group-segmentation]]) inherit it unchanged, so the planning Gantt, the critical chain, and the project buffer all read the corrected schedule.
- SRS-123 is clarified so that "its owner is free" explicitly means an idle gap of the row's duration anywhere at or after the row's dependency finish, making gap backfill a required property of the levelling rule rather than an implementation choice.

On the `release 0.4.4` group the corrected schedule places ADR-216.Tests on day 7, drops it from the critical chain (which now runs through the DEV lane and ADR-218.Tests), and shortens the group's makespan from 14 to 13 day columns.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | DEV |  | 1 | 2 | Done | 05-07-2026 | 05-07-2026 | Root Cause Investigation |
| 2 | Code | DEV |  | 1 | 2 | Done | 05-07-2026 | 06-07-2026 | In `lib/almirah/project/work_item_scheduler.rb`, replace the per-owner next-free-day cursor with per-owner occupied-interval tracking and an earliest-fit search; derive the binding predecessor from the lane row the start actually waited behind; keep `ChainScheduler` and `GanttScheduler` inheriting the levelling unchanged |
| 3 | Requirements | BA |  | 1 | 2 | Done | 05-07-2026 | 06-07-2026 | Clarify SRS-123 so the resource-levelling rule explicitly requires each row to start at the earliest idle gap of its owner's lane that fits its duration at or after its dependency finish, so a low-priority short row backfills a gap instead of queueing behind the lane |
| 4 | Tests | TEST |  | 1 | 2 | Done | 07-07-2026 | 07-07-2026 | Extend `spec/critical_chain_spec.rb` with a regression case where a short low-priority row whose dependency finishes inside an idle lane gap is backfilled into that gap and stays off the critical chain; confirm the full suite passes |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Analysis | DEV | 1 | Investigation |
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 05-07-2026 | Code | DEV | 1 | Implementation |
| 05-07-2026 | Tests | TEST | 1 | Verification |

# Out of Scope

- **Changing the critical-chain priority rule.** The longest-downstream-chain ordering of [[adr-195-critical-chain-buffer]] is kept; only the placement of an already-prioritised row changes. Reordering priorities could mask this defect on some networks but does not remove it.
- **Globally optimal scheduling.** The scheduler remains a deterministic greedy list scheduler; backfill removes the observed artefact but does not turn it into an exhaustive resource-constrained-schedule optimiser.
- **Splitting a row across gaps.** A row occupies one contiguous span; a gap shorter than the row's duration stays idle.
- **Cross-group scheduling.** Groups are still scheduled independently per [[adr-201-gantt-group-segmentation]]; predecessors in other groups remain already-available inputs.

# Consequences

## Positive

- A row becomes ready inside an idle gap of its owner's lane and is scheduled there, so the Gantt reflects how the work would actually be done and lane order follows readiness rather than placement priority.
- The critical chain no longer reports phantom members created by artificial queueing, so buffer sizing and the fever chart ([[adr-196-buffer-fever-chart]]) read a chain that really binds the completion.
- The group makespan can only shrink or stay equal for any network, never grow: every row still starts at or after its dependency finish and lanes never double-book.

## Negative

- Schedules rendered before the fix are not comparable bar-for-bar with re-rendered ones; groups whose lanes had exploitable gaps will show earlier starts, a different chain, and a shorter projected duration.

## Neutral

- Determinism is preserved: the priority order is unchanged and the earliest-fit search is itself deterministic, satisfying SRS-139 and SRS-145.
- A working prototype of the Code and Tests items already exists uncommitted in `Almirah.Code` (all 426 examples pass); the scope rows above formalise, review, and land it.

# Alternatives Considered

- **Keep the cursor and reorder the chain priority** (place rows by resource-free earliest start first, as the base overview scheduler does). Rejected: it fixes the observed group by accident of ordering, but any network where priority and readiness disagree reproduces the defect; the missing capability is gap reuse, not a better ordering.
- **Post-pass compaction** (schedule with the cursor, then slide rows left into gaps). Rejected: a second pass must re-derive the binding predecessors anyway and can invalidate the chain trace it runs after; building placement on occupied intervals is simpler and single-pass.
- **Accept the behaviour and document it.** Rejected: the schedule is the input to critical-chain identification and buffer sizing; a one-day-inflated makespan and a phantom chain member misdirect the buffer and the fever chart, which exist precisely to trigger replanning decisions.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.4 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall schedule the planning network with a deterministic resource-levelling rule in which each row starts at the earliest day, at or after the finish of all its predecessor rows, where its owner's lane holds no other row for the row's whole duration, so that a row may fill an idle gap between rows placed earlier and rows sharing the same owner never run concurrently. | >[SRS-123] |

# References

- [[adr-198-workitem-gantt-visualization]] — introduced the swimlane Gantt and the resource-levelled forward pass this issue corrects
- [[adr-195-critical-chain-buffer]] — the critical-chain scheduler whose longest-downstream priority exposes the missing backfill, and the chain/buffer computation that consumed the wrong schedule
- [[adr-201-gantt-group-segmentation]] — the per-group scheduling scope the corrected levelling runs inside
- [[adr-196-buffer-fever-chart]] — downstream consumer of the chain and buffer this issue makes truthful
- [[adr-194-full-kit-readiness]] — the work-item dependency network the scheduler places
