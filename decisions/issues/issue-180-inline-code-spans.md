---
title: "ISSUE-180: Inline Code Spans Not Recognised"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 21-05-2026 | Proposed |
|   | 21-05-2026 | Accepted |
|   | 21-05-2026 | In-Progress |
| * | 22-05-2026 | Implemented |

# Context

While reviewing the rendered HTML of [[issue-179-emphasis-flanking-rule]], it was observed that text wrapped in single backticks was not rendered as inline code. In the source, the paragraph contained `<i>…</i>` wrapped in single backticks — intended to display the literal HTML tags. The rendered page instead showed the body italicised and the `<i>` tags missing, because the backticks were emitted as literal characters and the browser interpreted the `<i>…</i>` content between them as actual italic markup.

Two facts about the parser explain this:

1. [doc_parser.rb:292](./../../../Almirah.Code/lib/almirah/doc_parser.rb#L292) handles **fenced** code blocks (lines starting with three consecutive backticks) at the line level, but there is no corresponding handling for **inline** (single-backtick) code spans.
2. [text_line.rb:57](./../../../Almirah.Code/lib/almirah/doc_items/text_line.rb#L57) (`TextLineParser`) registers tokens for `*`, `**`, `***`, `(`, `)`, `[`, `]`, `](` — but not for the backtick. Backticks therefore fell into the literal-character branch of the tokenizer and were emitted verbatim into the HTML output, along with anything between them.

This is a real authoring trap: any decision record, requirement, or test step that documents a piece of HTML, an XML tag, or an entity reference would have the inner content silently re-interpreted by the browser.

# Decision

Add inline code-span support to the text-line tokenizer.

## Tokenisation

Introduce a `BacktickToken` (whose `value` is the backtick character itself) registered in `TextLineParser#supported_tokens`. After the main token-emission loop, run a second pass (`fuse_backticks`) that walks the token list, pairs consecutive `BacktickToken` instances, and replaces each opener/content/closer triple with a single `InlineCodeToken` whose value is the raw content between the backticks. An unmatched `BacktickToken` (no closer found) is converted to a literal backtick character.

This second pass runs before any emphasis matching in `restore`, so a code span fully shadows whatever other markers it contains: when the source places `*foo*` inside a single-backtick code span, the rendered HTML preserves the literal asterisks rather than wrapping `foo` in `<i>` tags. The raw content is reconstructed by concatenating the `value` of each token between the two backticks, which is straightforward because every token in the token stream carries its source-level text in `value`.

## Rendering

Extend `TextLineBuilderContext` with a default `inline_code(str)` method that returns the raw string. The active `TextLine` subclass overrides it to:

- HTML-escape the raw content using `CGI.escapeHTML` (covering `&`, `<`, `>`, `"`, `'`), and
- wrap the escaped content in `<code>…</code>` tags.

Escaping is performed at the rendering layer, not the parser, to keep concerns separated. The tokenizer stays HTML-agnostic.

## Coverage of existing cases

`<i>…</i>` between backticks renders as `<code>&lt;i&gt;…&lt;/i&gt;</code>`. `A & B` renders as `<code>A &amp; B</code>`. Asterisks, brackets, and other emphasis markers inside backticks remain literal.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Done | 21-05-2026 | 22-05-2026 | Add `BacktickToken` and `InlineCodeToken` classes in `text_line.rb`; register the backtick token; add `fuse_backticks` and `next_backtick_index` helpers called at the end of `tokenize`; add `InlineCodeToken` case in `TextLineBuilder#restore`; add `inline_code` to `TextLineBuilderContext` and `TextLine` (the latter HTML-escapes via `CGI.escapeHTML` and wraps in `<code>` tags); `require 'cgi'` at the top of the file |
| Tests | Done | 21-05-2026 | 22-05-2026 | Seven new unit tests in `text_line_spec.rb` covering: a basic `foo()` code span, HTML escaping of `<` and `>` inside backticks, ampersand escaping, emphasis markers inside a code span staying literal, two code spans on the same line, an unmatched single backtick rendered as a literal character, and code-span followed by an italic span |

# Out of Scope

- **Fenced code blocks (triple backticks)** are unchanged. Those are line-level constructs and already work via `doc_parser.rb`.
- **Multi-backtick code spans** (the CommonMark syntax where a pair of double-backtick delimiters surrounds a code-span body that may itself contain a single backtick). Only single-backtick delimiter pairs are supported.
- **Language hints in inline code** (e.g., a `js:` prefix inside a code span to mark the contained text as JavaScript). Inline code is unstyled beyond the `<code>` tag.
- **CSS styling of the `<code>` tag.** The HTML now carries `<code>` elements; whether they receive distinctive monospace/background styling is left to `main.css` and can be addressed independently.
- **HTML escaping of plain (non-code) text.** Authors who write raw `<` or `>` outside backticks still produce browser-interpreted markup. This ADR scope is limited to closing the inline-code authoring trap.
- **Backslash-escape of backticks** (using a leading backslash to render a literal backtick character). Authors who need a literal backtick outside a code span can write it as-is when it is unmatched; for matched-pair situations escaping is out of scope.

# Consequences

## Positive

- The HTML rendering of [[issue-179-emphasis-flanking-rule]] now matches the source intent: `<code>&lt;i&gt;…&lt;/i&gt;</code>` is displayed verbatim.
- Documentation, decision records, and test steps can safely refer to HTML tags, XML elements, and code snippets without manual escaping.
- The change removes a footgun where inline content between backticks was silently re-interpreted by the browser.

## Negative

- Pre-existing rendered HTML that *unintentionally* relied on the loose behaviour (e.g., a document where a stray backtick happened to surround already-correct HTML) would now render as a code span. No such document exists in `Almirah.Doc`.
- An author who writes a single unpaired backtick in a paragraph (e.g., as part of prose) will see it rendered as a literal backtick character, which is the same as the prior behaviour and is the desired fallback.

## Neutral

- The change is local to `text_line.rb`. No public API change; `format_string` still takes a raw string and returns formatted HTML.
- The default `TextLineBuilderContext#inline_code` returns the raw string, so any non-HTML consumer of the parser (e.g., future plain-text or PDF renderers) sees inline-code content as plain text without modification.
- All 26 pre-existing tests in `text_line_spec.rb` continue to pass without modification.

# Alternatives Considered

- **Handle backticks inside `restore` instead of as a preprocessing pass.** Rejected: `restore` is a sequential walk; if an italic opener appears before a backtick opener, its closer search would compete with the code-span boundary. Fusing in a preprocessing pass cleanly establishes "code spans win over emphasis" without complicating `restore`.
- **Store the rendered `<code>…</code>` string directly in `InlineCodeToken#value`.** Rejected: that pushes HTML knowledge into the parser. Keeping escaping in `TextLine#inline_code` lets the base `TextLineBuilderContext` remain HTML-agnostic for future non-HTML consumers.
- **Manually substitute `&`, `<`, `>` instead of using `CGI.escapeHTML`.** Rejected: `CGI` is stdlib, the canonical escape covers `"` and `'` in addition to the three reserved characters, and using the standard helper avoids accumulating ad-hoc escape logic.
- **Require authors to backslash-escape `<` and `>` outside backticks too.** Rejected: out of scope for this ADR. The narrower fix closes the documented authoring trap without changing how plain text is treated.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.0 |

# References

- [ISSUE-179](./issue-179-emphasis-flanking-rule.md) — the document whose rendered HTML surfaced the defect
- [ADR-170](./../adr-170-introduce-decision-records.md) — the decision-record convention that this ISSUE follows

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/28)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47) 
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/47)