---
title: "ADR-211: Per-Group Planning Start Dates"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-06-2026 | Proposed |
|   | 21-06-2026 | Accepted |
|   | 21-06-2026 | In-Progress |
| * | 21-06-2026 | Implemented |

# Context

Decision records are gathered into planning groups, one per first-level folder under `decisions/` ([ADR-197](./adr-197-decision-group-collection.md)), and the overview renders each group as a left-to-right block in the resource-swimlane Gantt ([ADR-198](./adr-198-workitem-gantt-visualization.md)), segmented by group ([ADR-201](./adr-201-gantt-group-segmentation.md)) and projected onto real calendar dates ([ADR-205](./adr-205-calendar-gantt-view.md)) on a business-day axis ([ADR-206](./adr-206-working-day-columns.md)).

The groups already appear in the order the user wants — folder-encounter order, which is alphabetical (`release 0.4.0`, `release 0.4.1`, …). What is missing is *calendar sequencing of the groups themselves*. Today every block shares one `WorkingCalendar` anchored at the single project-wide `planning.start_date`, and each block's `business_columns(width)` counts from working day 1. So each group's day and month headers restart at the same project anchor, and the accumulating column offset is purely spatial — it shifts a block rightward only so blocks do not visually overlap. The result is that groups are drawn side by side in space but **parallel in time**: there is no notion that `release 0.4.1` begins after `release 0.4.0`, and the today rule repeats inside every block.

Real planning needs the groups laid one after another on a shared timeline, with the freedom to let a later group begin before the previous one has fully finished. The motivating case is resource hand-off across the group boundary: the BA resource frees up and can open the next group's analysis while the TEST resource is still wrapping up the previous group. Different resources overlapping in time is exactly the parallelism that side-by-side, anchor-restarting blocks cannot express.

# Decision

Introduce an optional per-group planning **start date** and place each group's block on a single shared business-day axis at that date.

**Configuration.** Extend the existing `planning:` block in `project.yml` (parsed by `ProjectConfiguration`) with an optional `groups:` mapping from a group's first-level folder name to a `start_date` in the same `DD-MM-YYYY` form the project anchor and holidays already use, reusing `parse_planning_date` unchanged:

```
planning:
  start_date: 12-05-2026
  groups:
    release 0.4.0: 12-05-2026
    release 0.4.1: 25-05-2026
    release 0.4.2: 08-06-2026
    release 0.4.3: 22-06-2026
```

**Default.** A group without an entry anchors its block at the project `planning.start_date`. With no `groups:` mapping at all, every block anchors at the project start and the Gantt renders as it did before this ADR. The feature is opt-in per group, and each group is scheduled independently with its own buffer either way.

**Projection.** Each group's block re-anchors its `WorkingCalendar` ([ADR-205](./adr-205-calendar-gantt-view.md)) at its configured start date (or the project start date when unset), so the block's month, day, background, buffer, and today columns are labelled with that group's real dates instead of every block restarting at the project anchor. The block's *position* is unchanged from [ADR-201](./adr-201-gantt-group-segmentation.md): the blocks are laid left to right in folder order with a one-column gutter between them and never overlap. A start date sets the dates a block shows, not where it sits — so two groups whose dates overlap stay in separate, gutter-separated blocks, each read as its own mini-calendar, rather than colliding on shared columns.

**Today marker.** Because each block is its own calendar, the current-date rule is drawn only in the block whose date range contains today, and omitted from past and future blocks — at most one rule, where it means something.

**What does not change.** Group order is still folder-encounter (alphabetical) order ([ADR-197](./adr-197-decision-group-collection.md)); a start date relabels a group's block, it never reorders or repositions the groups. Each group is still scheduled in isolation by its own `GanttScheduler`, with its own critical chain and project buffer ([ADR-195](./adr-195-critical-chain-buffer.md)) computed exactly as before. The Critical Chain page's projected completion date, until now measured for every group from the project anchor, is now counted from the group's own start. Owner lane order ([ADR-204](./adr-204-consensus-owner-order.md)) and the canonical Scope table ([ADR-210](./adr-210-scope-table-format.md)) are untouched.

**Requirements.** This adds [SRS-163](./../../specifications/srs/srs.md) for the per-group start-date configuration and the block-calendar anchoring rule, generalises SRS-151 and SRS-155 from the single project anchor to each group's own start date, and amends SRS-160 so the today rule appears once, in the block containing today. SRS-141 keeps the sequential, gutter-separated layout; SRS-149 (the project start date) is unchanged — it is now the per-group fallback anchor.

**Why not overlap the blocks.** Positioning each block by its real start date on one shared axis was tried first; where group dates overlap it collided two groups' bars in the same owner lane and stacked their band labels, and the per-block today rules scattered across the chart. Keeping the sequential gutter-separated layout and using the start date only to *label* each block keeps the view clean; modelling true cross-group overlap is left to a future per-group lane-stack view (see Alternatives).

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 2 | Done | 21-06-2026 | 21-06-2026 | Specify the optional `planning.groups` mapping (folder name to `DD-MM-YYYY` start date), the block-calendar anchoring rule, the project-start-date fallback, and that the start date labels a block's dates without changing the sequential gutter-separated layout |
| 2 | Code | DEV |  | 2 | 3 | Done | 21-06-2026 | 21-06-2026 | Parse `planning.groups` in `ProjectConfiguration` reusing `parse_planning_date`; in the overview Gantt, anchor each block's `WorkingCalendar` at its group's start date (or the project start date) while keeping the left-to-right gutter-separated block layout; draw the today rule only in the block whose calendar contains today; count the Critical Chain page's per-group completion date from the same group start |
| 3 | Tests | TEST |  | 1 | 2 | Done | 21-06-2026 | 21-06-2026 | Add `Almirah.Code` specs for the config parsing and the per-block calendar labelling (declared date, project-start fallback, single in-range today rule, gutter retained); rebuild `Almirah.Doc` and confirm the blocks tile cleanly with no new broken links or parse errors |

# Out of Scope

- **True time-overlap of groups on the Gantt.** Blocks stay sequential and gutter-separated; a start date relabels a block, it does not let two groups share columns. Showing overlapping groups cleanly needs the per-group lane-stack view rejected below.
- **Cross-group resource levelling and double-book validation.** Because blocks do not overlap, the tool neither models nor checks a resource working two groups at once.
- **Reordering groups.** Order remains folder-encounter (alphabetical); the start date only relabels a group's block.
- **Changes to within-group scheduling, critical chain, or buffer.** Each group's plan is computed exactly as before; only the dates it is labelled with change.
- **A per-group marker file.** The start dates live in `project.yml`, not in a new file inside each group folder.

# Consequences

## Positive

- Each group's block now reads as its own mini-calendar with that group's real dates, instead of every block restarting at the project anchor.
- The Critical Chain page's projected completion date is counted from the group's own start, so it lands on the group's real calendar.
- The blocks stay sequential and gutter-separated, so the view is as clean and uncluttered as before — no colliding bars or stacked band labels.
- The today rule appears at most once, in the block that actually spans today, removing the scatter of edge-clamped rules.
- The change is opt-in and additive: a project with no `groups:` mapping renders unchanged, so there is no migration and no risk to existing output.

## Negative

- Start dates are maintained by hand and can drift from the real schedule; nothing reconciles a declared date against the group's computed duration.
- The blocks no longer form one continuous timeline: reading left to right across a gutter, the dates can step backward when a later group's start precedes the previous group's finish. Each block is meant to be read on its own.

## Neutral

- The start date does not move a block, so it cannot express that a group genuinely starts before the previous one finishes; that overlap is conveyed by the dates in the headers, not by position.
- Groups with no estimates or owners still contribute no bars and simply occupy their block as before.

# Alternatives Considered

- **Shared continuous axis positioning each block by its real start date.** Tried and rejected: where group dates overlap it collided two groups' bars in one owner lane, stacked their band labels, and scattered an edge-clamped today rule per block. Clean only when groups never overlap — which the sequential layout already guarantees without date arithmetic.
- **Per-group lane-stack (portfolio) view — each group its own owner lanes stacked vertically over one shared horizontal calendar.** The correct way to show overlapping groups cleanly, but a substantial rendering rework (lanes become per-group, the chart grows much taller) beyond this change; kept as the future home for true overlap.
- **Global resource-aware scheduling (one scheduler across all groups).** Rejected: it would compute and validate overlap automatically, but it dissolves the per-group block model and a single global axis does not fold each group's project buffer in cleanly. Keeping per-group scheduling preserves buffer correctness.
- **A per-group marker file (`group.yml` inside each folder) instead of `project.yml`.** Rejected: it adds a new file and parse path for one date per group; the existing `planning:` block already owns the project anchor and holidays, so the group dates belong beside them.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

This decision adds SRS-163 and amends SRS-141, SRS-151, SRS-155, and SRS-160 to the text below.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall read an optional per-group planning start date for each decision-record group from the project configuration in DD-MM-YYYY form, and shall anchor that group's Gantt block calendar — the dates labelling its day columns and the start its projected completion date is counted from — at that start date, falling back to the project start date when the group declares none. The blocks remain laid out sequentially (SRS-141); a group's start date sets the dates shown for its block, not the block's position. | >[SRS-163] |
| 2 | The Decision Records Overview work-item schedule shall be segmented into one block per decision-record group, the blocks laid left to right in the groups' folder-encounter order with a gutter column between adjacent blocks so they never overlap, each block carrying its own day-index axis beginning at one. | >[SRS-141] |
| 3 | The software shall map each working-day index of a decision-record group's schedule to a calendar date counted from that group's planning start date (SRS-163), skipping non-working days, so that the Nth working day falls on the Nth working date on or after that anchor. | >[SRS-151] |
| 4 | The Critical Chain page shall render, per decision-record group with an estimated critical chain, a projected completion date obtained by counting the projected duration in working days from that group's planning start date (SRS-163) across non-working days. | >[SRS-155] |
| 5 | The Decision Records Overview Gantt shall mark the current date with a full-height vertical rule at the column of the first working day on or after the current date, drawn only in a group block whose calendar range contains the current date and omitted from blocks the current date falls before or after. | >[SRS-160] |

# References

- [ADR-197](./adr-197-decision-group-collection.md) — the decision-group collection these start dates label
- [ADR-201](./adr-201-gantt-group-segmentation.md) — the sequential gutter-separated Gantt blocks this ADR keeps and re-labels per group
- [ADR-205](./adr-205-calendar-gantt-view.md) — the `WorkingCalendar` projection re-anchored per group
- [ADR-206](./adr-206-working-day-columns.md) — the business-day columns each block's dates are drawn on
- [ADR-210](./adr-210-scope-table-format.md) — the canonical Scope table this record follows

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
