---
title: "ISSUE-192: Double-Bracket Links Fail to Resolve by Filename"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 11-06-2026 | Proposed |
|   | 11-06-2026 | Accepted |
|   | 11-06-2026 | In-Progress |
| * | 11-06-2026 | Implemented |

# Context

Double-bracket cross-document links of the form `[[target]]` (introduced by [[adr-186-cross-document-links]], specified by SRS-090) are rendered as broken whenever the target is a decision record written by its full filename, e.g. `[[adr-170-introduce-decision-records]]`. Building `Almirah.Doc` reports 38 broken links, almost all of this form, even though every target document exists.

SRS-090 requires resolving the target "by its unique document identifier **or filename**, independent of the document's folder location." Only the identifier half is implemented — there is no resolution by filename.

The chain of cause:

- `[[…]]` resolution calls `find_by_id(target)` in [text_line.rb:486](./../../../Almirah.Code/lib/almirah/doc_items/text_line.rb#L486), which looks up `LinkRegistry`'s `@by_id` map ([link_registry.rb:31-33](./../../../Almirah.Code/lib/almirah/link_registry.rb#L31-L33)).
- `LinkRegistry#register` ([link_registry.rb:19-29](./../../../Almirah.Code/lib/almirah/link_registry.rb#L19-L29)) keys `@by_id` solely on `doc.id`. There is no filename-stem key.
- A `Decision`'s `id` is **not** its filename stem: `assign_id_parts` ([decision.rb:155-166](./../../../Almirah.Code/lib/almirah/doc_types/decision.rb#L155-L166)) parses the stem with `/\A([A-Za-z]+)-(\d+)/` and keeps only the `<letters>-<digits>` prefix, so `adr-170-introduce-decision-records.md` registers as `adr-170` and the descriptive slug is dropped.

As a result the registry holds `adr-170`, while authors link to `adr-170-introduce-decision-records` (the natural Obsidian-style handle matching the file on disk), and the two never match. By contrast `Specification` and `Protocol` set `id` to the whole filename stem ([specification.rb:27](./../../../Almirah.Code/lib/almirah/doc_types/specification.rb#L27), [protocol.rb:9](./../../../Almirah.Code/lib/almirah/doc_types/protocol.rb#L9)), so their `[[…]]` links already work — the gap is specific to decision records, which is exactly the document type the `[[…]]` links point at.

# Decision

Index every managed document in the registry by its **filename stem** in addition to its `id`, fulfilling the "or filename" clause of SRS-090.

In `LinkRegistry#register`, after registering by `doc.id`, also register the document under its lowercased filename stem (`File.basename(doc.path, '.*').downcase`) when that stem differs from `doc.id` and `doc.path` is present. Both keys point at the **same document object**, so a decision record resolves from either form:

- `[[adr-170]]` → existing `@by_id["adr-170"]` entry (unchanged)
- `[[adr-170-introduce-decision-records]]` → new `@by_id["adr-170-introduce-decision-records"]` alias

The change is purely additive — one extra dictionary entry per decision record. For specifications and protocols the stem already equals the `id`, so the second insert is skipped and their behaviour is unchanged. The existing collision guard ([link_registry.rb:21-24](./../../../Almirah.Code/lib/almirah/link_registry.rb#L21-L24)) only fires when a key maps to a *different* document, and both the short id and the full stem are unique across decision records, so no false collisions are introduced.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Implemented | 11-06-2026 | 11-06-2026 | `LinkRegistry#register` ([lib/almirah/link_registry.rb](./../../../Almirah.Code/lib/almirah/link_registry.rb)) now indexes each document by its lowercased filename stem (`File.basename(doc.path, '.*')`) in addition to its `id`, via a shared `register_id` helper that skips empty keys and flags a collision only on a genuinely different document. Decision records resolve by full filename as well as short id |
| Requirements | Implemented | 11-06-2026 | 11-06-2026 | No new requirement — this is a defect against existing SRS-090, whose "or filename" clause was unimplemented. SRS-090 text is unchanged |
| Tests | Implemented | 11-06-2026 | 11-06-2026 | Added a unit example in `spec/link_registry_spec.rb` (resolve by full stem and by id, no collision) and an end-to-end example in `spec/e2e/cross_document_links_spec.rb` (`[[adr-701-second]]` resolves to the id-named page, not broken). Full suite green (260 examples). Re-running `almirah please .` in `Almirah.Doc` dropped the broken-link count from 38 to 9, with every `[[…]]` entry gone (the 9 remaining are plain Markdown relative-path links, out of scope here) |

# Out of Scope

- **The plain Markdown relative links** (e.g. `./../specifications/srs/srs.md`) that also appear in the broken-links report. Those resolve by on-disk relative path, not by id/filename, and are a separate defect.
- **Changing how a `Decision`'s `id` is derived.** The short `<letters>-<digits>` id remains the stable handle used for display, ordering, and the overview page; the filename stem is added only as an additional resolution alias.
- **Resolving the Markdown `[text](path.md)` form by note name.** Per [[adr-186-cross-document-links]] constraint 2, name-based resolution is offered only through the `[[…]]` form.

# Consequences

## Positive

- All 38 reported `[[…]]` links in `Almirah.Doc` resolve, with no edits to any document.
- Authors may reference a decision record by either its full filename (the natural editor/Obsidian handle) or its short id — both work and point to the same page.
- SRS-090's "or filename" promise becomes true for every document type, not just specifications and protocols.

## Negative

- A document now occupies up to two entries in the id map. This is negligible in memory and cannot cause a wrong resolution, since both keys map to the same object.

## Neutral

- Already-rendered projects must be re-rendered to pick up corrected links; the fix is in the gem, not applied retroactively.
- Short-id resolution (`[[adr-170]]`) is unchanged.

# Alternatives Considered

- **Rewrite every `[[…]]` link in `Almirah.Doc` to the short id (`[[adr-170]]`).** Rejected: works today but contradicts SRS-090 (filename should resolve), is a manual sweep across many records, and leaves the underlying defect in the framework so the next author hits it again.
- **Make `Decision#id` the full filename stem.** Rejected: the short `<letters>-<digits>` id is the stable handle relied on for display, sorting, and the overview; widening it would ripple through unrelated code and break short-id links. Adding an alias is narrower and safer.
- **Resolve `[[…]]` by scanning documents for a filename match at link time.** Rejected: duplicates the registry's purpose and is O(n) per link; a second registry key is O(1) and consistent with the existing design.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.1 |
| Issue Found in Version | 0.4.1 |
| Target Release Version | 0.4.2 |

# References

- [[adr-186-cross-document-links]] — introduced the `[[…]]` form and SRS-090/091/092.

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/30)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/49)
