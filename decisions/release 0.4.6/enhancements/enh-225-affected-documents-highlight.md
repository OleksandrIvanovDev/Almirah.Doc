---
title: "ENH-225: Affected Documents Dangling-Reference Highlight"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

A decision record's Affected Documents table states the controlled paragraphs the decision adds or amends, each as a Req-ID uplink. The intended workflow writes the decision first: at that moment the referenced paragraph — SRS-175, say — does not exist yet, and it appears in the specification only when the Requirements scope item is done. The rendered record gives no visual cue about which of these references have been satisfied: the Req-ID cell renders the same plain clickable link whether the paragraph exists or not, so a reviewer cannot tell a stated-but-pending requirement from a delivered one without following every link.

The framework already detects the condition and already has two presentation precedents:

- During linking, a Req-ID whose specification exists but whose paragraph does not is recorded in the owning record's wrong-links collection — the same data that feeds the Index page's "Wrong links" column, which highlights its count cell with the red background when the count is non-zero.
- The risk register table ([[adr-218-risk-affected-documents]]) already renders a dangling Affected Documents ID in the broken-link style instead of a live link.

Only the record's own page ignores the information it already has.

# Decision

The Req-ID cell of a rendered controlled table is highlighted with the same red background the Index page uses for the "Wrong links" column (background colour #fcc) when at least one of the cell's uplinks is recorded as dangling — referencing a controlled paragraph that does not exist as of the build.

- **The link stays clickable.** A dangling reference keeps its link to the target specification document — that is the document the pending work will modify — but gains the existing broken-link style (red wavy underline) and a tooltip naming the condition, matching the register-table precedent. In a cell carrying several uplinks, only the dangling ones take the broken-link style; the cell background flags the presence of any.
- **Driven by the existing wrong-links data.** The highlight consults the owning document's wrong-links collection at render time; no new detection, linking pass, or console reporting is added. The cell renders exactly as today when the collection has no entry for the uplink.
- **Where it applies.** Any document type that records dangling table uplinks: decision records and risk records (their Affected Documents tables), specifications with controlled tables, and — once [[issue-226-protocol-dangling-crash]] gives protocols the wrong-links collection the linker already expected — test protocols' Req-ID cells. The renderer guards for the collection's presence rather than assuming it.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | State SRS-175: the Req-ID cell of a controlled table is highlighted red and the dangling references take the broken-link style when they point to controlled paragraphs that do not exist, the links staying clickable |
| 2 | Code | DEV |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | In the controlled-table reference column renderer, consult the owning document's wrong-links collection (when the document carries one) and add the red cell background, the broken-link class and the tooltip to dangling references in both the single-uplink and the multi-uplink cell forms |
| 3 | Tests | TEST |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | Extend the `Almirah.Code` e2e specs: a decision whose Affected Documents row references a missing paragraph renders the red cell and the broken-link styled clickable link; a resolved reference renders exactly as today; a multi-uplink cell flags only its dangling references; a protocol page with a dangling Req-ID builds and carries the same highlight |

# Out of Scope

- **References to documents that do not exist at all.** The wrong-links collection is filled only while linking against existing specifications; a Req-ID whose document is missing entirely is not recorded there and renders as today. Extending detection to that case is a separate decision.
- **New detection or reporting.** The console output, the Index page "Wrong links" column, and the linking passes are unchanged; the enhancement is presentation of already-collected data.
- **The risk register table.** Its dangling-reference rendering ([[adr-218-risk-affected-documents]]) already exists and is untouched.
- **The protocol crash on dangling references.** Building the test coverage for this enhancement exposed that a protocol with a dangling Req-ID crashes the build outright; that defect and its fix are recorded separately as [[issue-226-protocol-dangling-crash]].

# Consequences

## Positive

- A freshly written decision record shows at a glance which stated requirements are still pending — the exact state between "decision written" and "specification modified" the workflow produces — and the cue disappears on rebuild once the paragraph is added.
- The colour and the broken-link style are both established meanings in the rendered output, so no new visual vocabulary is introduced.

## Negative

- A record page's Affected Documents table no longer renders identically to the pre-enhancement output while references are pending; snapshots or scraped HTML from older builds differ in those cells.

## Neutral

- Specifications with controlled tables gain the same highlight for their dangling table uplinks — the same meaning in the same cell type, beyond the Affected Documents motivation.

# Alternatives Considered

- **Dropping the link and rendering only a broken-link span,** as the risk register table does. Rejected: on the record's own page the link to the still-to-be-modified document is the useful navigation — the register table links to record pages, a different target with a different purpose.
- **Highlighting only the record types and not specifications.** Rejected: the renderer is shared, the data is per-document, and the meaning — a table uplink to a non-existing paragraph — is identical; special-casing would add a type check to hide consistent information.
- **A new linking pass covering references to non-existing documents.** Rejected as scope: it changes detection semantics (and the console report) rather than presentation; this enhancement renders what is already known.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.5 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.6 |

# Affected Documents

This enhancement adds SRS-175.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall highlight the Req-ID cell of a rendered controlled table with the wrong-links red background when any of the cell's uplinks references a controlled paragraph that does not exist in an existing specification, rendering each such dangling reference in the broken-link style with an explanatory tooltip while keeping its link to the target document clickable, and shall render the cell as before when all its uplinks resolve. | >[SRS-175] |

# References

- [[adr-218-risk-affected-documents]] — the Affected Documents uplink semantics and the register table's broken-link precedent
- [[adr-224-font-size-setting]] — the preceding release 0.4.6 decision, whose pending SRS-174 reference exemplified the missing cue
- [[issue-226-protocol-dangling-crash]] — the pre-existing build crash exposed by this enhancement's test coverage, whose fix extends the highlight to protocol pages

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 09-07-2026 | Requirements | BA | 0.5 | Initial Proposal |
| 09-07-2026 | Requirements | BA | 0.25 | SRS-175 stated under Up-links |
| 09-07-2026 | Code | DEV | 0.5 | Reference-column renderer: red cell, broken-link class, tooltip |
| 09-07-2026 | Tests | TEST | 0.5 | Dangling-uplink e2e spec added, suite 342 green |
