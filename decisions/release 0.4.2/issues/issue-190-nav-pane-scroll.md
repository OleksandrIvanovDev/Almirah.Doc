---
title: "ISSUE-190: Navigation Pane Cannot Scroll to Its Last Items"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 07-06-2026 | Proposed |
|   | 07-06-2026 | Accepted |
|   | 07-06-2026 | In-Progress |
| * | 07-06-2026 | Implemented |

# Context

The left navigation pane (`#nav_pane`), opened on demand, holds the document sections tree. When the tree is taller than the viewport, scrolling to the bottom does not reveal the last items — they stay below the fold and are unreachable.

The pane is declared in [main.css:226-237](./../../../Almirah.Code/lib/almirah/templates/css/main.css#L226-L237) as `position: fixed; height: 100%` with `padding: 32px 8px 8px 8px` and `border: 1px solid #ddd`, and scrolls internally via `overflow-y: auto`. With the default `box-sizing: content-box`, padding and border are added *outside* the declared height, so the pane's rendered border box is `100vh + 32px + 8px + 2px = 100vh + 42px` — taller than the viewport. Its bottom edge, which is also the end of the internal scroll range, therefore sits ~42px below the fold. At maximum scroll the last tree items land off-screen and can never be brought into view.

This is a computed layout/scroll defect, not a markup or content problem. The on-demand visibility toggle in [main.js:1-11](./../../../Almirah.Code/lib/almirah/templates/scripts/main.js#L1-L11) only flips `visibility` and does not affect sizing.

# Decision

Anchor the pane to both the top and the bottom of the viewport instead of giving it a percentage height, so its border box spans exactly the viewport and its padding and border are accounted for inside that box.

Replace `height: 100%` with top/bottom anchoring in the `#nav_pane` rule:

```css
#nav_pane {
    position: fixed;
    top: 0;
    bottom: 0;
    /* ... */
}
```

When both `top` and `bottom` are set on a fixed element with `height: auto`, the used height is the distance between them and the border box fills the viewport exactly, regardless of `box-sizing`; padding and border are drawn inside it. The internal `overflow-y: auto` scroll range then ends at the viewport's bottom edge, so the last item is reachable. The existing `padding-top: 32px` continues to keep the first item clear of the fixed top navigation bar.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Implemented | 07-06-2026 | 07-06-2026 | In `lib/almirah/templates/css/main.css`, replace `#nav_pane`'s `height: 100%` with `top: 0; bottom: 0;` so the fixed pane spans exactly the viewport and its internal scroll reaches the last item |
| Requirements | Implemented | 07-06-2026 | 07-06-2026 | Authored SRS-101 (under the "Navigation" subsection of the User Interface chapter in `srs.md`) specifying that the navigation pane scrolls so every sections-tree item can be brought fully into view; recorded in this issue's Affected Documents section |
| Tests | Implemented | 07-06-2026 | 07-06-2026 | Automated end-to-end browser test (`spec/e2e/nav_pane_scroll_spec.rb`) that renders a tall sections tree, opens the pane, scrolls it to the bottom in headless Chrome, and asserts the last item is fully within the viewport; traced to SRS-101. Verified to fail without the fix (last item bottom 534px vs viewport 513px) and pass with it |

# Out of Scope

- **Restyling or resizing the pane**, its colours, fonts, or the responsive width breakpoints.
- **Changing the open/close interaction** or the on-demand visibility behaviour.
- **The fixed top navigation bar anchor offset** — a separate concern addressed in [ISSUE-189](./issue-189-anchor-scroll-offset.md).
- **A global `box-sizing: border-box` reset.** Top/bottom anchoring fixes this pane without altering the box model project-wide; a broad reset is a larger, riskier change.

# Consequences

## Positive

- The entire sections tree is reachable by scrolling, regardless of its length, in every newly rendered Almirah project.
- The fix is robust against `box-sizing` because top/bottom anchoring makes the border box fill the viewport independent of the box model.
- No markup or JavaScript change; a single CSS rule edit.

## Negative

- The pane is now explicitly pinned to the full viewport height. If a future design wants it to start below the top navigation bar (rather than underlapping it with `padding-top`), the `top` value must be revisited.

## Neutral

- Already-rendered projects must be re-rendered to pick up the fix; the change is in the template CSS, not applied retroactively.
- Visual appearance for trees that already fit within the viewport is unchanged.

# Alternatives Considered

- **Add `box-sizing: border-box` to `#nav_pane` and keep `height: 100%`.** Equivalent effect for this element, but relies on `height: 100%` resolving against the viewport and on the `top: auto` static position being zero; top/bottom anchoring states the intent directly and is independent of those assumptions.
- **Subtract the padding via `height: calc(100% - 42px)`.** Rejected: brittle magic number that must track every padding/border change, where top/bottom anchoring needs no arithmetic.
- **Project-wide `* { box-sizing: border-box }` reset.** Rejected as out of scope: too broad a change to introduce for a single pane's defect.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.1 |
| Issue Found in Version | 0.4.1 |
| Target Release Version | 0.4.2 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | When the document sections tree in the navigation pane is taller than the viewport, the software shall allow the pane to scroll so that every item, including the last one, can be brought fully into view. | >[SRS-101] |

# References

- [ISSUE-189](./issue-189-anchor-scroll-offset.md) — the preceding issue in this release, a related fixed-layout/scroll defect; used as the structural template for this one
- [ADR-170](./../../release%200.4.0/adr-170-introduce-decision-records.md) — the decision-record convention this ISSUE follows
