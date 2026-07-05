---
title: "ISSUE-203: Navigation Pane Button Shown on Pane-less Pages"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 18-06-2026 | Proposed |
|   | 18-06-2026 | Accepted |
|   | 18-06-2026 | In-Progress |
| * | 18-06-2026 | Implemented |

# Context

The top-navigation **"Navigation Pane"** toggle (`#document_tree_menu_item`, `title="Navigation Pane"`) is shown on the Decision Records Overview and — since [[enh-202-critical-chain-page]] — the new Critical Chain page, even though neither page renders a navigation pane. The button is therefore dead: clicking it calls `openNav()`, which toggles a `#nav_pane` element that has no tree content.

The chain of cause:

- The toggle button and the `<div id="nav_pane">` wrapper are both **static** in `page.html` (the button defaulting to `style="display: none;"`; the wrapper holding the `{{NAV_PANE}}` placeholder).
- `base_document.rb` fills `{{NAV_PANE}}` from `nav_pane.to_html` only when a nav pane is passed; a page rendered with `nav_pane = nil` leaves the `#nav_pane` div **empty**.
- `DecisionsOverview#to_html` and `CriticalChainPage#to_html` both call `save_html_to_file(html_rows, nil, …)` — no nav pane — so `#nav_pane` is empty on both.
- `main.js`'s `window.onload` decides the button's visibility purely by page title: it hides it for the `Document Index` and `Traceability Matrix:` pages and **shows it (`display: block`) on everything else**, regardless of whether a pane exists. So the two pane-less decision pages get a visible, non-functional button.

Individual decision records, specifications, and protocols pass a real `NavigationPane`, so their pane is populated and the button is legitimately useful there — the defect is specific to the pages rendered without a pane.

# Decision

Make the navigation toggle's visibility depend on the nav pane actually having content, not on the page title.

In `main.js`'s `window.onload`, in the branch that currently always shows the button, read `#nav_pane` and treat it as present only when its `innerHTML` is non-empty; show the button only when a pane is present (and the page is not a traceability matrix), otherwise leave it at its `display: none` default:

```js
var navPane = document.getElementById('nav_pane');
var hasNav = navPane && navPane.innerHTML.trim() !== "";
if (document.title.includes("Traceability Matrix:") || !hasNav) { /* hide */ }
else { /* show */ }
```

This is a general gate rather than special-casing the two page titles: the button now appears exactly on pages that render a navigation tree, so the Decision Records Overview and Critical Chain page no longer show it, while specifications, protocols, and individual decision records are unchanged.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  |  |  | Done | 18-06-2026 | 18-06-2026 | In `templates/scripts/main.js` (`window.onload`), gate the `#document_tree_menu_item` toggle on `#nav_pane` having non-empty `innerHTML`, so it shows only when a navigation pane is rendered; pane-less pages (Decision Records Overview, Critical Chain) keep the button at its `display: none` default |
| 2 | Requirements | BA |  |  |  | Done | 18-06-2026 | 18-06-2026 | No new requirement — UI defect; the navigation-pane toggle is expected to appear only on pages that render a navigation pane |
| 3 | Tests | TEST |  |  |  | Done | 18-06-2026 | 18-06-2026 | Verified by rendering and inspecting output: the `#nav_pane` div is empty on `decisions/overview.html` and `decisions/critical-chain.html` (button stays hidden) and populated on an individual decision record (button shown). No automated assertion was added because the visibility is decided by page script at load time and the e2e suite parses static HTML without executing JavaScript; the full suite remains green (353 examples) |

# Out of Scope

- **The collapsed-pane strip (`#closed_nav_pane`)** and the `openNav` / `closeNav` behaviour itself — only the top-navigation toggle's visibility is corrected.
- **Server-side suppression** of the button or the `#nav_pane` wrapper for pane-less pages; the markup stays static and visibility remains a client-side concern, consistent with the existing title-based logic.
- **The Index and Traceability Matrix pages**, whose existing hide rules are retained as-is.

# Consequences

## Positive

- The Decision Records Overview and Critical Chain page no longer present a dead navigation toggle.
- The gate is general: any current or future page rendered without a navigation pane hides the toggle automatically, rather than relying on a per-title exception list.

## Negative

- Visibility now reads `#nav_pane`'s content at load; a page that rendered a pane as only whitespace would be treated as pane-less. No such page exists (a real pane always emits element markup).

## Neutral

- Already-rendered projects must be re-rendered to pick up the corrected script; the fix is in the gem asset, not applied retroactively.
- Pages with a populated pane (specifications, protocols, individual decision records) are unchanged.

# Alternatives Considered

- **Hide the button by matching the two page titles** (`Decision Records Overview`, `Critical Chain & Project Buffer`) in `main.js`, extending the existing title list. Rejected: brittle (renaming a page silently reintroduces the dead button) and it does not cover future pane-less pages; gating on pane content fixes the class of problem.
- **Stop emitting the button server-side when `nav_pane` is nil.** Rejected as heavier: the button and pane wrapper are static template markup, and the existing visibility logic already lives in `main.js`; a content gate there is the smallest change consistent with the current design.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | 0.4.3 |
| Target Release Version | 0.4.3 |

# References

- [[enh-202-critical-chain-page]] — added the Critical Chain page, the second pane-less page where the dead toggle was visible
- [[adr-170-introduce-decision-records]] — introduced the Decision Records Overview (the first pane-less page) and the page/menu structure
- [[issue-190-nav-pane-scroll]] — prior navigation-pane behaviour fix

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
