---
title: "ADR-218: Risk Affected Documents"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-07-2026 | Proposed |
|   | 07-07-2026 | Analysis |
| * | 07-07-2026 | In-Progress |

# Context

Traceability from a risk to the requirements that control it is what separates a living register from a spreadsheet, and it is the discipline ISO 14971 makes mandatory for medical devices: every risk control measure must trace to its implementing requirement, and the requirement's verification closes the loop. Almirah already owns every link in that chain — requirements trace to tests through coverage and to source through the repositories — except the first one: nothing points from a risk to a requirement.

Decision records solved exactly this with the `# Affected Documents` section ([[adr-174-dr-specification-links]]): a table with the columns numbered row, Proposed Text, and Req-ID, parsed into a ControlledTable whose Req-ID cells carry standard uplinks to controlled paragraphs, producing real links on the record page and downlinks on the specification side.

# Decision

Handle a risk record's `# Affected Documents` section identically to a decision record's, and surface it in the register as bare linked IDs.

- **Same section, same parsing.** A risk record may carry an `# Affected Documents` section holding a table with the columns numbered row, Proposed Text, and Req-ID. The doc parser applies the existing Decision path to RiskRecord documents: the table becomes a ControlledTable, each Req-ID cell's uplink resolves to the controlled paragraph, and the linked specification paragraph shows the risk record among its downlinks.
- **The register cell shows IDs only.** When a registry's configured columns ([ADR-216](./adr-216-risk-register-table.md)) include `Affected Documents`, the cell does not render the section's prose; it renders only the distinct controlled-paragraph IDs the section's Req-ID column links to (for example SRS-123, SDD-015), each a clickable link to that paragraph, in the section's row order. The full table with Proposed Text stays on the record page.
- **Broken links are checked as everywhere.** A Req-ID pointing at a non-existent controlled paragraph is reported by the existing link check; the register cell renders the dangling ID in the existing broken-link style rather than dropping it.
- **The section heading is the established one.** The engine's convention is `Affected Documents`; risk records reuse it verbatim rather than introducing a synonym such as Impacted Documents, so one heading means one behaviour across record types.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA | >[ADR-216] | 1 | 2 | Done | 07-07-2026 |  | State SRS-171: a risk record's Affected Documents section shall be parsed as a controlled table with requirement uplinks as in decision records, and the register table's Affected Documents column shall render only the linked controlled-paragraph IDs as clickable links |
| 2 | Code | DEV | >[ADR-216] | 2 | 3 | Done | 07-07-2026 |  | Extend the doc parser's Affected Documents controlled-table path to RiskRecord documents; collect the section's linked IDs per record; render the register cell as linked IDs with broken links in the existing style |
| 3 | Tests | TEST |  | 1 | 2 | Done | 07-07-2026 |  | Add an `Almirah.Code` e2e asserting the record-page table links, the specification downlinks, the IDs-only register cell, and the broken-link rendering, with inline fixtures including a risk record carrying valid and dangling Req-IDs |

# Out of Scope

- **Linking risks to test protocols or source files directly.** The chain runs risk to requirement; the requirement's existing coverage and implementation links carry it onward.
- **Risk-to-risk links.** Records may reference each other with ordinary cross-document links; no dedicated mechanism is added.
- **A dedicated risk traceability matrix page.** The specification downlinks and the register column are the v1 views; a matrix can follow as its own decision if needed.

# Consequences

## Positive

- The ISO 14971 loop closes inside one build: risk to controlling requirement to verifying test, all as existing link machinery — the differentiator no spreadsheet register has.
- Specifications gain the reverse reading for free: a controlled paragraph shows which risks it exists to control, through the downlinks the linker already computes.
- Reusing the Decision parsing path verbatim means no new table grammar and no new author-facing convention.

## Negative

- The register cell hides the Proposed Text, so a reviewer wanting the control wording must open the record page; the cell trades completeness for a scannable register.

## Neutral

- A risk record without the section is simply a risk without document impact — an empty register cell, valid and common for accepted or process risks.
- The Proposed Text column keeps its decision-record meaning: the requirement text this risk proposes or amends, pending its incorporation into the specification.

# Alternatives Considered

- **Naming the section Impacted Documents.** Rejected: the engine and every existing record say Affected Documents; a second spelling for the same behaviour would fork the convention and double the parser's section matching for no expressive gain.
- **Rendering the full Affected Documents table in the register cell.** Rejected: Proposed Text is a paragraph per row and would dominate the register; the linked IDs are the register-level fact, the record page keeps the detail.
- **Frontmatter list of requirement IDs instead of the table.** Rejected: it would bypass the ControlledTable and uplink machinery, losing the downlinks, the broken-link check, and the Proposed Text that makes the impact reviewable.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.3 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.4 |

# Affected Documents

This decision adds SRS-171.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall parse a risk record's Affected Documents section as a controlled table whose Req-ID column carries uplinks to controlled paragraphs, as for decision records, and shall render the register table's Affected Documents column as only the distinct linked controlled-paragraph IDs, each a clickable link to its paragraph. | >[SRS-171] |

# References

- [[adr-174-dr-specification-links]] — the decision-record Affected Documents mechanism extended here
- [ADR-215](./adr-215-risk-record-collection.md) — the risk record type the parser path is extended to
- [ADR-216](./adr-216-risk-register-table.md) — the register table the IDs-only column renders into

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 05-07-2026 | Requirements | BA | 1 | Initial Proposal |
| 07-07-2026 | Requirements | DEV | 0.25 | Analysis |
