---
title: "ADR-217: RPN Computed Columns"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | 05-07-2026 | Proposed |

# Context

The register table ([ADR-216](./adr-216-risk-register-table.md)) renders authored sections. What it cannot yet do is rank the risks: every register-style technique prioritises through a score computed from the authored factors — Probability times Impact in a project register, Severity times Occurrence times Detection in an FMEA (the classic Risk Priority Number), Probability times monetary Impact for expected monetary value. ISO 14971 and PMBOK additionally demand the score twice: for the initial risk and for the residual risk after mitigation, so the reduction is visible.

A raw number also does not answer the register's core question — is this risk acceptable? Risk matrices express that as zones (acceptable, ALARP, unacceptable); the cheapest faithful rendering of zones over a product score is a pair of thresholds colouring the cell.

The score belongs to the table, not the record: a risk record carries the factors, never an RPN section, so the number can never disagree with its inputs.

# Decision

Add named, computed RPN columns to the register table, configured per registry.

- **Named RPN groups.** Each `risks:` registry entry may carry an `rpn:` list. Each group has a `name`, an ordered `inputs` list of column names, and optional `thresholds`:

```yaml
risks:
  - folder: product
    columns: [Severity, Occurrence, Detection, Mitigation, Status]
    rpn:
      - name: Initial
        inputs: [Severity, Occurrence, Detection]
        thresholds:
          acceptable: 20
          unacceptable: 100
      - name: Residual
        inputs: [Residual Severity, Residual Occurrence, Residual Detection]
        thresholds:
          acceptable: 20
          unacceptable: 100
```

- **The computed columns.** For each group, in configured order, the register table gains a column headed `<Name> RPN` after the configured columns. A record's value is the product of its inputs, each input being the numeric value of the record section named by that input. Inputs may name sections that are not surfaced as columns (as `Residual Severity` above); the factor set and the visible schema are independent.
- **Blank, never zero.** When any input section is missing or does not parse as a number, the group's cell renders blank. A broken or unfilled record must not compute as the safest risk in the registry; the blank cell is the visible data-quality signal.
- **Two or more groups express initial and residual risk.** Nothing limits a registry to two groups, and a single-input group is valid — `inputs: [CVSS Score]` surfaces a precomputed score as a rankable RPN column unchanged.
- **Threshold colouring.** When `thresholds` is present: a value at or below `acceptable` renders in the acceptable style (green), at or above `unacceptable` in the unacceptable style (red), and between them in the caution style (amber) — the ALARP band. Either bound may be given alone; with no `thresholds` the cell is uncoloured. The aggregate pages (ADR-219) reuse the same group definitions.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA | >[ADR-216] | 1 | 2 | To-Do |  |  | State SRS-169 and SRS-170: the register table shall append one computed column per configured named RPN group as the product of the record's numeric input sections, blank when any input is missing or non-numeric, and shall colour the cell by the group's acceptable and unacceptable thresholds |
| 2 | Code | DEV | >[ADR-216] | 2 | 4 | To-Do |  |  | Parse the rpn groups and thresholds from project.yml; compute the product per record from the input sections' numeric values; render the appended columns with blank-on-missing semantics and the three threshold styles |
| 3 | Tests | TEST |  | 2 | 4 | To-Do |  |  | Add an `Almirah.Code` e2e covering a three-factor FMEA group, an initial-plus-residual pair, a single-input group, the blank cell for a non-numeric factor, and the three colour bands; extend the `Almirah.TDS` fixtures |

# Out of Scope

- **Qualitative scale maps.** Translating Low, Medium, High cell text into numbers via a per-registry map is deferred to a follow-up decision; until then RPN inputs must be numeric.
- **Lookup-table acceptability.** Full risk-matrix zone lookup and the AIAG-VDA Action Priority table are mappings, not products; the threshold pair is the deliberate approximation.
- **Weighted sums and other formulas.** The product is the only computation; a registry needing another formula precomputes it into a section and surfaces it through a single-input group.
- **Aggregate statistics across a registry.** Totals, open counts, and highest and average RPN belong to the all-registries page (ADR-219).

# Consequences

## Positive

- One mechanism covers the mainstream scoring schemes: two-factor project registers, three-factor FMEA, expected monetary value, precomputed scores, and the initial-versus-residual pairing ISO 14971 and PMBOK require.
- Thresholds turn the bare number into a verdict at a glance, giving the register the acceptable, ALARP, and unacceptable reading of a risk matrix without a matrix.
- Blank-on-missing makes incomplete records conspicuous instead of quietly ranking them last.

## Negative

- Multiplying ordinal scale values is a known methodological criticism of RPN itself; the engine inherits it by matching the industry's practice.
- Registries using qualitative levels in cells cannot compute RPN until the scale-map follow-up lands; they run with authored columns only.

## Neutral

- The RPN columns exist only in tables; a risk record never carries an RPN section, so score and factors cannot diverge.
- Threshold styles state a policy the registry preface should explain; the engine renders, it does not judge the chosen bounds.

# Alternatives Considered

- **A single hard-coded two-factor RPN.** Rejected: it fits project registers only; FMEA needs three factors and security registries often need one precomputed score, which the free-length inputs list covers uniformly.
- **Positional group syntax without names.** Rejected: a list of bare input arrays would head the columns RPN 1 and RPN 2; named groups head them Initial RPN and Residual RPN, which is what a reviewer needs the table to say.
- **Zero as the missing-input value.** Rejected: zero sorts an unfilled record as the registry's safest risk — the most dangerous possible failure mode for a risk register; blank is the honest rendering.
- **A full colour matrix per probability-severity pair.** Rejected for now: it requires a per-registry matrix definition for marginal gain over thresholds on the product; revisit if a regulated user needs zone-exact ISO 14971 acceptability.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This decision adds SRS-169 and SRS-170.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall append to a risk register table one computed column per RPN group configured for the registry, each group having a name and an ordered list of input section names, the cell value being the product of the record's numeric input section values and blank when any input is missing or not numeric. | >[SRS-169] |
| 2 | The software shall colour an RPN cell by the group's optional thresholds: acceptable style at or below the acceptable bound, unacceptable style at or above the unacceptable bound, caution style between the bounds, and no colouring when thresholds are not configured. | >[SRS-170] |

# References

- [ADR-216](./adr-216-risk-register-table.md) — the register table these columns extend
- [ADR-219](./adr-219-risks-menu-page.md) — the aggregate page reusing these group definitions
- [[adr-196-buffer-fever-chart]] — precedent for computing a derived management signal from authored record data

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
