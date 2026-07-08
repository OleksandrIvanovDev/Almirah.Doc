---
title: "ADR-219: Risks Menu and Registries Page"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-07-2026 | Proposed |
|   | 07-07-2026 | Analysis |
|   | 07-07-2026 | In-Progress |
| * | 08-07-2026 | Implemented |

# Context

The registry pages ([ADR-216](./adr-216-risk-register-table.md)) each show one risk type in full. What a project review opens with is the level above: how many risks each registry holds, how many are still open, and how severe the worst of them is — the portfolio read ISO 31000 files under monitoring and review. Decision records set the navigation precedent: a collection earns a top-menu entry leading to its overview page.

# Decision

Add a Risks entry to the top menu bar leading to an all-registries summary page.

- **The menu button.** The top menu bar gains a Risks button linking to `build/risks/overview.html`. The button is emitted only when the project has at least one risk registry, so projects without a `risks/` folder see no change.
- **The registries table.** The page holds one table with a row per registry, in file-system order, with the columns Risk Registry, Total Risks, Open Risks, Highest RPN, and Average RPN. The Risk Registry cell is the registry name linked to its registry page.
- **Open Risks** counts the records whose current lifecycle status ([ADR-215](./adr-215-risk-record-collection.md)) is neither Closed nor empty of meaning to the reader: precisely, every record whose marked status is not Closed. The definition is deliberately simple; a registry using another terminal status documents it in its preface and the count stays honest to the marker.
- **RPN aggregates use the registry's leading group.** Highest and Average RPN are computed over the records' first configured RPN group ([ADR-217](./adr-217-rpn-computed-columns.md)) — the initial risk by convention — ignoring records whose group value is blank. Both cells are blank for a registry with no RPN configuration or no computable values. When the leading group carries thresholds, the Highest RPN cell is coloured by them, so the worst risk's verdict carries up to the portfolio view.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA | >[ADR-217] | 1 | 2 | Done | 07-07-2026 | 08-07-2026 | State SRS-172: the software shall add a Risks top-menu entry, present only when a registry exists, leading to a summary page with one row per registry carrying total, open, highest-RPN and average-RPN figures with the registry name linked to its page |
| 2 | Code | DEV | >[ADR-217] | 2 | 3 | Done | 07-07-2026 | 08-07-2026 | Emit the conditional top-menu button; render the registries page with the aggregate table; compute open counts from the lifecycle marker and the RPN aggregates from the leading group, blank-safe; colour Highest RPN by the leading group's thresholds |
| 3 | Tests | TEST |  | 2 | 3 | Done | 07-07-2026 | 08-07-2026 | Add an `Almirah.Code` e2e asserting the button's presence and absence, the row per registry, the open count, the blank aggregates without RPN configuration, and the threshold colouring of Highest RPN, with inline fixtures |

# Out of Scope

- **Charts on the registries page.** A status distribution or RPN heatmap can follow the decisions-overview precedent later; v1 is the table.
- **A Lowest RPN column.** Dropped from the original sketch: reviews are driven by the worst and the open risks, not the safest one.
- **Cross-registry aggregation.** No combined total row; registries use different scales, so summing their RPNs would be meaningless.
- **Above-threshold counts.** A count of risks in the unacceptable band is a natural follow-up once thresholds see real use; the coloured Highest RPN carries the v1 signal.

# Consequences

## Positive

- The portfolio question — how many risks, how many open, how bad is the worst — is answered on one page, one click from anywhere in the build.
- The conditional button keeps projects without risks entirely unchanged.
- Colouring Highest RPN by the leading group's thresholds propagates the acceptability verdict to the summary without recomputing anything.

## Negative

- Average RPN over ordinal-scale products is statistically shaky; it is kept because register practice expects it, but Highest RPN and Open Risks are the columns that should drive action.

## Neutral

- File-system row order matches how every other Almirah collection lists; reordering registries means renaming folders.
- A registry whose leading group is Residual rather than Initial gets residual aggregates — the configuration order expresses the intent.

# Alternatives Considered

- **The originally sketched columns Lowest, Average, Highest RPN.** Reshaped: Lowest RPN answers no management question and its slot is better spent on Open Risks, the count every review actually asks for first.
- **Always emitting the Risks button.** Rejected: a dead menu entry on projects without registries is noise; the decisions button precedent is a collection-driven menu.
- **Aggregating over all RPN groups per registry.** Rejected: mixing initial and residual values in one maximum or average conflates before and after mitigation; the leading group is a deliberate, configurable choice.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This decision adds SRS-172.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall add a Risks entry to the top menu bar, present only when the project has at least one risk registry, leading to a summary page holding one row per registry with the columns Risk Registry, Total Risks, Open Risks, Highest RPN and Average RPN, where the registry name links to its registry page, the open count excludes records whose current status is Closed, and the RPN aggregates are computed over the registry's leading RPN group ignoring blank values. | >[SRS-172] |

# References

- [ADR-215](./adr-215-risk-record-collection.md) — the lifecycle marker the open count reads
- [ADR-216](./adr-216-risk-register-table.md) — the registry pages this page links to
- [ADR-217](./adr-217-rpn-computed-columns.md) — the RPN groups and thresholds the aggregates reuse
- [[adr-170-introduce-decision-records]] — the collection-earns-a-menu-entry precedent

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/32)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/51)

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 07-07-2026 | Requirements | DEV | 0.25 | Analysis |
