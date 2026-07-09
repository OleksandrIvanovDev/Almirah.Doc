---
title: "ADR-222: Remove Critical-Chain Planning Layer"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

Release 0.4.3 built a Critical-Chain Project Management layer on top of decision records, across fourteen records: the WIP-by-Owner heatmap ([[adr-193-owner-wip-heatmap]]), the work-item dependency network and full-kit gates ([[adr-194-full-kit-readiness]]), estimates, the critical chain and project buffer ([[adr-195-critical-chain-buffer]]), effort-driven buffer consumption and the fever chart ([[adr-196-buffer-fever-chart]]), and the resource-swimlane Gantt ([[adr-198-workitem-gantt-visualization]]) with its refinement train ([[adr-201-gantt-group-segmentation]], [[adr-204-consensus-owner-order]], [[adr-205-calendar-gantt-view]], [[adr-206-working-day-columns]], [[adr-211-group-start-dates]], [[adr-212-critical-chain-highlight]], [[adr-213-gantt-actuals-lanes]], plus [[enh-199-gantt-separator-lines]], [[enh-200-gantt-status-styling]], [[enh-202-critical-chain-page]], [[enh-208-gantt-dependency-tooltip]], [[enh-209-buffer-utilization-visibility]], [[issue-207-fever-completed-overrun]]).

Living with the layer through the 0.4.4 cycle showed that it overcomplicates the gem and reads ambiguously:

- The Gantt alone needed seven corrective or refining ADRs within one release, and the codebase carries two different critical-chain definitions — the Critical Chain page's and the Gantt scheduler's — that ADR-212 had to reconcile.
- Every bar on the chart is a projection built from estimate columns that nothing else consumes, over a planning network whose kit semantics (activity-type-aligned fallback resolution, in-group versus cross-group edges) take longer to explain than the schedule saves.
- The footprint is out of proportion to its value: roughly 1,900 lines of production code (eight dedicated source files plus about 470 of the overview renderer's 763 lines), 51 of the SRS's 170 stated requirements, four unit spec files and about a hundred e2e examples, and a large planning block of CSS — around a quarter of the gem — serving one feature no other collection touches.

Almirah's core value is the traceability graph over specifications, tests, source, risks, and decisions. Project scheduling is a different product's job; the parts of the layer worth keeping are the authoring conventions the records themselves use.

# Decision

Remove the planning computation and visualization entirely; keep the Scope-table and Effort authoring conventions so no existing document needs editing.

- **Remove the computation and views.** The WIP-by-Owner chart and its freeze line, the Kit column and the kit console warnings, the WorkItem dependency network and its resolution rules, the estimate semantics, the deterministic scheduler, the critical chain, the project buffer, the effort computation (totals, as-of-date sums, per-row sums), the fever chart, the Critical Chain page and its top-menu entry, the whole Gantt (segmentation, consensus lane order, calendar and business-day axis, chain highlight, tracking lanes and toolbar), the working calendar, and the entire planning block of project.yml configuration (wip_limit, buffer_ratio, hours_per_day, start_date, groups, holidays).
- **Restore the pie chart.** ADR-193 replaced the overview's Decision Records by Type pie chart with the WIP chart and deliberately kept the pie builder unrendered for rollback; the pie returns to the left chart slot. The velocity and status charts (ADR-191) are untouched.
- **Keep the authoring conventions.** The canonical Scope table format ([[adr-210-scope-table-format]]) remains the documented convention, and the ScopeTable renderer is slimmed to presentation only: header-addressed cells, the anchored step-number column, clickable Depends On references, and the date-cell styling of [[enh-214-scope-date-nowrap]]. The Owner column on the overview keeps rendering each record's distinct Scope owners (SRS-107 through SRS-110 stay). The overview Target Date (SRS-102) still reads the Status table and the Scope Start and Target Date columns. The Effort section remains a documented convention rendered as a plain table; only its computation goes.
- **Requirements.** SRS-113 and SRS-115 are rewritten as rendering-only statements; SRS-111, SRS-112, SRS-114, SRS-116 through SRS-155, and SRS-158 through SRS-165 are deleted. Deleted identifiers are retired permanently and never reassigned.
- **Supersede, do not rewrite, history.** Each of the eighteen release 0.4.3 planning records listed above, plus ISSUE-220 (a 0.4.4 fix to the removed scheduler), gets one new Status row, Superseded; their content stays untouched as history. ADR-197 (decision groups), ADR-210 (Scope format), ENH-214 (date styling) and ISSUE-203 are not planning records and keep their statuses.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | Rewrite SRS-113 and SRS-115 as rendering-only statements; delete SRS-111, SRS-112, SRS-114, SRS-116..155 and SRS-158..165; keep SRS-107..110 and SRS-102 unchanged |
| 2 | Code | DEV |  | 2 | 4 | Done | 09-07-2026 | 09-07-2026 | Unhook the Critical Chain menu link, page rendering, kit reporting and work-item linking from the pipeline; delete the scheduler, critical-chain, fever-chart, working-calendar, work-item, decision-grouping and planning-configuration code and the planning CSS; slim ScopeTable to cells, anchors and dependency-link rendering; restore the retained pie chart in the overview's left slot |
| 3 | Tests | TEST |  | 2 | 3 | Done | 09-07-2026 | 09-07-2026 | Delete the six planning unit spec files; excise the scheduler-tagged examples from the decisions e2e spec while keeping owner, target-date, chart and grouping coverage plus new slimmed Scope-rendering coverage; full suite and Rubocop green; rebuild Almirah.Doc as the real-world regression check |
| 4 | Documents | BA |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | Add a Superseded status row to the eighteen planning records of release 0.4.3 and to ISSUE-220; drop the planning block from the Almirah.Doc project.yml |

# Out of Scope

- **The Scope table format.** ADR-210's ten columns, including both estimate columns and both date columns, stay the documented authoring convention; existing records keep rendering unchanged.
- **The Effort convention.** Records keep logging effort; the table renders as authored content.
- **The Owner column and Target Date on the overview.** Both stay; they are record metadata, not scheduling.
- **The velocity and status charts.** Pre-date the planning layer (ADR-191) and stay.
- **Renumbering.** Deleted SRS identifiers are retired, not reused; remaining requirements keep their numbers.

# Consequences

## Positive

- Roughly a quarter of the gem's production code and 51 of 170 requirements disappear, with the decisions overview renderer shrinking to a fraction of its size; the surface a contributor must understand contracts accordingly.
- One planning concept remains — the record lifecycle status plus dated Scope rows — instead of four overlapping ones (kit, chain, buffer, schedule).
- The removal is invisible to documents: every existing record renders as before, minus the charts and columns that summarised it.

## Negative

- All planning visualization is lost; a project that relied on the Gantt or the fever chart must plan elsewhere. The record trail (this record and the superseded ones) preserves the design should a leaner successor ever be wanted.
- A project.yml carrying a planning block gets no warning; the key is simply ignored, as unknown configuration keys always are.
- The superseded records' Affected Documents tables still uplink the retired requirements; those links now point at anchors that no longer exist on the SRS page. Accepted: superseded content stays untouched, and the retired identifiers are never reassigned, so the links can never silently point at the wrong requirement.

## Neutral

- The 0.4.3 records remain Implemented history with an appended Superseded row — the register stays honest about both what was built and what was withdrawn.
- Release 0.4.5 becomes a removal release: no new behaviour, less of it.

# Alternatives Considered

- **Keep and stabilise the layer.** Rejected: the ambiguity is structural, not a defect backlog — two chain definitions, projection-quality inputs, and semantics that need a book reference to follow. Stabilising it means maintaining all of it indefinitely.
- **Extract it into an optional gem or plugin.** Rejected: Almirah has no plugin seam today, and building one to preserve a feature being removed for overcomplication inverts the goal. The record trail preserves the design at zero carrying cost.
- **Remove the conventions too (full rollback to the 0.4.2 Scope handling).** Rejected: every 0.4.x record authored since carries step numbers, Depends On references, estimates and dates; regressing the renderer would turn those cells into raw text across the whole register.
- **Keep the Gantt as a plain visualization without the CCPM math.** Rejected: the bars are the schedule; without the scheduler there is nothing truthful to draw, and the calendar machinery is most of the weight.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.4 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.5 |

# Affected Documents

This decision rewrites two requirements and deletes fifty-one. Kept unchanged: SRS-102, SRS-107, SRS-108, SRS-109, SRS-110. Deleted (retired, listed as text to avoid dangling references once removed): SRS-111, SRS-112, SRS-114, SRS-116 to SRS-155, and SRS-158 to SRS-165.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The Decision Record Scope table shall support a leading step-number column rendered as an anchored, centered row number so that individual Scope rows can be linked; when the column is absent, the intrinsic row order applies. | >[SRS-113] |
| 2 | The Decision Record Scope table shall support a per-row Depends On column, identified by header text and not position, whose references to other Decision Records render as clickable links to those records. | >[SRS-115] |

# References

- [[adr-193-owner-wip-heatmap]], [[adr-194-full-kit-readiness]], [[adr-195-critical-chain-buffer]], [[adr-196-buffer-fever-chart]] — the planning core being removed
- [[adr-198-workitem-gantt-visualization]], [[adr-201-gantt-group-segmentation]], [[adr-204-consensus-owner-order]], [[adr-205-calendar-gantt-view]], [[adr-206-working-day-columns]], [[adr-211-group-start-dates]], [[adr-212-critical-chain-highlight]], [[adr-213-gantt-actuals-lanes]] — the Gantt and its refinements
- [[enh-199-gantt-separator-lines]], [[enh-200-gantt-status-styling]], [[enh-202-critical-chain-page]], [[enh-208-gantt-dependency-tooltip]], [[enh-209-buffer-utilization-visibility]], [[issue-207-fever-completed-overrun]] — presentation refinements superseded with their subjects
- [[adr-210-scope-table-format]], [[enh-214-scope-date-nowrap]] — the authoring conventions that stay
- [[adr-191-overview-target-date]] — the pre-planning overview features restored to prominence

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/33)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/33)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/52)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/52)

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 08-07-2026 | Requirements | BA | 1 | Removal analysis across code, specs and SRS |
| 09-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 09-07-2026 | Requirements | BA | 0.5 | SRS planning section removed; SRS-113/115 rewritten |
| 09-07-2026 | Code | DEV | 2 | Planning layer removed; ScopeTable slimmed; pie restored |
| 09-07-2026 | Tests | TEST | 1.5 | Suite pruned 476 to 328 examples, green with Rubocop |
| 09-07-2026 | Documents | BA | 0.5 | 18 records superseded; planning block dropped |
| 09-07-2026 | Documents | BA | 0.25 | Reference audit: ISSUE-220 superseded; SRS-171 example re-pointed |
