---
title: "ISSUE-171: Dash as Marker for Unordered List"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 17-05-2026 | Proposed |
|   |17-05-2026 | Accepted |
| * |17-05-2026 | Implemented |

# Context

Initially the only "*" markers were implemented in the Almirah Ruby gem for bullet lists (unordered lilsts). However Markdown format allows to use both "*" and "-" fur such purpose.

# Decision

It would be good to add the support of "-" along as "*".

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Done | 17-05-2026 | 17-05-2026 | SRS-017 updated to allow "*" or "-" as the item marker |
| Code | Done | 17-05-2026 | 17-05-2026 | Dash support added in doc_parser.rb and markdown_list.rb |
| Tests | Done | 17-05-2026 | 17-05-2026 | Five new unit tests in doc_parser_spec.rb; `<REQ>` traces added for SRS-017, SRS-019, SRS-024 |

# Out of Scope

Not identified

# Consequences

## Positive

- Existing ADR-170 will be parsed correctly.
- In most of the cases I see the "-" instead of "*" in bullet lists, so there will be less rework efforts.

## Negative

Not identified. 

## Neutral

- The unordered-list continuation regex was anchored to start-of-line and tightened to require whitespace after the marker. This is stricter than the previous regex for "*" as well, and closes a latent edge case where mid-paragraph "*" could be read as a list continuation.

# Alternatives Considered

No alternatives were considered

# Proposed Changes

1. Broaden the unordered-list entry-point regex in the document parser to match either `*` or `-` followed by a space.
2. Update the `MarkdownList.unordered_list_item?` class method so list continuation lines starting with `-` are recognised.
3. Extend the marker state in `MarkdownList#calculate_text_position` to treat `-` as a list-item marker alongside `*`.
4. Add unit tests for `-` as a top-level marker, for mixed `*`/`-` nested lists, and for `-` items with bold/italic formatting.
5. Update the SRS-017 example to show that both `*` and `-` are valid unordered-list markers.
6. Verify ADR-170 renders correctly after the change (it already uses `-` for its Positive / Negative / Neutral bullets).

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.3.1 |
| Target Release Version | 0.4.0 |

# References

n/a

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47)