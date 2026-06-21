---
title: "ADR-177: Pie Chart of Decision Record Types on Overview Page"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 20-05-2026 | Proposed |
|   |  | Accepted |
|   |  | In-Progress |
| * |  | Implemented |

# Context

The Decision Records Overview page (`build/decisions/overview.html`) lists every decision record as a single table. As the project accumulates records the table grows, and a reader has no top-of-page summary of *what kinds* of decisions exist or in what proportion.

There is currently no chart rendering capability anywhere in Almirah — all rendered outputs are HTML + CSS over Markdown-derived content. Introducing visualization is an architectural step (new external JS dependency), which is why this is an ADR rather than an ENH.

# Decision

Add a pie chart above the table on the Decision Records Overview page that visualizes the count of decision records grouped by their type prefix (`ADR`, `ENH`, `ISSUE`).

Architectural choices:

1. **Charting library: Chart.js.** Loaded from a public CDN (`https://cdn.jsdelivr.net/npm/chart.js`), following the existing precedent of loading Font Awesome from a CDN in `templates/page.html`. No build step is added; no JS bundler is introduced. Chart.js is included only on the Decision Records Overview page, not on every rendered page.
2. **Page layout: 3-column CSS grid above the table.** A new container element above `<table class="controlled decisions_overview">` shall use `display: grid; grid-template-columns: repeat(3, 1fr)`. The pie chart shall occupy the **leftmost** column. The middle and right columns shall be empty placeholders, reserved for two further charts that are out of scope of this ADR. The grid is the architectural commitment; the additional charts are not.
3. **Data source: in-renderer aggregation.** `DecisionsOverview#to_html` shall count records per `record_type` from `@project.project_data.decisions` and emit the counts as a JSON literal inside an inline `<script>` tag adjacent to the `<canvas>`. No new data pipeline, parser, or AJAX call is introduced.
4. **Rendering trigger: inline script on the page.** A small inline `<script>` instantiates `new Chart(ctx, { type: 'pie', data: ... })` against a `<canvas id="decisions_type_pie">`. No shared chart-rendering helper is extracted yet — that abstraction is deferred until the second chart lands.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Code | DEV |  |  |  | Implemented | 20-05-2026 | 23-05-2026 | Implementation |

No new requirements documents, test protocols, or end-to-end tests are introduced — this is a presentation change and is verified by visual inspection of the rendered overview page.

# Out of Scope

- The middle and right cells of the 3-column grid.
- Self-hosting Chart.js inside the gem. CDN loading is sufficient for now; vendoring can be revisited if offline-build support is later required.
- Interactivity beyond Chart.js defaults (legend toggle, hover tooltip). No custom click handlers or drill-down behavior.
- Color theming, dark-mode support, or accessibility-specific patterns (e.g., text-equivalent of the chart). Defaults inherited from Chart.js.
- Charting on other index pages (Specifications Index, Traceability Matrices, etc.).
- A reusable chart-rendering abstraction in Ruby or JS — deferred until the second chart exists. See [[adr-170-introduce-decision-records]] precedent of introducing structure only when a second instance demands it.

# Consequences

## Positive

- Readers get an at-a-glance proportional view of decision record types before scrolling the table.
- Establishes a sanctioned charting library (Chart.js) for the framework, so future visualizations have a clear default rather than re-litigating the choice.

## Negative

- First external JS dependency beyond the CDN-loaded Font Awesome icon font. Page weight on the overview page increases by the size of Chart.js (~70 KB gzipped).
- CDN dependency introduces a network requirement to render the chart; if the CDN is unreachable, the table still renders but the chart cell is blank.
- Inline aggregation script in `DecisionsOverview` couples the Ruby renderer to a specific JS API surface (`new Chart(...)`); a future library swap touches both Ruby and JS.

## Neutral

- Chart.js is loaded only on the overview page, so the cost is not paid by Specification, Test Protocol, or other rendered pages.

# Alternatives Considered

- **Render the chart as a static SVG at build time (e.g., via a Ruby SVG library).** Rejected: introduces a Ruby-side charting dependency, gives up interactivity (hover tooltips, legend toggle), and produces a heavier git diff on every count change. Chart.js client-side rendering is simpler.
- **Inline a small hand-rolled SVG pie chart with no library.** Rejected: viable for a single chart but does not generalize to the two follow-on charts the grid reserves space for. A library decision made once is better than three bespoke implementations.


# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records and the overview page
- [ENH-173](./enhancements/enh-173-decisions-table-view.md) — established the `decisions_overview` CSS class on the table and the precedent of reserving empty visual space for future content
- [ENH-175](./enhancements/enh-175-overview-sort-id.md) — sort order of the overview table rows
- Chart.js — `https://www.chartjs.org/`

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
