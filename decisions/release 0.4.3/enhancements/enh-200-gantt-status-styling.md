---
title: "ENH-200: Work-Item Gantt Status Colours and Blocked-Item Pulse"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-06-2026 | Proposed |
|   | 17-06-2026 | Accepted |
|   | 17-06-2026 | In-Progress |
| * | 17-06-2026 | Implemented |

# Context

The work-item swimlane Gantt from ADR-198 shipped with a first-pass status palette: In-Progress bars in a solid Chart.js blue with white text, Done bars in mid grey, and blocked bars marked by a heavy `2px` bold dark-red inset outline. Against the rest of the Decision Records Overview — a light, flat, table-driven page — that palette read as too saturated and the blocked outline too heavy, while the lighter bars were hard to make out without a defined edge.

After dogfooding the chart on this repo's own decision records, the palette was reworked toward the page's existing colours (the controlled-table hover yellow, the table-border greys) and the blocked emphasis was moved from a static red outline to a slow background pulse, so a blocked item draws the eye through gentle motion rather than a loud border — which also keeps the light fills legible.

This enhancement records those refinements over the committed ADR-198 / [ENH-199](./enh-199-gantt-separator-lines.md) baseline.

# Decision

Restyle the three Gantt status bars and add a pulse for blocked items, scoped to the `.workitem_gantt` styles and the overview renderer only:

1. **In-Progress** — background becomes the controlled-table hover yellow `#ffffdd` with the regular page text colour `#333`, plus a `1px` inset border in the header-line grey `#bbb` so the light bar has a defined shape.
2. **Done** — background becomes a light green `#cfc` (text unchanged at `#57606a`), with a `1px` inset border in the container grey `#e1e4e8`.
3. **Blocked** — the bold weight is dropped and the inset outline is reduced from `2px` to `1px` and recoloured to the same `#bbb` as the other bars; the red is no longer carried by the border.
4. **Blocked pulse** — a small inline script eases each blocked bar's background between its *own* status colour (read at load) and the Chart.js red `rgb(255, 99, 132)` over a `2.4s`, six-phase cycle (`current → 1/5 → 1/3 → red → 1/3 → 1/5`), so red is a brief accent on a mostly-yellow bar. The script is emitted only when a blocked bar is present, leaves the bar's border untouched, and is skipped entirely under `prefers-reduced-motion`.

To Do bars are unchanged (`#9aa5b1`, white text, no border).

Implementation note: items 1–3 are CSS in `templates/css/main.css`; item 4 is the `gantt_pulse_script` method added to `decisions_overview.rb`, emitted as an inline `<script>` next to the chart (the same pattern the overview's Chart.js blocks use). The chart's HTML structure and the ADR-198 scheduling logic are untouched, so the ADR-198 end-to-end tests are unaffected.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  |  |  | Done | 17-06-2026 | 17-06-2026 | In `templates/css/main.css`: `.gantt_inprogress` → `background:#ffffdd; color:#333; box-shadow: inset 0 0 0 1px #bbb`; `.gantt_done` → `background:#cfc` with `box-shadow: inset 0 0 0 1px #e1e4e8`; `.gantt_blocked` → `box-shadow: inset 0 0 0 1px #bbb` (no bold, 1px). In `decisions_overview.rb`: add `gantt_pulse_script` (2.4s six-phase background pulse to `rgb(255,99,132)`, reads each bar's base colour, `prefers-reduced-motion`-aware) and emit it from `render_workitem_gantt` only when a blocked work item exists |

As with [ENH-199](./enh-199-gantt-separator-lines.md), these are presentational refinements, so there are neither requirements nor end-to-end tests in scope.

# Out of Scope

- Any change to To Do bar styling, lane layout, bar placement, or the scheduling logic from ADR-198.
- Row-hover highlighting of Gantt lanes (considered separately and deliberately not implemented).
- Variable bar durations, estimates, or buffers ([[adr-195-critical-chain-buffer]]).
- Making the pulse configurable (period, palette, phase weighting are fixed constants).

# Consequences

## Positive

- The chart's palette now matches the page's light, table-consistent aesthetic; the light bars gained a defined border so their shape is clear.
- Blocked items are signalled by motion rather than a heavy outline, so they stand out even though every fill is now light, without a loud static red.

## Negative

- The blocked pulse depends on JavaScript; with scripting disabled a blocked bar shows only its grey-bordered status fill, with no red accent (the console kit-violation warning and the overview `Kit` cell still convey the state).
- Animation can distract; this is mitigated by the slow `2.4s` cadence, the yellow-biased phase weighting, and honouring `prefers-reduced-motion`.

## Neutral

- The pulse is emitted only when a blocked bar exists, so most overview pages carry no extra script.
- No HTML structure or Ruby scheduling logic changed, so the ADR-198 renderer and its end-to-end tests are unaffected.

# Alternatives Considered

- **A pure-CSS `@keyframes` pulse.** Rejected: the pulse eases from each bar's *own* status colour (yellow or green), which is read from the computed style at runtime; a static keyframe cannot reference a per-element starting colour generically.
- **A solid red fill for blocked bars.** Rejected: it overrides the underlying To Do / In-Progress / Done colour, losing the status the rest of the chart conveys.
- **Keeping the original `2px` bold red outline.** Rejected: too heavy against the new light palette; the 1px grey border plus the pulse reads cleaner.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-198](./../adr-198-workitem-gantt-visualization.md) — introduced the work-item swimlane Gantt this enhancement restyles
- [ENH-199](./enh-199-gantt-separator-lines.md) — the immediately preceding cosmetic Gantt change (the header separator line)

# Review Evidences

- [Decision Record]()
- [Code]()
