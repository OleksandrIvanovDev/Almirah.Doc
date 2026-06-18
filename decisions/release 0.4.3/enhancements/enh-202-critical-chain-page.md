---
title: "ENH-202: Move Critical Chain to a Dedicated Page"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 18-06-2026 | Proposed |
|   | 18-06-2026 | Accepted |
|   | 18-06-2026 | In-Progress |
| * | 18-06-2026 | Implemented |

# Context

The [[adr-195-critical-chain-buffer]] step rendered the **"Critical Chain & Project Buffer"** view as a section on the Decision Records Overview, directly below the work-item Gantt. The overview is already a dense page — the status charts, the group-segmented Gantt ([[adr-201-gantt-group-segmentation]]), and the records table — and the critical-chain view grows with the project: one block per decision group, each carrying its own chain table, buffer, and projected duration. As more groups gain estimates, that section pushes the records table far down the page.

The planning view is a distinct concern from the records listing and deserves its own home. Moving it to a dedicated page keeps the overview focused on the records and their charts, gives the chain view room to grow, and makes it directly linkable from the top menu. This is an independent presentation refinement over the committed [[adr-195-critical-chain-buffer]] baseline; the chain/buffer **computation is unchanged** — only where it is rendered and how it is reached.

# Decision

Move the Critical Chain & Project Buffer view off the overview and onto a dedicated page reachable from the top menu.

1. **New page document type.** Extract the section renderer — `render_critical_chain` and its helpers `critical_chain_block`, `cc_chain_html`, `cc_chain_row`, and `format_days` — from `decisions_overview.rb` into a new `CriticalChainPage` document type (a `BaseDocument` subclass, modelled on `DecisionsOverview`), rendered to `build/decisions/critical-chain.html`. The renderer body is unchanged; it still iterates `grouped_work_items` and builds a `CriticalChain` per group.
2. **Remove from the overview.** Drop the `html_rows.append render_critical_chain` line from `DecisionsOverview#to_html`. The status charts, the Gantt, and the records table remain on the overview untouched.
3. **Render in the pipeline.** Add a `render_critical_chain_page` step in `project.rb` alongside `render_decisions_overview`, gated by `@project_data.decisions.any?` (so the page exists exactly when the overview does) and writing to `build/decisions/`.
4. **Top-menu link.** Add a `critical_chain_link` to `base_document.rb`, emitted in the `{{HOME_BUTTON}}` block **immediately after** `decisions_link`, gated by the same `BaseDocument.show_decisions_link` flag. Its title is **"Critical Chain"** and it links to `decisions/critical-chain.html`. The resulting menu order is Home/Index → **Decision Records → Critical Chain** → (remaining items).
5. **Styling.** The existing `.critical_chain` CSS in `templates/css/main.css` is unchanged; it now styles the dedicated page.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Done | 18-06-2026 | 18-06-2026 | Add a `CriticalChainPage` (`BaseDocument` subclass) holding the moved `render_critical_chain` / `critical_chain_block` / `cc_chain_html` / `cc_chain_row` / `format_days` renderer, output to `build/decisions/critical-chain.html`; remove the `render_critical_chain` call from `DecisionsOverview#to_html`; add `render_critical_chain_page` in `project.rb` (guarded by `@project_data.decisions.any?`) and wire it into rendering; add `critical_chain_link` in `base_document.rb`, emitted in the `{{HOME_BUTTON}}` block right after `decisions_link` and gated by `show_decisions_link`, titled "Critical Chain" and pointing at `decisions/critical-chain.html` |
| Tests | Done | 18-06-2026 | 18-06-2026 | E2E tests under `spec/e2e/decisions_spec.rb`: the Decision Records Overview no longer contains a `div.critical_chain`; `build/decisions/critical-chain.html` renders the per-group chains, buffers, and projected durations (and the unestimated note); the top menu contains a `#critical_chain_menu_item` link titled "Critical Chain" positioned immediately after the `#decisions_menu_item` link; the link is absent when the project has no decision records |

# Out of Scope

- **The chain/buffer computation.** `CriticalChain` and the per-row estimate model ([[adr-195-critical-chain-buffer]]) are unchanged; this is a rendering-location and navigation change only.
- **The Gantt and its Buffer lane.** The group-segmented Gantt ([[adr-201-gantt-group-segmentation]]) stays on the overview; only the standalone chain view moves.
- **New planning data or visual redesign** of the chain view itself; the markup and CSS are carried over as-is.

# Consequences

## Positive

- The overview returns to a focused records-plus-charts page; the planning view gets a dedicated, linkable home that can grow without crowding the listing.
- Low risk: the renderer and CSS move verbatim and the computation is untouched, so the change is structural (a page + a menu link) rather than behavioural.

## Negative

- Adds a new document type, a new generated page, and a new menu item to maintain.
- Any external bookmark pointing at the overview's critical-chain section anchor will need to target the new page instead.

## Neutral

- The page is omitted when the project has no decision records, exactly as the overview is, so empty projects are unaffected.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall render the per-decision-record-group critical chain, project buffer, and projected duration on a dedicated Critical Chain page rather than on the Decision Records Overview, indicating when a group has no estimated work. | >[SRS-127] |
| 2 | The software shall place a "Critical Chain" link in the top navigation menu immediately after the Decision Records link, pointing at the dedicated Critical Chain page, and shall show it exactly when the Decision Records link is shown. | >[SRS-146] |

# References

- [[adr-195-critical-chain-buffer]] — introduced the Critical Chain & Project Buffer view (as an overview section) this enhancement relocates; its computation is reused unchanged
- [[adr-201-gantt-group-segmentation]] — the group-segmented Gantt and Buffer lane that remain on the overview
- [[adr-170-introduce-decision-records]] — introduced the Decision Records Overview, the page/menu structure, and the top-navigation links this extends

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
