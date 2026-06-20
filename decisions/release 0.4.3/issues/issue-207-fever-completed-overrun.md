---
title: "ISSUE-207: Fever Chart Drops Buffer Consumed by Completed Chain Rows"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 20-06-2026 | Proposed |

# Context

The buffer-consumption fever chart ([[adr-196-buffer-fever-chart]]) is meant to show how much of the project buffer the critical chain has eaten. A chain row that overran its focused estimate consumed real buffer; in CCPM that consumption is permanent and stays on the chart for the rest of the project — finishing the task does not give the buffer back.

The implementation does the opposite. The fever chart is built from the group's `CriticalChain`, and `CriticalChain` excludes Done rows at construction (`work_items.reject(&:done?)`, per [[adr-195-critical-chain-buffer]], which correctly treats finished work as not needing re-scheduling). The page hands that same Done-filtered chain to `FeverChart`. So the moment a chain row is marked Done, its overrun disappears from buffer consumption and its completion weight disappears from the chain total — the most valuable signal (a task that blew its estimate) is erased exactly when it becomes a fact.

Two further faults follow from the same root:

- **The Done-credit branch is dead code.** [[adr-196-buffer-fever-chart]] specifies that the live point credits a Done row in full via its per-row `Status`, and `FeverChart#credit` implements `return 1.0 if live && work_item.done?`. But because Done rows are filtered out upstream, that work item never reaches `FeverChart`, so the branch can never run. ADR-195 ("drop Done rows") and ADR-196 ("credit Done rows") silently contradict each other, and ADR-195 wins.
- **The consumption denominator drifts.** The buffer is recomputed from only the remaining chain on every render, so as rows finish the buffer the chart measures *against* shrinks. Buffer consumption percent therefore has a moving baseline, instead of the fixed plan-time buffer CCPM compares against.

The net effect, reproduced on a two-row group whose Analysis row overran (5 days logged against a 3-day focused estimate) and was then marked Done: the live fever point jumps from a true "40% complete, buffer over-consumed" back to a misleading "remaining-work-only" point with zero recorded consumption. The team loses the one "act / replan" signal the chart exists to give.

The root cause is that a single `CriticalChain` object serves two jobs with opposite needs for completed work: **scheduling the remaining work** (projected duration, completion date) rightly drops Done rows, while **buffer accounting** (the fever chart) must keep them.

# Decision

Separate the two views inside `CriticalChain` and feed the fever chart the one that keeps completed work.

- Keep the existing `chain` / `buffer` / `projected_duration` as the **remaining-work** view (Done rows excluded), unchanged — it still drives the chain table and the projected completion date.
- Add a **baseline** view, `baseline_chain` and `baseline_buffer`, computed over the full Scope rows of the group with Done rows *included*, as the plan was originally sized.
- Point `FeverChart` at `baseline_chain` / `baseline_buffer`. Completion is then focused-weighted over every chain row (a Done row credits in full at the live point via its per-row `Status`, reviving the intended branch; historical trail points reconstruct even a completed row's progress and overrun from its dated `# Effort` log, which a per-row `Status` with no dated history could not). Consumption sums the overrun of all baseline chain rows over the stable baseline buffer.

`estimated?` reads the baseline too, so a fully-completed group still reports as estimated and keeps its (100%-complete) fever chart instead of collapsing to a "not sized" note.

The fever-chart math in `FeverChart` is unchanged — it was already correct; it was being fed the wrong set of rows. The chart still reads only the bounded per-row `Status` and the effort log, never the record lifecycle status, consistent with [[adr-193-owner-wip-heatmap]] and [[adr-172-current-status-marker]].

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  | 1 | 2 | To do | 20-06-2026 | 20-06-2026 | In `project/critical_chain.rb`, retain `@items = reject(&:done?)` for the remaining-work chain and add an `@all_items` baseline scheduler; expose `baseline_chain` and `baseline_buffer` (a shared `buffer_for(rows)` helper), and base `estimated?` on the baseline. In `project/fever_chart.rb`, read `plan.baseline_chain` / `plan.baseline_buffer` instead of `plan.chain` / `plan.buffer` |
| 2 | Tests | TEST |  | 1 | 2 | To do | 20-06-2026 | 20-06-2026 | Add an e2e case in `spec/e2e/decisions_spec.rb`: a Done chain row that overran keeps its overrun in buffer consumption and its full completion credit (the live point lands at the baseline-derived coordinate, not the remaining-only one); confirm the existing non-Done fever cases are unaffected because baseline equals remaining when nothing is Done |
| 3 | Requirements | BA |  | 1 | 2 | To do | 20-06-2026 | 20-06-2026 | Clarify SRS-131 / SRS-132 wording so completion and consumption are explicitly computed over the *baseline* critical chain (completed rows included) against the plan-time buffer, not the remaining-work chain |

# Out of Scope

- **Feeding-buffer consumption.** Only the single project buffer is accounted, as in [[adr-195-critical-chain-buffer]].
- **Re-baselining on scope change.** The baseline is derived from the current Scope rows and their estimates each render; it is stable while scope and estimates are stable, but this issue does not add a frozen, edit-time-snapshotted baseline that would survive later estimate edits.
- **Showing a completed row in the remaining-work chain table.** The left-hand chain table keeps listing only remaining rows; only the fever chart (right) accounts for completed ones.
- **Configurable fever-chart zones.** Unchanged from [[adr-196-buffer-fever-chart]].

# Consequences

## Positive

- A chain row that overran and then completed keeps consuming buffer, so the fever chart shows the act/replan signal it was built to show, rather than erasing it at completion.
- The Done-credit path promised by [[adr-196-buffer-fever-chart]] actually runs now, resolving the ADR-195 / ADR-196 contradiction.
- Buffer consumption has a stable plan-time denominator; the percentage no longer drifts as rows finish.
- The fix is small and additive: existing non-Done behaviour is byte-for-byte unchanged, because the baseline chain equals the remaining chain whenever no row is Done.

## Negative

- The baseline is recomputed per render from the current Scope rows, so editing a completed row's estimate after the fact retro-changes its recorded overrun. A truly frozen baseline (snapshotted at plan time) is deferred (see Out of Scope).
- A fully-completed group now renders a chain table with no remaining rows (buffer and projected duration 0) beside a 100%-complete fever chart, where it previously showed only a "not sized" note. This is intentional but is a visible cosmetic change.

## Neutral

- Re-rendering is required to pick up the corrected accounting; nothing is retroactive to already-built HTML.
- A working prototype of the Code and Tests items exists in the `Almirah.Code` working tree (the full decisions e2e suite passes, 140 examples) pending acceptance of this record.

# Alternatives Considered

- **Stop excluding Done rows in `CriticalChain` entirely.** Rejected: the remaining-work chain table and the projected completion date must exclude finished work, so a single un-filtered chain would break the schedule view to fix the accounting view. Two views is the correct decoupling.
- **Credit and account Done rows over the remaining buffer.** Rejected: it fixes the dropped overrun but leaves the consumption denominator drifting as rows finish; the baseline buffer is needed for a stable percentage.
- **Account buffer consumption over all Scope rows rather than the chain.** Rejected: only critical-chain overruns consume the *project* buffer in CCPM; non-chain (feeding) overruns belong to feeding buffers, which remain out of scope.
- **Carry forward only completed rows that overran, dropping completed rows that finished early.** Rejected as inconsistent: an early finish legitimately offsets nothing in this model (consumption is clamped at zero per row already), so keeping the whole baseline chain is simpler and correct.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The critical-chain completion and buffer consumption shall be computed over the baseline critical chain, which includes chain rows whose Status is Done, so that a row that overran and then completed continues to contribute its completion credit and its buffer consumption. | >[SRS-131] |
| 2 | The buffer consumption shall be expressed as a percentage of the plan-time project buffer derived from the baseline critical chain, so that the denominator does not change as chain rows complete. | >[SRS-132] |

# References

- [[adr-196-buffer-fever-chart]] — the fever chart this issue corrects; its Done-credit promise was unreachable and its consumption dropped completed overruns
- [[adr-195-critical-chain-buffer]] — supplies `CriticalChain` and the Done-row exclusion that is right for scheduling but wrong for buffer accounting; this issue adds the baseline view beside it
- [[adr-193-owner-wip-heatmap]] — the bounded per-row `Status` the live completion point reads
- [[adr-172-current-status-marker]] — the lifecycle status the fever chart still must not consult
- [[enh-202-critical-chain-page]] — the page that renders the chain table and the fever chart side by side

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
