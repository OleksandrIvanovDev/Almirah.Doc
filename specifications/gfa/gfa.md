# Goldratt Flow Analysis

## Overview

*Almirah as an ALM: Project Planning & Tracking through the Lens of "Goldratt's Rules of Flow"*

Initial Analysis and enhancement proposal made before the work on release 0.4.3 actually started. Alalysis was made with Claude AI (Opus model). Authored 2026-06-13. Further development may differ from this analyais.

> Lens: *Goldratt's Rules of Flow* (Efrat Goldratt-Ashlag) — Critical Chain / Theory of Constraints applied to project flow.

> This is a strategy/advisory note, not a formal decision record. It is intended as input to future ADRs under `decisions/`.

## 1. What Almirah is today, seen through a "flow" lens

Almirah is a **traceability-first ALM**: Markdown specs/tests/source compiled into interlinked HTML, with a hard rule that *the document is the single source of truth* — attributes like `start_date`, `target_date`, `current_status` are **derived during parsing** from tables already in the file. ADR-191 is explicit that a hand-maintained frontmatter field "duplicates information and creates a third place that can drift."

What is striking from a Goldratt / Critical-Chain standpoint is **how much planning substrate already exists**, almost accidentally:

- A **Decision record's `# Scope` table** is already a work breakdown — rows of work items (`Requirements / Code / Tests`) with `Status`, `Start Date`, `Target Date`, `Description`. That is one decomposition step away from a task list with estimates.
- The **`# Status` table** is a lifecycle event log with dated transitions, and `effective_status_on(date)` (`Almirah.Code/lib/almirah/doc_types/decision.rb`) already reconstructs state at any past date — the engine for any burn-up / flow chart.
- The **overview** (`Almirah.Code/lib/almirah/doc_types/decisions_overview.rb`) already renders a velocity chart (status-over-time, stacked) and a status distribution — i.e. it is already doing rudimentary throughput reporting.
- **Traceability links** (spec→spec uplinks, test→spec coverage, source→spec) are a real dependency graph — the raw material for a task network.

So the gap is **not** "build a planner from scratch." It is: *the data model records **state and dates** but not **work, capacity, dependency, or effort** — and it has no concept of a constraint or a buffer.* That is exactly where Rules of Flow has something to say.

**Where it is strong (keep):** single source of truth, derived metrics, git-native, no external DB, time-travel via `effective_status_on`.

**Where it is blind (the flow gaps):** no estimate vs. actual, no resource/owner load (the `Owner` column is a literal empty placeholder in `decisions_overview.rb`), no dependency sequencing, no WIP / multitasking signal, no buffer, due-dates treated as hard per-task targets (the anti-pattern CCPM warns against).

## 2. The Rules of Flow → ALM mapping

The five rules from Goldratt-Ashlag's book, translated into features Almirah could actually derive:

| Rule of Flow | What it means | Almirah feature it implies |
|---|---|---|
| **1. Bad multitasking is the #1 killer** — limit open work | Fewer things In-Progress at once finish faster | A **WIP signal**: count of `In-Progress` records per owner; flag when a person/release exceeds a freeze limit |
| **2. Full kit** — don't start until prerequisites are ready | Starting half-kitted work guarantees re-multitasking | A **readiness gate**: a record can't move to `In-Progress` until its up-links (input specs, prior decisions, test data) are themselves done |
| **3. Stagger against the constraint** | Release work at the pace of the most-loaded resource | A **load/capacity view** keyed off `Owner`; stagger start dates so the constraint isn't overloaded |
| **4. Aggressive estimates + a project buffer** | Strip per-task safety; protect the *chain*, not each task | **Two-number estimates** (aggressive + safe) and a derived **project buffer**, not padded per-task `Target Date`s |
| **5. Manage by buffer consumption** | Prioritize by what threatens the buffer (fever chart) | A **fever chart**: % chain complete vs. % buffer consumed, per release |

The elegant part: **all five can be expressed as new Markdown columns / tables + derived attributes**, staying true to the "no third place that drifts" principle in ADR-191.

## 3. Concrete proposal

Split into **low-level** (inside one decision record / one release) and **high-level** (portfolio across releases), because Almirah already has those two altitudes: the *Scope table* (low) and the *Decisions Overview* (high).

### A. Data model — extend the Scope table, add nothing external

Today's Scope row: `Item | Status | Start Date | Target Date | Description`.

Proposed columns (all optional, all parsed by header text so old records keep working — same compatibility discipline as adr-191):

```
| Item | Owner | Depends On | Est (focused) | Est (safe) | Actual | Status | Start Date | Target Date | Description |
```

- **`Owner`** — finally fills the empty Owner column and unlocks per-resource load.
- **`Depends On`** — references other Scope items / decision IDs (`>[ADR-178]`, or `#Requirements`). This builds the **task network** Critical Chain needs, reusing the existing link syntax and `link_registry.rb`.
- **`Est (focused)` / `Est (safe)`** — the two-estimate CCPM model (rule 4). The *focused / aggressive* number drives the schedule; the difference between safe and focused, aggregated, becomes the buffer.
- **`Actual`** — actual effort logged (tracking). Could be a single number, or — more git-native — derived from dated entries (see C).

A new derived attribute `chain` on `Decision` computes the **critical chain** through the Scope items (longest dependency + resource path), and `buffer = Σ(safe − focused)/2` along that chain (CCPM's 50% buffer heuristic). All derivable in a `decision.rb` method exactly like `extract_target_date`.

### B. High-level (portfolio / release planning) — extend the Overview

The Decisions Overview is the portfolio board. Add three derived views:

1. **WIP / multitasking heatmap (Rule 1).** Group `In-Progress` records by `Owner`; render a bar of open-task count per person with a configurable freeze line (e.g. `wip_limit: 2` in `project.yml`). This is the single highest-leverage report — it makes bad multitasking *visible*, which the book argues is 90% of the fix. Reuses the existing chart grid in `render_charts_grid`.

2. **Constraint load chart (Rule 3).** Sum focused estimates of not-yet-Done work per `Owner` over the next N weeks (reuse `recent_fridays`-style bucketing). The most-loaded owner *is* the constraint; staggering release start dates against that bar is the planning act.

3. **Full-kit readiness column (Rule 2).** For each record, walk its up-links; mark **"kitted"** only if every prerequisite record / spec is `Done` / `Implemented`. A record that is `In-Progress` but *not* kitted is a flow defect — surface it like a broken link (Almirah already has a "Check" pass for dangling references; this is the same machinery applied to readiness).

### C. Low-level (within a record) — estimation & actual effort

- **Estimation:** the two-estimate columns above. The rendered record page shows the critical chain and the computed buffer, so the author sees *"focused path = 9 days, buffer = 4 days, commit date = 13 days from kit-complete"* rather than hand-typing a `Target Date` per row.
- **Actual effort tracking, git-native:** rather than a mutable `Actual` cell, add an **effort log table** that is append-only — which matches how the Status table already works:

```
# Effort
| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 11-06-2026 | Code | A.I. | 3 | parser scaffolding |
```

`collect_dates`-style parsing already exists; summing `Hours` per item gives actuals, and the daily series feeds a real burn-up. This keeps the "single source of truth, derived, append-only, diffable in git" philosophy intact.

### D. The payoff report — fever chart (Rule 5)

Per release, derive **% critical-chain complete** (from Effort actuals vs. focused estimates) against **% buffer consumed** (elapsed buffer vs. total buffer), and plot the point on a red / yellow / green fever chart. This replaces "are we past the per-task Target Date" (the padded-due-date thinking the book attacks) with "is the *release* buffer healthy." It is the same `effective_status_on` time-machine you already have, just measuring buffer instead of status counts.

## 4. Suggested sequencing (and where it lands in the repos)

Per the workspace routing rules, every step is an ADR in `Almirah.Doc/decisions/` first, then code in `Almirah.Code`, then fixtures in `Almirah.TDS`. Phased to deliver flow value early:

1. **`Owner` + WIP heatmap** — smallest change, biggest behavioral payoff (makes multitasking visible). One column already reserved; one chart in the existing grid.
2. **`Depends On` + full-kit readiness check** — reuses link_registry + the Check pass.
3. **Two-estimate columns + critical chain + project buffer** — the CCPM core.
4. **Effort table + fever chart** — actuals and buffer management.

Each is independently shippable and each maps to one Rule of Flow, so you can dogfood it on Almirah's own decision records (`Almirah.Doc`) as you go — which is the fastest way to find out whether the model is right.

## 5. PM caution

The book's central message is that **the constraint is rarely the tool — it is the policy of starting too much work.** If only one thing were built, it should be **#1 (the WIP heatmap)**: ship it, and watch whether your own decision-record throughput improves *before* investing in the heavier critical-chain machinery. Estimation and buffers are worth far less until multitasking is under control.
