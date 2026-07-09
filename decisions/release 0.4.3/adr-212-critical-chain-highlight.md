---
title: "ADR-212: Critical Chain Highlight on the Gantt"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-06-2026 | Proposed |
|   | 21-06-2026 | Accepted |
|   | 21-06-2026 | In-Progress |
|   | 21-06-2026 | Implemented |
| * | 09-07-2026 | Superseded |

# Context

The Decision Records Overview Gantt ([ADR-198](./adr-198-workitem-gantt-visualization.md)) draws one bar per Scope work item, segmented into per-group blocks ([ADR-201](./adr-201-gantt-group-segmentation.md)) on a calendar axis ([ADR-205](./adr-205-calendar-gantt-view.md), [ADR-206](./adr-206-working-day-columns.md)). Each bar is coloured by its row Status and a started-but-blocked row is emphasised ([SRS-140](./../../specifications/srs/srs.md)).

The critical chain — the longest resource-levelled path of focused-estimate durations through a group, the constraint that sets the group's finish ([ADR-195](./adr-195-critical-chain-buffer.md)) — is already computed for every block. The Gantt's own `GanttScheduler` traces it (`critical_chain`), and the dedicated Critical Chain page ([ENH-202](./enhancements/enh-202-critical-chain-page.md)) lists it as a table per group. What the Gantt does not do is show *which bars* are on that chain. A reader looking at the schedule cannot tell, without cross-referencing the Critical Chain page, which bars are the ones that must not slip — exactly the bars CCPM says to watch. The data is in hand; only the bars are untagged.

# Decision

Outline, on each group block, the work-item bars that lie on that block's critical chain.

- **Which chain.** The chain is `GanttScheduler#critical_chain` — the chain of the very scheduler that lays out the bars, traced back through the dependency and resource hand-offs that bind each start. Using the layout scheduler's own chain (rather than the Critical Chain page's `CriticalChain#chain`) keeps the highlight consistent with the drawn bar positions: every outlined bar is one the Gantt actually placed, in the order it placed them. No new computation is introduced; the chain is read off the existing schedule and collected into a per-block set.
- **Visual channel.** The highlight is a thin outline in the project buffer's amber hugging the bar, deliberately on a *separate* channel from the two already in use: row Status is the bar's background fill, and the blocked-row emphasis is an inset box-shadow with a red pulse. Because outline, background, and box-shadow are independent CSS properties, one bar can carry all three at once — a To-Do, on-chain, blocked bar reads as a grey fill inside an amber outline with a red pulse — and none masks another.
- **Scope of the highlight.** Only work-item bars are outlined. The project-buffer bar at the tail of each block is the chain's aggregated safety, already drawn as a distinct element, and is left as-is.

This is purely additive to the render: a new CSS class on the bars that are on the chain, and the per-block chain set the class is decided from. The schedule, the chain, the buffer, and the layout are unchanged.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 1 | Done | 21-06-2026 | 21-06-2026 | State SRS-164: the Gantt shall distinguish the bars on each group block's critical chain on a channel separate from the Status colour and the blocked emphasis |
| 2 | Code | DEV |  | 1 | 2 | Done | 21-06-2026 | 21-06-2026 | Collect `GanttScheduler#critical_chain` into the block descriptor; add the `gantt_critical` class to a bar whose work item is in that set; add the outline CSS rule beside the Status and blocked rules |
| 3 | Tests | TEST |  | 1 | 1 | Done | 21-06-2026 | 21-06-2026 | Add an `Almirah.Code` e2e asserting the chain bars carry `gantt_critical` and an off-chain bar does not; rebuild `Almirah.Doc` and confirm no new broken links or parse errors |

# Out of Scope

- **Highlighting the project-buffer bar.** The buffer is the chain's safety tail and already has its own distinct styling.
- **A Gantt legend.** Naming the channels (Status / on-chain / blocked) in an on-page key is a separate readability enhancement.
- **Reconciling with the Critical Chain page's chain.** That page excludes Done rows and uses exact durations; the Gantt highlights its own layout scheduler's chain, so the two may differ by a row (see Consequences).
- **Any change to scheduling, the chain, or the buffer.** The highlight only reads the existing schedule.

# Consequences

## Positive

- The constraint is now visible directly on the Gantt: a reader sees at a glance which bars carry the group's finish, without opening the Critical Chain page.
- The highlight composes with the existing encodings because it is a separate visual channel — Status fill, on-chain outline, and blocked pulse coexist on one bar.
- It reuses the already-computed chain, so there is no new scheduling math and no measurable render cost.

## Negative

- The Gantt highlights the layout scheduler's chain (rounded day durations, completed rows included), while the Critical Chain page lists the exact-duration remaining chain (Done rows excluded). On a group with completed or fractional-estimate rows the two chains can differ by a row, so a reader comparing the views may see a small discrepancy.

## Neutral

- A group with no estimates still has a chain by dependency path, so some bars are outlined even when every duration is the one-day placeholder; harmless, and consistent with how the unestimated group is already scheduled.
- The outline follows the bar's rounded corners in current browsers; in much older engines it may render as a square ring, a cosmetic-only fallback.

# Alternatives Considered

- **Colour the chain bars (a distinct background).** Rejected: the background fill already encodes row Status, so recolouring would either hide the Status or fight it. An outline leaves the fill free.
- **Highlight the Critical Chain page's `CriticalChain#chain` instead.** Rejected: it excludes Done rows and uses exact durations, so its rows would not line up one-to-one with the drawn bars; the layout scheduler's own chain is the one consistent with the bars' positions.
- **A separate critical-chain overlay or toggle.** Rejected as heavier than the need: a single always-on outline conveys the chain without new controls or a second render pass.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

This decision adds SRS-164.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Records Overview Gantt shall visually distinguish the work-item bars lying on their group block's critical chain, using a channel separate from the row-Status colour and the blocked-work-item emphasis so the three can be read together. | >[SRS-164] |

# References

- [ADR-195](./adr-195-critical-chain-buffer.md) — the critical chain and project buffer this highlight surfaces
- [ADR-198](./adr-198-workitem-gantt-visualization.md) — the work-item Gantt whose bars are outlined
- [ADR-201](./adr-201-gantt-group-segmentation.md) — the per-group blocks each chain is scoped to
- [ENH-202](./enhancements/enh-202-critical-chain-page.md) — the Critical Chain page that lists the same chain as a table

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
