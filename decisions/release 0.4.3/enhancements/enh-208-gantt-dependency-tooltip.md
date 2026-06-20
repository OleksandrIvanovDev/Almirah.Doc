---
title: "ENH-208: Readable Work-Item Gantt Dependency Tooltip"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-06-2026 | Proposed |
|   | 20-06-2026 | Accepted |
|   | 20-06-2026 | In-Progress |
| * | 20-06-2026 | Implemented |

# Context

The work-item swimlane Gantt from [ADR-198](./../adr-198-workitem-gantt-visualization.md) already gives each bar a native `title` tooltip naming the rows it waits on. It was a first-pass hint: a single comma-joined line, prefixed `After:`, listing each predecessor by its full canonical id (record, step number, activity — e.g. `ADR-201.1.Analysis`). With more than one or two predecessors the line grows wide and is hard to scan, the step number is noise for a reader who thinks in record-and-activity terms, and the bar's own identity is not repeated in the tooltip, so a hovered bar gives no anchor for the list that follows.

Dogfooding the Gantt on this repo's own decision records, where rows routinely carry both an intra-record phase predecessor and a cross-record `Depends On`, made the single-line form noticeably cramped. This enhancement reformats that existing hint for readability; it does not change which rows are considered predecessors (that set is defined by SRS-116) or any scheduling behaviour.

# Decision

Reformat the bar's `After:` tooltip from a one-line comma list into a short labelled block, scoped to `bar_tooltip` in `decisions_overview.rb`:

- **First line** — the hovered work item itself, written `RECORD-ID.Activity` with the record id upper-cased and the step number dropped (e.g. `ADR-208.Code`), so the tooltip is anchored by the bar it belongs to.
- **A blank line**, separating the item from its prerequisites.
- **`After:`** on its own line, then **one predecessor per line**, each in the same `RECORD-ID.Activity` form. Internal predecessors (the lower-numbered same-record phase steps) are listed first, then external ones (the resolved cross-record `Depends On` work items), matching how a reader reasons about "what came before this": its own earlier phases, then the other records it depends on.

The predecessor set is unchanged — every predecessor SRS-116 already defines, internal and external alike — only its presentation. A bar with no predecessors shows just its own name, with no `After:` block. Newlines render as line breaks in the native title tooltip (the attribute encoder preserves them), so no markup or scripting is involved.

Before and after, for a Code row with one phase predecessor and one cross-record dependency:

```
After: ADR-208.1.Analysis, ADR-201.2.Code
```

```
ADR-208.Code

After:
ADR-208.Analysis
ADR-201.Code
```

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | In `decisions_overview.rb`, rewrite `bar_tooltip` to emit the work item's own `RECORD-ID.Activity` name, a blank line, an `After:` label, and one predecessor per line (`intra_record_predecessors` then `cross_record_predecessors`, de-duplicated), each via a small `work_item_tip_label` helper that drops the step number; a predecessor-less bar shows only its name |
| 2 | Tests | TEST |  | 1 | 2 | Done | 20-06-2026 | 20-06-2026 | Extend the cross-record `Depends On` context in `spec/e2e/decisions_spec.rb` to assert a dependent Code bar's `title` equals its name, a blank line, `After:`, then the internal phase predecessor before the external `Depends On`, confirming both ordering and the step-free format |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 20-06-2026 | Code | DEV | 1 | reformat `bar_tooltip` and add `work_item_tip_label` in `decisions_overview.rb` |
| 20-06-2026 | Tests | TEST | 1 | multi-line `After:` tooltip assertion in `decisions_spec.rb` |

# Out of Scope

- **Which rows are predecessors.** The internal-plus-external predecessor set is defined by SRS-116 and is unchanged; only its rendering in the tooltip changes.
- **Distinguishing internal from external in the tooltip text** (e.g. separate headings or labels). Ordering carries the distinction; no extra labelling is added.
- **A richer hover panel** (HTML tooltip, statuses, dates, links to the predecessor bars). The native `title` attribute is kept, so the hint stays plain text.
- **Bar styling, lane layout, the calendar axis, or scheduling** from ADR-198, [ENH-199](./enh-199-gantt-separator-lines.md), [ENH-200](./enh-200-gantt-status-styling.md), and the calendar-axis work — untouched.

# Consequences

## Positive

- A bar with several predecessors is now scannable top-to-bottom instead of as one wide line, and the leading name anchors the list to the bar under the cursor.
- Dropping the step number and listing one prerequisite per line matches how the records are referred to elsewhere (record-and-activity), reducing visual noise.
- Internal-before-external ordering reads as a natural "own phases first, then other records," mirroring the predecessor definition.

## Negative

- The tooltip is taller; for a row with many predecessors it can become a long column, though still easier to read than the equivalent wide line.
- `RECORD-ID.Activity` omits the step number, so two same-record rows of the same activity type would render identically in the tooltip — a degenerate case the Scope step column otherwise disambiguates.

## Neutral

- Unlike the purely-cosmetic [ENH-199](./enh-199-gantt-separator-lines.md) / [ENH-200](./enh-200-gantt-status-styling.md), the tooltip text is deterministic and assertable, so a regression test is in scope here even though no new requirement is.
- No HTML structure, CSS, or scheduling logic changes; the ADR-198 renderer and its other end-to-end tests are unaffected.
- Already-rendered projects must be re-rendered to pick up the new tooltip; nothing is retroactive.

# Alternatives Considered

- **Keep the single comma-joined line, just drop the step number.** Rejected: narrower than before but still one wide line for many predecessors, and it still lacks the anchoring item name.
- **Label the dependency block `Depends On:` and list only cross-record predecessors.** Considered and reverted: `Depends On` is the framework's term for the cross-record column specifically, so it would wrongly exclude the intra-record phase predecessors a bar also waits on. `After:` (the existing ADR-198 wording) correctly covers every predecessor.
- **Two blank-line-separated sub-lists, one for internal and one for external predecessors.** Rejected as heavier than the hint warrants; a single ordered list (internal first) conveys the same with less height.
- **Replace the native `title` with a custom HTML tooltip** showing predecessor status and links. Deferred: a larger UX change beyond a readability fix, and it would forgo the zero-cost, scripting-free native tooltip.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-198](./../adr-198-workitem-gantt-visualization.md) — introduced the work-item swimlane Gantt and the original `After:` dependency hint this enhancement reformats
- [ENH-199](./enh-199-gantt-separator-lines.md), [ENH-200](./enh-200-gantt-status-styling.md) — the preceding cosmetic Gantt refinements this one follows
- [[adr-194-full-kit-readiness]] — defines a work item's intra-record and cross-record predecessors (SRS-116), the set the tooltip surfaces

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
