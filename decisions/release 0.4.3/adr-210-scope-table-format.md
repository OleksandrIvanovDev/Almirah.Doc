---
title: "ADR-210: Canonical Decision-Record Scope Table"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-06-2026 | Proposed |
|   | 21-06-2026 | Accepted |
|   | 21-06-2026 | In-Progress |
| * | 21-06-2026 | Implemented |

# Context

The `# Scope` table of a decision record grew one column at a time as planning features landed: the Owner column with the WIP heatmap ([ADR-193](./adr-193-owner-wip-heatmap.md)), the `#` step and `Depends On` columns with full-kit readiness ([ADR-194](./adr-194-full-kit-readiness.md)), and the `Est (focused)` / `Est (safe)` columns with the critical chain ([ADR-195](./adr-195-critical-chain-buffer.md)). Records authored before each step never gained the later columns, so the corpus drifted into four different Scope layouts.

The parser tolerates this: a Scope table is addressed by header *name*, not column position (see `ScopeTable#locate_columns`), so a missing column simply disables its feature for that record and nothing breaks. The cost is consistency, not correctness — a reader comparing two records meets two different tables, an author copying an older record starts from an incomplete layout, and the scaffold that `almirah create` writes (`project_template.rb`) was itself still on the oldest layout. There was no single statement of what a Scope table should contain.

A survey of the 40 records found:

- 27 on the minimal layout (`Item`, `Status`, `Start Date`, `Target Date`, `Description`);
- 2 with `#` and `Owner` added;
- 4 with `Depends On` added as well;
- 7 already on the full layout.

Crucially, every layout's columns are a subsequence of the full layout in the same order, so the variants differ only by *which* columns are present, never by their order.

# Decision

Adopt one canonical Scope table for every decision record, with these ten columns in this exact order and spelling (the spelling matters — the parser matches headers literally):

```
| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
```

- **#** — the 1-based row number, giving each row a per-record anchor a dependent record can deep-link to (ADR-194).
- **Item** — the activity, drawn from the workflow vocabulary the scheduler understands (`Analysis`, `Requirements`, `Code`, `Tests`).
- **Owner** — the resource the row's bar lands on (ADR-193, SRS-107–112).
- **Depends On** — cross-record prerequisites as `>[ID]` links (ADR-194, SRS-116).
- **Est (focused)** / **Est (safe)** — the per-row scheduling and safety estimates feeding the critical chain and project buffer (ADR-195, SRS-125).
- **Status** — the bounded per-row state (`To Do` / `In-Progress` / `Done`).
- **Start Date** / **Target Date** — the row dates that, with the Status table, fix the record's start and target (SRS-061, SRS-102).
- **Description** — free text; display only.

The canonical table is the union of columns each already individually required by an existing requirement; this ADR consolidates them into one mandated layout rather than introducing new behaviour.

**Alignment is structural, not a data backfill.** Because every prior layout is a column-subsequence of the canonical one, aligning a record means *inserting* the missing columns in place and leaving every existing cell untouched. Cells with no known value are left blank: the parser reads a blank estimate as zero, so a structurally-aligned record behaves exactly as it did before (an estimate-less record stays "unestimated" and out of the critical chain). No owners, dependencies, or estimates are invented for historical records; back-filling real values remains an opt-in, per-record exercise.

This decision covers three actions, all carried out here:

1. The scaffold `project_template.rb` now writes the canonical Scope table.
2. All 33 non-canonical records were migrated to the canonical columns, preserving existing cells and numbering rows.
3. An illustrative Scope table embedded inside a fenced code block in [ADR-187](./../release%200.4.1/adr-187-template-decisions-example.md) is left as written: it is historical narrative showing the template of its time, not a live Scope section.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 1 | 2 | Done | 21-06-2026 | 21-06-2026 | State the canonical ten-column Scope layout, its order and exact header spelling, and the structural (no-backfill) alignment rule; confirm each column is already required individually (Owner/Status SRS-107–112, Depends On SRS-116, estimates SRS-125, dates SRS-061/SRS-102) |
| 2 | Code | DEV |  | 2 | 3 | Done | 21-06-2026 | 21-06-2026 | Update the Scope example in `project_template.rb`; structurally align all 33 non-canonical records by inserting the missing `#`/`Owner`/`Depends On`/`Est` columns as blanks while preserving every existing cell, leaving ADR-187's fenced illustrative table as historical |
| 3 | Tests | TEST |  | 1 | 2 | Done | 21-06-2026 | 21-06-2026 | Rebuild `Almirah.Doc` and confirm all live Scope headers are identical and no new broken links or parse errors appear; run the `Almirah.Code` suite for the template change |

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 21-06-2026 | Code | DEV | 1 | canonical Scope example in `project_template.rb` |
| 21-06-2026 | Code | DEV | 2 | scripted column-insertion migration of 33 records |
| 21-06-2026 | Tests | TEST | 1 | `Almirah.Doc` rebuild check and `Almirah.Code` suite |

# Out of Scope

- **Back-filling owners, dependencies, or estimates** into historical records. Alignment is structural only; real values stay an opt-in, per-record exercise.
- **Adding matching `# Effort` tables** to the migrated records. The fever chart needs effort history, but reconstructing it for past records is not part of this layout alignment.
- **A new requirement pinning the column set.** The columns are already required individually by the SRS items above; this ADR consolidates them rather than adding a requirement.
- **Parser changes.** Header-name addressing and blank-tolerance are unchanged; no code path is modified beyond the scaffold template string.
- **The fenced illustrative Scope table in ADR-187**, left as historical narrative.

# Consequences

## Positive

- Every record now presents the same Scope table, so they read and compare uniformly and a new record copied from any existing one starts complete.
- The scaffold teaches the full layout, so projects created from the template are canonical from the first record.
- Migration touched only Scope-table rows (33 files, symmetric insert/delete), inventing no data, so behaviour is unchanged for every record.

## Negative

- Aligned records carry blank `Owner` / `Depends On` / estimate cells, which is visually emptier than a populated row and could read as "missing data" rather than "not applicable".
- The corpus now implies a richer plan than these older records actually have, since the columns exist without values.

## Neutral

- A structurally-aligned record stays out of the WIP heatmap, critical chain, and Gantt placement exactly as before, because blank cells carry no owner or estimate.
- The fenced example in ADR-187 remains on the old layout by design, so a header-name survey that ignores code fences will still find one non-canonical table.

# Alternatives Considered

- **Full data backfill (reconstruct owners, dependencies, estimates, and effort for all records).** Rejected for now: the values would be invented history, a large effort for little gain on closed records. Kept available as opt-in per record.
- **Leave the layouts as they are.** Rejected: the drift is the problem the user raised; the parser's tolerance hides it but does not remove the inconsistency for readers and authors.
- **Introduce a new SRS requirement defining the columns.** Rejected as redundant: each column is already required by an existing item; an ADR consolidating them is the lighter, non-overlapping record.
- **Also rewrite the fenced ADR-187 example.** Rejected: it documents the template as it was when that ADR was decided; editing it would falsify the historical record.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [ADR-193](./adr-193-owner-wip-heatmap.md) — introduced the Owner column and the per-row Status
- [ADR-194](./adr-194-full-kit-readiness.md) — introduced the `#` step and `Depends On` columns and the `ScopeTable`
- [ADR-195](./adr-195-critical-chain-buffer.md) — introduced the `Est (focused)` / `Est (safe)` columns
- [ADR-187](./../release%200.4.1/adr-187-template-decisions-example.md) — the scaffold decision record whose template this ADR brings to the canonical layout
- [[enh-209-buffer-utilization-visibility]] — the most recent record already authored on the canonical layout

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
