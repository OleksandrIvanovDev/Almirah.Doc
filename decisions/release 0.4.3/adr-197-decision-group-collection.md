---
title: "ADR-197: Decision Group Collection by Folder"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 15-06-2026 | Proposed |
|   | 16-06-2026 | Accepted |
|   | 16-06-2026 | In-Progress |
| * | 16-06-2026 | Implemented |

# Context

Decision records live under `decisions/` in first-level sub-folders that are themselves a planning artifact — `release 0.4.0`, `release 0.4.1`, `release 0.4.2`, `release 0.4.3`. Each folder is a **logical group of records planned together**. The parser already walks this tree (`parse_decisions` in [project.rb](./../../../Almirah.Code/lib/almirah/project.rb)) and computes each record's relative directory (`rel_dir`) purely to lay out the HTML output path — then **discards the grouping**. There is today no aggregation unit between a single `Decision` and the whole portfolio held in `@project_data.decisions`.

The planning/flow roadmap ([goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md)) needs that unit. Its later steps — the project buffer ([[adr-195-critical-chain-buffer]]) and the per-release fever chart ([[adr-196-buffer-fever-chart]]) — are inherently computed over a *set* of records sharing a delivery, not over one record. Before any of that machinery is built, the cheapest enabling step is to **collect the grouping that already exists on disk** and keep it as object references for future use.

This step deliberately introduces **no new source of truth**. Membership is derived from the folder layout the author already maintains, consistent with the derive-don't-duplicate discipline of [[adr-191-overview-target-date]] and [[adr-193-owner-wip-heatmap]]: nothing is hand-listed, so nothing can drift.

# Decision

Add a single new collection, **`@project_data.decision_groups`**, populated record-by-record inside `parse_decisions`. `@project_data.decisions` is **left exactly as it is** — `decision_groups` is an additional view over the same `Decision` object references, not a replacement.

## Shape

`decision_groups` is an **insertion-ordered list of single-key hashes**. Each hash maps a **group name** (a string) to the **list of `Decision` object references** that belong to it:

```
[ { "release 0.4.0" => [adr170, adr171, ...] },
  { "release 0.4.3" => [adr193, adr194, adr197, ...] } ]
```

The list-of-single-key-hashes form mirrors the house style already used for per-row predecessor/successor lists in [[adr-194-full-kit-readiness]]. Insertion order follows the order records are encountered during the directory walk, so groups appear in the order their first record is parsed.

## Group key

The group name is the **first-level sub-folder under `decisions/`** — the first path segment of the record's `rel_dir`. A record nested deeper (for example `release 0.4.3/issues/issue-5.md`) folds into its first-level parent, `release 0.4.3`. A record placed directly in `decisions/` (where `rel_dir` is `.`) is kept under the `.` group rather than dropped, so no record is silently lost.

## Population

`rel_dir` is already computed in the `parse_decisions` loop for `html_rel_path`; its first path segment is reused as the key. For each parsed record: find the existing single-key hash whose key matches the record's group; if none exists, append a fresh `{ group => [] }` (this is what preserves folder-encounter order); then push the `Decision` reference into that group's list. The rest of the loop — appending to `@project_data.decisions` and setting `html_rel_path` — is unchanged.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | BA |  |  |  | Done | 15-06-2026 | 16-06-2026 | This decision record: the data shape (insertion-ordered list of single-key hashes), the first-level-folder group key, the `.`-group handling for top-level records, and the decision to leave `@project_data.decisions` untouched |
| 2 | Code | DEV |  |  |  | Done | 16-06-2026 | 16-06-2026 | Add `@project_data.decision_groups` (initialized to an empty list, exposed via `attr_reader`) in [project_data.rb](./../../../Almirah.Code/lib/almirah/project/project_data.rb); in `parse_decisions` ([project.rb](./../../../Almirah.Code/lib/almirah/project.rb)) derive the group key as the first path segment of the already-computed `rel_dir`, find-or-append the matching single-key hash, and push the `Decision` reference into its list; leave the `@project_data.decisions` append and `html_rel_path` logic unchanged |
| 3 | Tests | TEST |  |  |  | Done | 16-06-2026 | 16-06-2026 | Unit tests ([spec/decision_groups_spec.rb](./../../../Almirah.Code/spec/decision_groups_spec.rb)): parsing a project groups each record under its first-level folder name; records in nested sub-folders fold into their first-level parent; a record directly under `decisions/` lands in the `.` group; group order follows folder-encounter order; every reference in `decision_groups` is the same object held in `@project_data.decisions` and the two collections hold the same total record count |

# Out of Scope

- **A `Release` class or any release-level configuration.** This step collects raw groupings only; naming the entity "release", giving it ordering/buffer config in `project.yml`, or attaching a committed delivery date is deferred to the steps that consume it ([[adr-195-critical-chain-buffer]], [[adr-196-buffer-fever-chart]]). The collection is intentionally named `decision_groups`, not `releases`.
- **Reconciling the folder with each record's `Target Release Version`.** A record's Software Versions table also names a target release; this ADR does not check the folder against it, nor report disagreement. That validation is deferred until a consumer needs it.
- **Any rendering or reporting.** `decision_groups` has no consumer yet — it is inert collection. The Decisions Overview is not changed, no new page is produced, and no console output is added.
- **New requirements.** Because there is no externally observable behavior, this step adds no SRS items; it is internal plumbing verified by unit tests.

# Consequences

## Positive

- The grouping that already exists on disk, and was previously discarded after computing the output path, is now retained as object references — the enabling substrate for the buffer and fever-chart steps, gathered at near-zero cost.
- No new source of truth and nothing hand-authored: membership derives from the folder layout, so it cannot drift, consistent with [[adr-191-overview-target-date]].
- `@project_data.decisions` is untouched, so every existing consumer (overview, linking, rendering) is unaffected.

## Negative

- The collection is unused on arrival, which is deliberate but means its shape is validated only by its unit tests until a real consumer lands.

## Neutral

- Records placed directly under `decisions/` form a `.` group rather than being omitted; current projects keep all records in release folders, so this path is exercised only by the tests.

# Alternatives Considered

- **Hand-listing each group's records in code or `project.yml`.** Rejected: it is a second place to maintain membership that would drift as records are added or moved — the exact anti-pattern [[adr-191-overview-target-date]] warns against. The folder already lists the records.
- **Grouping by each record's derived `Target Release Version` instead of the folder.** Rejected for this step: it conflates "planned together" (the folder) with "ships in version X" (the record's own declaration), and offers no home for future group-level metadata. The folder is the author's explicit planning grouping.
- **Storing the group name as an attribute on `Decision` rather than a separate collection.** Rejected as insufficient for the consumers: the buffer and fever-chart steps need to iterate groups and their members, which a flat per-record attribute does not provide directly. The two are not mutually exclusive and a `Decision#group` accessor can be added later if a consumer wants it.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.2 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.3 |

# References

- [goldratt-flow-analysis.md](./../../goldratt-flow-analysis.md) — the roadmap whose later steps consume this grouping
- [[adr-195-critical-chain-buffer]] — the project-buffer step that aggregates over a group of records
- [[adr-196-buffer-fever-chart]] — the per-release fever chart, the other consumer
- [[adr-193-owner-wip-heatmap]] — sibling derive-from-existing-structure step; established the per-row planning vocabulary
- [[adr-194-full-kit-readiness]] — the list-of-single-key-hashes shape reused here for `decision_groups`
- [[adr-191-overview-target-date]] — the derive-don't-duplicate / no-third-place-that-drifts discipline this step follows
- [[adr-170-introduce-decision-records]] — introduced decision records, their folders, and the overview

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()
