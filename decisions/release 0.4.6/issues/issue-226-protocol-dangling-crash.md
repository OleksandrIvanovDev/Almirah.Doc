---
title: "ISSUE-226: Protocol Dangling Req-ID Crashes the Build"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

A test protocol whose Req-ID column references a controlled paragraph that does not exist in an existing specification crashes the entire build with a NoMethodError, rendering nothing. A two-file project reproduces it: a specification holding AAA-001 and a protocol step linking >[AAA-999] abort in the linker before any HTML is written.

The root cause is an inconsistency among the document types. When linking finds a dangling Req-ID, it records the reference in the owning document's wrong-links collection — the data behind the Index page's "Wrong links" column. Specification, Decision (risk records included) and SourceFile each define that collection; Protocol never did, although the protocol-to-specification linker writes to it on exactly this path (doc_linker.rb, link_protocol_to_spec). Every other document type survives a dangling reference and reports it; a protocol kills the build.

The defect predates the current release and was found on 09-07-2026 while building the e2e coverage for [[enh-225-affected-documents-highlight]], whose renderer consults the same collection.

# Decision

Protocol gains the same wrong-links collection as the other linked document types: the accessor and an empty hash at construction, in lib/almirah/doc_types/protocol.rb. The linker then records a protocol's dangling Req-ID exactly as it does a specification's or a decision's, and the build completes.

One visible consequence follows deliberately: because the controlled-table reference renderer of [[enh-225-affected-documents-highlight]] highlights any cell whose owning document records the uplink as dangling, a protocol's Req-ID cell with a dangling reference now renders with the red background and broken-link style instead of crashing the build — the same meaning in the same cell type as everywhere else.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Analysis | DEV |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | Root cause: Protocol lacks the wrong-links collection the protocol-to-specification linker writes to on the dangling-reference path |
| 2 | Code | DEV |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | Define the wrong-links collection on Protocol as on the other document types |
| 3 | Tests | TEST |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | e2e regression: a project with a protocol step linking a missing paragraph of an existing specification builds successfully and renders the protocol page |

# Out of Scope

- **Reporting protocol wrong links on the Index page.** The Index "Wrong links" column keeps its current sources; only the crash and the cell rendering change.
- **References to documents that do not exist at all.** As for the other document types, only references whose specification exists are recorded; a missing document is a separate, pre-existing gap.

# Consequences

## Positive

- A dangling test-step reference — a normal intermediate state while requirements move — no longer takes down the whole build; it is recorded and, via [[enh-225-affected-documents-highlight]], visibly flagged on the protocol page.

## Negative

- None identified; the crash had no workaround short of removing the reference.

## Neutral

- Protocols now behave like every other linked document type on this path, removing a special case rather than adding one.

# Alternatives Considered

- **Guarding the linker against documents without the collection.** Rejected: it would silently drop the dangling reference for protocols only, entrenching the inconsistency the crash exposed.
- **Hoisting the collection into the shared document base class.** Rejected for scope: three document types declare it locally today; unifying them is a refactoring decision, and the minimal fix mirrors the existing pattern.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.5 |
| Issue Found in Version | 0.4.5 |
| Target Release Version | 0.4.6 |

# Affected Documents

This issue adds no requirement: not crashing on valid input is an implicit property, and the newly visible rendering of the recorded reference is stated by SRS-175 ([[enh-225-affected-documents-highlight]]).

# References

- [[enh-225-affected-documents-highlight]] — the enhancement whose test coverage exposed the crash and whose renderer displays the now-collected references

# Review Evidences

- [Decision Record]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 09-07-2026 | Analysis | DEV | 0.5 | Reproduced with a two-file project; root cause in link_protocol_to_spec |
| 09-07-2026 | Code | DEV | 0.25 | wrong_links_hash defined on Protocol |
| 09-07-2026 | Tests | TEST | 0.25 | Regression covered in the dangling-uplink e2e spec |
