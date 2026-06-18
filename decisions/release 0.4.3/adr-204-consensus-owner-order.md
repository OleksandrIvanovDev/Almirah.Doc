---
title: "ADR-204: Consensus Owner Order for Gantt Lanes"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 18-06-2026 | Proposed |
|   | 18-06-2026 | Analysis |
|   | 18-06-2026 | Accepted |
|   | 18-06-2026 | In-Progress |
| * | 18-06-2026 | Implemented |

# Context

The resource-swimlane Gantt ([[adr-198-workitem-gantt-visualization]], segmented per group by [[adr-201-gantt-group-segmentation]]) renders one lane per owner. The lane **order** — top to bottom — is currently `ordered_owners(in_progress_tally)` ([decisions_overview.rb:106](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L106)), the same helper the Work-In-Progress heatmap uses ([[adr-193-owner-wip-heatmap]]): owners sorted by **descending current in-progress count**, idle owners last in first-seen order.

That order is right for the heatmap — it answers "who is most loaded right now" — but it is the wrong axis for the Gantt. A Gantt reads top-to-bottom as a process, and the records themselves encode a process: a Scope table's rows run, in the overwhelming majority, `Analysis (BA) → Requirements (BA) → Code (DEV) → Tests (TEST)`. Ordering the lanes by today's WIP makes the swimlanes jump around between renders as work moves, and shows the workflow out of sequence (TEST above DEV whenever a tester happens to be busier). The lanes should instead follow the **workflow order the records collectively describe**, so the chart flows downward the way the work flows forward.

There is no place that records the canonical role order, and hand-listing one would be a second source of truth that drifts — exactly the anti-pattern [[adr-191-overview-target-date]] and [[adr-193-owner-wip-heatmap]] reject. The order is already implicit in the data: every record's Scope table is one vote on how roles are sequenced. Each `Decision` already exposes `owners` — its distinct, first-seen-ordered owner list ([[adr-193-owner-wip-heatmap]]) — so `[BA, DEV, TEST]` for the example above is already computed. What is missing is a way to aggregate those per-record sequences into one consensus order.

Records disagree only at the margins, and the disagreement is benign:

- A handful may run roles in an unusual order (a fix where TEST precedes DEV). These are negligible and should simply be **outvoted**, not detected and excluded.
- Many name only a subset (`[BA, DEV]`, no test row) or add a role (`[BA, TEST, DevOps]`). A missing role should cast no vote; an added role should be placed by whatever evidence exists, with the shared roles still agreeing on their relative order.

This is the classic **rank-aggregation / consensus-ranking** problem: many partial orderings of a small item set, one majority total order out. The exact optimum (Kemeny–Young — the order minimising total disagreement) is NP-hard and far more machinery than this warrants.

# Decision

Derive a **consensus owner order** from the per-record `owners` sequences using pairwise-majority precedence (a Condorcet tally) aggregated by **Copeland score**, and use it to order the **Gantt lanes only**. The Work-In-Progress heatmap keeps its descending-count order. The two charts answer different questions — the heatmap "who is loaded now", the Gantt "how does the work flow" — and so deserve different orderings.

The governing principle for this step is **optimise for clarity, not cycles**. The data is tiny — a few dozen records, a handful of distinct roles — so every aggregation method is effectively free; the only thing worth optimising is how plainly the code reads. We therefore pick the method that is simplest to read and that resolves any disagreement *implicitly*, and we deliberately do **not** build Kemeny–Young, explicit cycle detection, or graph cycle-breaking.

## The aggregate — pairwise precedence by all ordered pairs

Maintain a pairwise tally `before[[a, b]]` = the number of records in which role `a` appears before role `b`. For each record, take its `owners` sequence (already distinct and first-seen-ordered) and, for **every ordered pair** `(a, b)` with `a` earlier than `b`, increment `before[[a, b]]`. All pairs, not just adjacent ones: this is what makes a missing middle role harmless — `[BA, TEST, DevOps]` still votes `BA before DevOps`, so the shared roles agree even when one is skipped.

This tally is what makes the two marginal cases self-handling, with no special code:

- **Opposite orders are outvoted, not excluded.** A rare `TEST before DEV` record adds one vote to `before[[TEST, DEV]]`, swamped by the many `before[[DEV, TEST]]` votes. Nothing detects or skips it.
- **Missing roles abstain; added roles are placed by their own votes.** An absent role contributes no pairs; an added role (`DevOps`) is ranked only by the pairs it actually appears in.

## The order — Copeland score with a first-seen tiebreak

Rank each role by its **Copeland score**: across every other role `o`, `+1` when the role precedes `o` more often than it follows (`before[[role, o]] > before[[o, role]]`), `-1` when it follows more often, `0` when tied. Sort by descending score; break ties with the existing first-seen order so the result is **identical across runs** over unchanged input, matching the determinism the Gantt already guarantees ([[adr-198-workitem-gantt-visualization]], [[adr-201-gantt-group-segmentation]]).

Copeland — not a pairwise comparator passed to `sort` — is the clarity choice precisely *because* of cycles. Pairwise majorities can form a Condorcet cycle (A beats B, B beats C, C beats A); a comparator over such a relation is not transitive, so a plain sort would give an order that depends on input order and could differ between runs. Copeland assigns every role a single integer regardless of cycles, so the sort is always well-defined and stable. We pay nothing for this — it is one nested loop over a handful of roles — and in exchange the code never needs to *think* about cycles: a (negligibly rare, tiny) cycle simply perturbs two adjacent scores and the first-seen tiebreak settles it. That is "optimise for clarity, not cycles" made concrete: the simplest method that also happens to be cycle-safe, with zero cycle-handling code.

## Where it computes — the loop that already exists

`ordered_owners` already iterates `@project_data.decisions` once to build `first_seen` ([decisions_overview.rb:366](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L366)). Build the `before` tally inside that same pass — no new iteration over the records — and add a sibling `consensus_owner_order` method that consumes the tally and the `first_seen` roster. `render_workitem_gantt` calls `consensus_owner_order` for its lanes ([decisions_overview.rb:106](./../../../Almirah.Code/lib/almirah/doc_types/decisions_overview.rb#L106)); the heatmap's `ordered_owners(tally)` call is untouched. Per-record cost is O(k²) in that record's distinct-owner count k (k ≈ 4); aggregation is O(R²) in the role count R (R ≈ 5) — both negligible.

# Scope

| # | Item | Owner | Depends On | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA | >[ADR-193], >[ADR-201] | Done | 18-06-2026 | 18-06-2026 | This decision record: framing the Gantt lane order as a rank-aggregation problem; choosing pairwise-precedence + Copeland over Kemeny–Young / comparator-sort / explicit cycle-breaking under the "optimise for clarity, not cycles" principle; folding the tally into the existing `ordered_owners` record pass; scoping the new order to the Gantt lanes only while the WIP heatmap keeps descending-count order |
| 2 | Requirements | BA | >[ADR-193], >[ADR-201] | Done | 18-06-2026 | 18-06-2026 | New SRS items SRS-147 and SRS-148 in the `srs.md` "Planning" chapter: deriving a consensus owner order by tallying every ordered owner pair across each record's distinct owner sequence and ranking by Copeland score with a first-seen tiebreak for run-to-run stability; and ordering the Decision Records Overview Gantt lanes by that consensus order while the Work-In-Progress by Owner chart retains its descending-count order |
| 3 | Code | DEV | >[ADR-193], >[ADR-201] | Done | 18-06-2026 | 18-06-2026 | In `decisions_overview.rb`: add `owner_precedence`, a single pass over the records building the first-seen roster and a `before[[a, b]]` pairwise tally over all ordered pairs of each `doc.owners`; add a `consensus_owner_order` method ranking owners by `copeland_score` with the `first_seen` order as tiebreak; switch `render_workitem_gantt` to call `consensus_owner_order` for the lane order; leave the WIP heatmap's `ordered_owners(in_progress_tally)` call unchanged |
| 4 | Tests | TEST | >[ADR-193], >[ADR-201] | Done | 18-06-2026 | 18-06-2026 | E2E tests under `spec/e2e/decisions_spec.rb`: the Gantt lanes follow the majority workflow order (`BA → DEV → TEST`) when records agree; a minority opposite-order record does not flip the lanes; a record naming only a subset of roles does not displace the shared roles' relative order and an added role is placed after the roles it follows; the lane order is identical across two runs; and the WIP heatmap order is unaffected (still descending count) |

# Out of Scope

- **Reordering the Work-In-Progress by Owner chart.** It keeps its descending-count order ([[adr-193-owner-wip-heatmap]]); only the Gantt lanes adopt the consensus order.
- **The exact (Kemeny–Young) consensus.** Rejected as machinery the data scale does not justify; the Copeland approximation is the clarity choice. See *Alternatives Considered*.
- **Cycle detection or cycle-breaking.** Condorcet cycles are negligibly rare at this scale and are absorbed by Copeland scores plus the first-seen tiebreak; no code detects or breaks them.
- **Owner identity normalisation** (aliases, initials vs. full names). Owner strings are compared verbatim after trimming, as in [[adr-193-owner-wip-heatmap]].
- **A hand-listed or configured role order.** Rejected as a second source of truth that drifts; the order is derived from the Scope tables authors already maintain.
- **Ordering work items *within* a lane, or the per-group schedule.** Those remain the scheduler's concern ([[adr-198-workitem-gantt-visualization]], [[adr-201-gantt-group-segmentation]]); this ADR orders only the lanes themselves.

# Consequences

## Positive

- The Gantt reads top-to-bottom as the workflow the records collectively describe, instead of jumping around with today's WIP — a steadier, more legible chart.
- No new source of truth: the order is derived from the Scope tables, consistent with [[adr-191-overview-target-date]] and [[adr-193-owner-wip-heatmap]]; nothing to hand-maintain or let drift.
- The marginal cases (opposite orders, missing roles, added roles) are handled by the tally itself, so the code carries no special-case branches.
- Effectively free and deterministic: one extra nested loop over a handful of roles inside a record pass that already runs, with a stable tiebreak.

## Negative

- The lane order is only as coherent as the Scope tables; a project whose records genuinely disagree on role order gets a consensus that satisfies no single record (though it still satisfies the majority of pairs).
- A new reader may expect the Gantt lanes and the WIP heatmap to share an order; they now deliberately differ. The tests and this ADR record why.

## Neutral

- Already-rendered projects must be re-rendered to pick up the new lane order; nothing is retroactive.
- With one record, or with all records sharing one owner order, the consensus order equals the first-seen order — visually unchanged from before for simple projects.

# Alternatives Considered

- **Keep the WIP (descending-count) order for the lanes.** Rejected: it answers a different question (load, not flow), shows the workflow out of sequence, and reshuffles the lanes as work moves between renders.
- **Hand-list the canonical role order in `project.yml`.** Rejected: a second source of truth that drifts from the Scope tables, against the derive-don't-duplicate discipline of [[adr-191-overview-target-date]].
- **Kemeny–Young (the exact optimum).** Rejected under "optimise for clarity, not cycles": NP-hard in general and far heavier than the data warrants; Copeland gives the same answer in practice at this scale with code anyone can read.
- **A pairwise comparator passed to `sort`.** Rejected: the majority relation can contain Condorcet cycles, making the comparator non-transitive and the sort order input-dependent and unstable across runs. Copeland assigns each role a single score, so the sort is always well-defined and deterministic.
- **Counting only adjacent owner pairs.** Rejected: a skipped middle role would drop the long-range votes (`BA before DevOps` in `[BA, TEST, DevOps]`), weakening the consensus for exactly the partial records that need it. All ordered pairs cost the same at this scale and are more robust.
- **Detecting and excluding "wrong-order" records before aggregating.** Rejected as needless: pairwise majority already outvotes them, with no detection threshold to tune.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall derive a consensus owner order across all Decision Records by tallying, for each record's distinct owner sequence, every ordered pair of owners that appears earlier-before-later, and ranking the owners by Copeland score — each owner scoring one for every other owner it precedes more often than it follows, minus one for every owner it follows more often — with the first-seen owner order as a tiebreak so the result is identical across repeated runs over unchanged input. | >[SRS-147] |
| 2 | The Decision Records Overview Gantt shall order its resource lanes by the consensus owner order, while the Work-In-Progress by Owner chart shall retain its descending-in-progress-count order. | >[SRS-148] |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the planning/flow roadmap these Gantt and owner features serve
- [[adr-201-gantt-group-segmentation]] — the group-segmented Gantt whose owner lanes this reorders
- [[adr-198-workitem-gantt-visualization]] — introduced the resource-swimlane Gantt and its determinism guarantee
- [[adr-193-owner-wip-heatmap]] — supplies each record's `owners` list and the `ordered_owners` / descending-count order this leaves on the heatmap and reorders for the Gantt
- [[adr-191-overview-target-date]] — the derive-don't-duplicate discipline followed by deriving the order from the Scope tables rather than hand-listing it

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
