---
title: "ISSUE-179: Quoted Asterisk Treated as Italic Marker"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-05-2026 | Proposed |
|   | 21-05-2026 | Accepted |
|   | 21-05-2026 | In-Progress |
| * | 21-05-2026 | Implemented |

# Context

While reviewing the rendered HTML of [[issue-171-dash-unordered-list]], it was observed that the first sentence of its Context section displayed an "asterisk" character as missing and the surrounding text in italic. The source Markdown reads:

> Initially the only "*" markers were implemented in the Almirah Ruby gem for bullet lists (unordered lilsts). However Markdown format allows to use both "*" and "-" fur such purpose.

The renderer turned the first `"*"` opening quote-plus-asterisk into a literal `"` followed by an italic opener; everything up to the second `"*"` was then wrapped in `<i>…</i>`, and the second `"*"` was consumed as the italic closer. Visually, the first asterisk disappeared and the body up to the second `"*"` was italicised.

The root cause is in [text_line.rb:73](./../../../Almirah.Code/lib/almirah/doc_items/text_line.rb#L73). `TextLineParser#tokenize` emitted an `ItalicToken` for every `*` character without any awareness of surrounding context. `TextLineBuilder#restore` then greedily paired the two `ItalicToken`s, producing the spurious italic span. The same defect applies to `**` and `***`: any quoted bold or bold-italic marker `"**"` / `"***"` would behave the same way.

Markdown specifications address this with delimiter "flanking" rules: an emphasis run can only open or close emphasis when its immediate neighbours qualify. The Almirah tokenizer had no such check.

# Decision

Add flanking-aware filtering to the emphasis tokenization step. When `tokenize` matches an `ItalicToken`, `BoldToken`, or `BoldAndItalicToken`, compute whether the matched run is left-flanking or right-flanking based on the character immediately before and after the run in the source string. A run that is *neither* left-flanking nor right-flanking is emitted as literal text instead of an emphasis token.

Use a tightened CommonMark-style rule (stricter than vanilla CommonMark for the punctuation-on-both-sides case):

- **Left-flanking**: the character immediately after the run is non-whitespace AND (either non-punctuation OR the character immediately before the run is whitespace or start-of-line).
- **Right-flanking**: symmetric — the character immediately before the run is non-whitespace AND (either non-punctuation OR the character immediately after the run is whitespace or end-of-line).

The "tightened" part is the removal of the "or preceded by punctuation" branch from the CommonMark left-flanking rule (and its symmetric counterpart for right-flanking). Vanilla CommonMark would still render `"*"` as italic; the stricter rule correctly leaves it as a literal asterisk while preserving legitimate cases such as `*"foo"*`, `(*foo*)`, `*foo*!`, etc.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Done | 21-05-2026 | 21-05-2026 | Add `can_flank?` / `left_flanking?` / `right_flanking?` helpers in `text_line.rb` and call them from `tokenize` before emitting emphasis tokens; fall through to literal text when neither flanking direction holds |
| Tests | Done | 21-05-2026 | 21-05-2026 | Five new unit tests in `text_line_spec.rb` covering: single quoted asterisk, two quoted asterisks on the same line, quoted phrase `*"foo"*` still italicises, lone asterisks surrounded by spaces stay literal, quoted double-asterisk `"**"` stays literal |

# Out of Scope

- A full CommonMark emphasis implementation. The Almirah tokenizer remains intentionally simple — backslash escapes (`\*`), the CommonMark "rule of 3" for adjacent runs, and intra-word `_`-emphasis restrictions are not introduced by this fix.
- Heuristics based on surrounding tokens after tokenization. The check is purely on raw source-string neighbours of each run.
- Other punctuation-disambiguation issues (e.g., `[` / `]` inside link text). Only the asterisk-based emphasis markers are affected.

# Consequences

## Positive

- The Context section of [[issue-171-dash-unordered-list]] and any future document containing a quoted asterisk renders as the author intended.
- `* lone * star ` patterns (asterisks surrounded by whitespace) are now also treated as literal, which closes a latent edge case where a paragraph containing two whitespace-adjacent asterisks would silently italicise the text between them.

## Negative

- A pre-existing rendered HTML of a document that *relied* on the loose old behaviour (e.g., intentionally writing `"*"` between two `"`s to produce italic output) would now render differently. No such document exists in `Almirah.Doc`.

## Neutral

- The change is local to `text_line.rb`. The `format_string` still takes a raw string and returns formatted HTML.
- All 21 pre-existing tests in `text_line_spec.rb` continue to pass without modification, confirming the rule does not regress legitimate `*foo*`, `**foo**`, `***foo***`, `*"foo"*`, mixed-format, and edge-case patterns already covered.

# Alternatives Considered

- **Vanilla CommonMark flanking rules.** Rejected: CommonMark treats `"*"` as left-flanking (via the "punctuation preceded by punctuation" branch of rule 2b), so the user-visible bug would remain. The tightened rule diverges from CommonMark only in this specific case and matches author intuition.
- **Backslash escapes (`\*`).** Rejected as the primary fix: it would push the burden onto every author to remember to escape asterisks inside quotes. The tokenizer can resolve the ambiguity itself by looking at neighbours.
- **Tagging tokens with `can_open` / `can_close` flags and resolving in `restore`.** Considered but not needed: dropping non-flanking runs to literal text during `tokenize` is simpler and sufficient for the observed defect, and avoids touching the matching algorithm.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ISSUE-171](./issue-171-dash-unordered-list.md) — the document whose rendered HTML surfaced the defect
- SRS items related to text-line formatting in [srs.md](./../../specifications/srs/srs.md)