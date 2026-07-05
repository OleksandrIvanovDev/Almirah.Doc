---
title: "ISSUE-189: Fixed Top Navigation Hides Anchor Targets"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 07-06-2026 | Proposed |
|   | 07-06-2026 | Accepted |
|   | 07-06-2026 | In-Progress |
| * | 07-06-2026 | Implemented |

# Context

Navigating to an in-page anchor in the rendered HTML scrolls the target underneath the fixed top navigation bar, so the linked heading or controlled item is hidden behind it.

The bar is `#top_nav`, declared `position: fixed` at the top of the viewport in [main.css:240-246](./../../../Almirah.Code/lib/almirah/templates/css/main.css#L240-L246). The page body scrolls behind it (the bar's height is already compensated for ordinary reading by the `margin-top: 30px` on `#content`, [main.css:14-20](./../../../Almirah.Code/lib/almirah/templates/css/main.css#L14-L20)). However, when the browser resolves a URL fragment (`#some-id`) it aligns the target to the very top of the scroll container — the same region the fixed bar occupies — so the first line of the target is covered.

This affects every anchor target the renderer emits: heading anchors (`a.heading_anchor`, [main.css:47-51](./../../../Almirah.Code/lib/almirah/templates/css/main.css#L47-L51)) and the `id`-bearing controlled items linked from coverage, traceability, and cross-document references. It is purely a CSS offset defect; markup, linking, and scroll behaviour during normal reading are unaffected.

# Decision

Reserve a band at the top of the scroll container equal to (slightly more than) the fixed bar's height, so fragment navigation lands the target below the bar instead of underneath it.

Add a `scroll-padding-top` to the root scrolling element in `main.css`:

```css
html {
    scroll-padding-top: 40px;
}
```

`scroll-padding-top` is applied to the scroll container and is honoured by the browser only when scrolling to a snap or fragment target, so it offsets anchor navigation without shifting layout or affecting manual scrolling. Placing it on the root element makes it cover every anchor target regardless of which element carries the `id`, avoiding a per-target rule. The value is `40px` — a small margin above the ~30px bar height already encoded in `#content`'s `margin-top` — so the target clears the bar with a little breathing room.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  |  |  | Done | 07-06-2026 | 07-06-2026 | Add an `html { scroll-padding-top: 40px; }` rule to `lib/almirah/templates/css/main.css` so fragment navigation offsets below the fixed `#top_nav` |
| 2 | Requirements | BA |  |  |  | Done | 07-06-2026 | 07-06-2026 | Authored SRS-100 (under a new "Navigation" subsection of the User Interface chapter in `srs.md`) specifying that in-page anchor navigation positions the target clear of the fixed top navigation bar; recorded in this issue's Affected Documents section |
| 3 | Tests | TEST |  |  |  | Done | 07-06-2026 | 07-06-2026 | Automated end-to-end browser test (`spec/e2e/anchor_navigation_spec.rb`) that renders a tall document, serves the build over HTTP, jumps to a heading fragment in headless Chrome, and asserts the target's top edge lands at or below the fixed `#top_nav` bottom edge; traced to SRS-100. Verified to fail without the fix (target top 3px vs bar bottom 29px) and pass with it |

# Out of Scope

- **Smooth scrolling (`scroll-behavior: smooth`).** A separate UX preference, independent of the offset defect.
- **Making `#top_nav` non-fixed or reflowing content around it.** The fixed bar is intended behaviour; only the anchor offset is wrong.
- **Computing the offset dynamically from the actual rendered bar height.** A static value matched to the existing `#content` `margin-top` is sufficient; a measured offset adds JS for no practical gain at current bar sizing.
- **The navigation pane (`#nav_pane`) interactions** and any other layout concerns.

# Consequences

## Positive

- In-page anchor links land their target fully visible below the top bar in every newly rendered Almirah project.
- A single root-level rule covers all anchor target types (headings and controlled items) with no per-element markup change.
- No layout shift and no change to normal scrolling — the rule only affects fragment/snap navigation.

## Negative

- The `40px` offset is a static value tied to the current `#top_nav` height. If the bar's height changes (e.g., font-size or padding edits), the offset must be revisited to stay in sync with `#content`'s `margin-top`.

## Neutral

- Already-rendered projects must be re-rendered to pick up the fix; the change is in the template CSS, not applied retroactively.
- Visual appearance during ordinary reading is unchanged.

# Alternatives Considered

- **Per-target `scroll-margin-top` on heading anchors and controlled items.** Rejected: equivalent effect but requires touching every anchor-target selector and keeping them in sync; the single root `scroll-padding-top` is simpler and exhaustive.
- **A JavaScript scroll handler that adjusts position after fragment navigation.** Rejected: adds script and event wiring for behaviour the CSS property delivers natively in all current browsers.
- **Dynamically measuring `#top_nav` height and setting the padding from JS.** Rejected as out of scope: the bar height is stable and already mirrored by `#content`'s static `margin-top`; a static value keeps the fix CSS-only.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.1 |
| Issue Found in Version | 0.4.1 |
| Target Release Version | 0.4.2 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | When the user navigates to an in-page anchor, the software shall position the target so that it is fully visible and not obscured by the fixed top navigation bar. | >[SRS-100] |

# References

N/A

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
