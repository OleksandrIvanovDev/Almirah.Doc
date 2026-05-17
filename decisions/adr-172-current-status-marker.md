---
title: "ADR-172: Current Status Marker in Decision Record Status Table"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-05-2026 | Proposed |
| * | 17-05-2026 | Accepted |

# Context

Decision records track their lifecycle (Proposed → Accepted → In-Progress → Implemented → ...) via the Status table introduced in ADR-170. The current convention is append-only: each status change adds a new row, and the latest row implicitly represents the current state.

This convention works for retrospective tracking, but it does not let an author plan transitions ahead of time. If a record is currently In-Progress and we know we want to be Implemented by 2026-05-24, we cannot write that future row today without making the record look like it is already implemented.

# Decision

Add a leading column to the Status table that holds an asterisk "*" in exactly one row — the row whose status is the present current state of the decision record. All other rows leave that column empty. Future-dated transitions may be written upfront for planning purposes; only the row marked with "*" is considered the current state.

Example with planned transitions ahead of the current state:

|  | Date | Status |
|:---:|---|---|
|   | 14-05-2026 | Proposed |
|   | 14-05-2026 | Accepted |
| * | 15-05-2026 | In-Progress |
|   | 24-05-2026 | Implemented |

The decision record owner authors the marker manually. In addition to the documentation convention, the Almirah Ruby gem shall extend its parsing and rendering of Decision Record documents as follows.

## Current status extraction during parsing

For documents of type Decision Record only, the parser shall locate the Status section, find the row whose leading column contains an asterisk "*", and store the corresponding Status value as an attribute of the Decision instance (e.g., `current_status`). Status sections of other document types (Specification, Protocol, etc.) shall be ignored — extraction is scoped to Decision Records.

When zero or more than one row carries an asterisk, the extracted `current_status` shall be `nil`; this leaves room for a future validation step if duplicate or missing markers become a maintenance problem in practice.

## Triangle substitution during HTML rendering

When rendering a Decision Record's Status table to HTML, the asterisk in the leading marker column shall be replaced with a solid right-pointing triangle "▶" (U+25B6 BLACK RIGHT-POINTING TRIANGLE), matching the visual convention used by database administration tools to highlight the current row. The substitution applies only to the marker column of the Status table in Decision Record documents; other documents and other tables are unchanged.

## Status column on the Decision Records Overview page

A new "Status" column shall be added to the Decision Records Overview page between the existing "Type" and "Title" columns. The cell value for each row shall be the `current_status` attribute of the corresponding Decision instance (empty when `current_status` is `nil`).

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Not-Started |  |  | New SRS items for current-status extraction on the Decision class, asterisk-to-triangle substitution during HTML rendering, and the new Status column on the Decision Records Overview page |
| Code | Not-Started |  |  | Add `current_status` attribute on Decision; parser locates the Status section and reads the "*"-marked row; renderer substitutes "*" with "▶" in the Status table's marker column for Decision Records; DecisionsOverview adds a Status column between Type and Title |
| Tests | Not-Started |  |  | Unit tests for `current_status` extraction (including zero / multiple markers); end-to-end tests for the rendered triangle character in the Status table and for the Status column value in the Decision Records Overview page |

# Out of Scope

- Automatic detection of the current status by comparing today's date against row dates. Authors mark the row manually.
- Validation that exactly one row carries a marker. If zero or more than one is present, `current_status` is `nil` — adding strict validation is a follow-up if it becomes a real maintenance problem.
- Additional visual emphasis on the marked row beyond the triangle substitution (e.g., row background colour, bold text). Can be added later if desired.
- Retroactive update of existing decision records (ADR-170, ISSUE-171) to use the new format. Can be done independently as each record is touched.

# Consequences

## Positive

- Authors can pre-plan transitions by writing future-dated rows without losing clarity about the present state.
- Existing decision records continue to work unchanged; the new column is purely additive and optional per record.
- The convention is plain-text Markdown — survives copy/paste, version control, and external tooling without special handling.

## Negative

- Requires manual maintenance — when status changes, the "*" must be moved to the new current row.
- Risk of multiple "*" or no "*" if maintenance is sloppy. Validation could be added later if it becomes a problem in practice.

## Neutral

- The visible weight of the current status depends on a single character ("*"), which is subtle. A CSS highlight could be added later if more visual emphasis is wanted.

# Alternatives Considered

- **Bold or italic on the current row's status text.** Rejected: harder to author and harder to scan in raw Markdown; the formatting marker travels with the cell content rather than living in a dedicated column.
- **Auto-derive current status from today's date and the row dates.** Rejected: brittle when a record is being viewed retrospectively — the "current" row would shift as the calendar moves. Manual marking gives a stable record of which state was current when the author last touched the file.
- **Use a different marker (e.g., "→", "✓", "now").** The asterisk is unambiguous, easy to type on any keyboard, and visible at small font sizes.
- **Render the marker with a Font Awesome glyph instead of the Unicode triangle.** Rejected: the Unicode "▶" character is plain text — it survives copy/paste from the rendered HTML and keeps the table portable when consumed outside the browser. A Font Awesome `<i>` element would require the Font Awesome stylesheet on the consuming page and would not copy as a glyph.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.0 |

# References

- [ADR-170](./adr-170-introduce-decision-records.md) — introduces decision records and the Status table
- [ISSUE-171](./issues/issue-171-dash-unordered-list.md) — earlier issue surfaced while dogfooding decision records
