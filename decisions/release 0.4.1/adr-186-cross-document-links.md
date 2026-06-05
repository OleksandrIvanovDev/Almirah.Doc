---
title: "ADR-186: Native Cross-Document Links (Markdown and Wiki)"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 01-06-2026 | Proposed |
|   | 01-06-2026 | Accepted |
|   | 04-06-2026 | In-Progress |
| * | 04-06-2026 | Implemented |

# Context

Almirah has two independent linking mechanisms:

- **Structured ID links** — the traceability backbone: `>[ID]` uplinks (specification→specification, test-step→specification, decision→specification "Affected Documents") and `<REQ>…>[ID]</REQ>` source links. These are resolved by the linker by item ID and rendered as correct `…html#ID` anchors. They work well and are unchanged by this ADR.
- **Free-text Markdown links** `[text](url)` — handled in `TextLine#link`. This is the weak area this ADR addresses.

For free-text links the current behaviour is inconsistent across the document types a project actually contains:

- **Specification → specification**: works only in a narrow case. `[t](sys.md)` is rewritten to the target's HTML page only when the target's filename stem is a registered specification id matched by the word-only regex `(\w+)\.md`. The emitted href uses **backslashes** (`.\..\sys\sys.html`), which are not valid URL separators, and a hardcoded `..\<id>\<id>.html` path that is correct only because every specification page sits at the same depth.
- **Specification ↔ test protocol**: unsupported. Protocols are never registered as link targets, so the link falls through to a raw external `.md` link.
- **Decision record ↔ decision record**: broken. Decision filenames are hyphenated (`adr-177-overview-pie-chart.md`); the word-only regex captures only the trailing `chart`, which resolves to nothing, so the link renders as a raw external `.md` link.
- **Decision record ↔ specification**: broken in practice. `[t](srs.md)` matches the specification id, but the rewrite path is computed for the specification layout, not the decision page's deeper, nested location, producing a wrong relative URL.
- **`[[wiki]]` / Obsidian `[[Page]]`**: not tokenised at all — rendered as literal `[[…]]` text.

The root causes are structural: the link-target registry holds only specification ids keyed by a hyphen-breaking stem; there is no map from a document to its generated output path and no function to compute the relative URL between any two generated pages (per-type output depth is hardcoded via `instance_of?` ladders); the link builder has no knowledge of which page it is currently rendering; and the parser has no `[[…]]` token.

Two project constraints shape the solution:

1. **A decision record's number is unique across the whole project; its folder is only a filesystem grouping convenience.** A link to a decision record therefore resolves by its unique identifier, independent of which folder it lives in. (Generated decision pages are already named by id — `adr-185.html` — so this is the natural mapping.)
2. **Markdown cross-document links must use native relative-path semantics**, so that the same link the author writes is navigable in an editor (VS Code) and by GenAI tooling that reads the Markdown directly. The link's relative path is resolved against the linking file's own directory; Almirah substitutes the corresponding generated page in the HTML output without changing the on-disk navigability of the source.

# Decision

Introduce native cross-document linking with two complementary syntaxes, backed by a single document registry and a single relative-URL mechanism. Unresolved targets are reported as broken references.

## Document registry

Build one registry covering **every** managed document type (specifications, test protocols, decision records, source files, the index, and the decisions overview). Each entry records the document's generated **output path relative to the build root** (e.g. `specifications/srs/srs.html`, `tests/protocols/tp-001/tp-001.html`, `decisions/release 0.4.1/adr-185.html`). The registry is keyed for lookup by:

- the document's **unique id** (e.g. `srs`, `tp-001`, `adr-185`) — case-insensitive; and
- the document's **source file path** (used to resolve native Markdown relative links).

Decision records are keyed by their unique id regardless of folder (constraint 1).

## Relative-URL mechanism (unified)

Add a single helper that, given the generated output path of the **page currently being rendered** and the generated output path of a **target document**, returns the relative URL between them (using `Pathname#relative_path_from`), with forward-slash separators and percent-encoded spaces.

This replaces the hardcoded per-type depth handling: the `instance_of?`-based `../`, `../../`, `../../../` ladders for stylesheet, script, index, and decisions-overview links are reworked to use the same helper, so **all** internal links share one mechanism. To make this possible, the renderer passes the current document's output location into the link builder, which today has no such context.

## Native Markdown cross-document links

A Markdown link `[text](relative/path.md)` whose relative path — resolved against the **linking document's source directory** — points at a managed document's source file is rewritten in the HTML output to a relative URL targeting that document's generated page. Because generated files are id-named, the rewrite maps the **source** file to the **output** path (e.g. source `…/adr-177-overview-pie-chart.md` → output `…/adr-177.html`); it is not a naive `.md`→`.html` extension swap.

The author's source link remains a valid on-disk relative path to the target `.md`, so it stays navigable in an editor and by GenAI tooling (constraint 2).

## Double-bracket (Obsidian / wiki) links

A `[[target]]` link resolves `target` to a managed document by its **unique id (or filename)**, independent of folder, and is rendered as a relative URL to that document's generated page. The parser gains a `[[…]]` token for this form.

## Aliases and anchors

- **Alias**: `[[target|display text]]` renders `display text` as the visible link text.
- **Anchor**: a fragment is supported on both forms — `path.md#fragment` (Markdown) and `[[target#fragment]]` (double-bracket) — and produces a link to that fragment within the target's generated page. Item ids are already HTML anchors, so `#SRS-001` passes straight through.

## Unresolved links

A cross-document link (either syntax) whose target cannot be resolved to a managed document is **reported as a broken reference**, naming the linking document, and is rendered as a visibly broken link — consistent with how dangling `>[ID]` references are surfaced (SRS-023). The build still completes.

## External links

Links with an explicit external scheme (`http:`, `https:`, `mailto:`, …) are left unchanged and continue to render as external links; they are never treated as cross-document targets.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Done | 01-06-2026 | 04-06-2026 | New SRS items (SRS-088 onward) covering: native Markdown relative cross-document links rewritten to the target's generated page; preserved on-disk editor navigability; double-bracket `[[target]]` links resolved by unique id/filename independent of folder; alias `[[target\|text]]`; anchors on both forms; the relative-URL-from-current-page mechanism for all internal links; broken-reference reporting for unresolved targets; external links left unchanged |
| Code | Done | 01-06-2026 | 04-06-2026 | Build a project-wide document registry (id and source-path keys → output path) populated for all doc types; add a relative-URL helper (`Pathname#relative_path_from`, forward slashes, encoded spaces); pass the current document's output location into the text-line link builder; rewrite `TextLine#link` to resolve native Markdown relative links and emit correct relative HTML URLs; add a `[[…]]` token to the parser with alias and anchor support resolving via the registry; report unresolved targets as broken; rework the hardcoded `instance_of?` depth ladders in `base_document` to use the shared relative-URL helper; remove the backslash-separator and word-only-stem defects |
| Tests | Done | 01-06-2026 | 04-06-2026 | End-to-end tests in `spec/e2e/cross_document_links_spec.rb` (12 examples): spec↔spec, spec↔protocol, decision↔decision, decision↔spec Markdown links each resolve to the correct generated page with a forward-slash relative URL; a decision link resolves by id regardless of folder; `[[target]]`, `[[target\|alias]]`, `[[target#anchor]]`, and `path.md#anchor` all resolve; an unresolved target is reported and rendered broken; an external `http(s)`/`mailto` link is left unchanged; source Markdown links remain valid relative paths on disk. Unit tests for the relative-URL helper and the document registry under `spec/` |

# Out of Scope

- Changing or replacing the structured `>[ID]` / `<REQ>…</REQ>` traceability links. Those keep their current ID-based resolution and rendering.
- Resolving Markdown links by note-name-ignoring-path (Obsidian-style) for the `[text](path.md)` form. Per constraint 2, the Markdown form uses native relative-path semantics; name-based resolution is provided only by the `[[…]]` form.
- A configurable alias table or fuzzy/synonym matching of link targets.
- Heading-text slug anchors beyond passing the fragment through; mapping arbitrary heading text to generated anchor ids can be a later enhancement.
- Rewriting links to non-managed files (images, attachments, a `README.md` that the build does not process). These are left unchanged.
- Backlink panels, link graphs, or "unlinked mentions" views.
- Auto-rewriting or migrating existing documents' links; existing content benefits automatically where links already resolve, but no bulk edit is performed.

# Consequences

## Positive

- All four cross-document cases (spec↔spec, spec↔protocol, decision↔decision, decision↔spec) work for in-text links, with correct forward-slash relative URLs regardless of the linking and target pages' depths.
- Authors write ordinary, editor-navigable Markdown links; the same links work in VS Code, in GenAI tooling, and in the generated site.
- `[[…]]` gives a concise, folder-independent way to reference any document by id — ideal for decision records, whose number is the stable handle.
- Folding the hardcoded depth ladders into one relative-URL helper removes a class of latent path bugs (the backslash separators, the word-only stem, the spec-only assumption) and makes future output-layout changes a single-point edit.
- Unresolved links become visible instead of silently rendering as dead `.md` hrefs.

## Negative

- "Unify path handling" has a broad blast radius: the stylesheet/script/index/overview links for every document type move onto the new helper and must be regression-tested across all output depths.
- Passing per-page render context into the previously stateless, class-level link builder is an architectural change that touches the rendering path for all text.
- Two link syntaxes with different resolution rules (Markdown = relative path; `[[…]]` = id/name) is a concept authors must learn; the distinction is deliberate but must be documented.
- Resolving Markdown links against the source directory means moving a source file can break its outgoing links until they are updated — the expected trade-off for native editor navigability.

## Neutral

- Registry construction and relative-URL computation are O(documents) / O(links) and negligible for any realistic project.
- No change to the generated directory layout, the id-based output filenames, or the structured traceability links.
- The `[[…]]` token is additive to the parser; documents that never use `[[…]]` are unaffected.

# Alternatives Considered

- **Resolve Markdown `[text](path.md)` by note name, ignoring the path (Obsidian-style).** Rejected per constraint 2: it would let authors write links that do not navigate in an editor or for GenAI tooling. Name-based resolution is offered only through the explicit `[[…]]` form.
- **Minimal, additive change that leaves the hardcoded depth ladders in place.** Rejected (the user chose to unify): keeping two path mechanisms would leave the existing backslash/stem/spec-layout defects in the per-type code and force every new output location to be hand-coded again.
- **Leave unresolved links untouched (silent), or fail the build.** Rejected: silent leaves dead links in the output with no signal; failing the build is too strict for authoring. Reporting a broken reference while completing the build matches the existing `>[ID]` behaviour (SRS-023).
- **Keep resolving by filename stem with the existing `(\w+)\.md` regex.** Rejected: it structurally cannot match hyphenated decision filenames and silently mis-resolves; a registry keyed by id and source path is required.
- **Support only one syntax.** Rejected: Markdown relative links serve editor/GenAI navigation; `[[…]]` serves concise id-based references. Both are wanted.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.1 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall resolve a Markdown link whose target, after resolving the link's relative path against the linking document's source directory, is a managed document's source file, and shall render it in the HTML output as a relative link to that target document's generated page. | >[SRS-088] |
| 2 | The software shall preserve the on-disk relative validity of a Markdown cross-document link, so that the link in the source Markdown navigates to the target Markdown file while the generated HTML link navigates to the corresponding generated page. | >[SRS-089] |
| 3 | The software shall support a double-bracket cross-document link of the form `[[target]]` that resolves the target to a managed document by its unique document identifier or filename, independent of the document's folder location. | >[SRS-090] |
| 4 | The software shall support an alias in a double-bracket link of the form `[[target\|display text]]`, rendering the display text as the visible link text. | >[SRS-091] |
| 5 | The software shall support an anchor fragment in a cross-document link, written as `target#fragment` in a Markdown link and as `[[target#fragment]]` in a double-bracket link, producing an HTML link to that fragment within the target document's generated page. | >[SRS-092] |
| 6 | The software shall compute the relative URL of every internal link from the location of the generated page that contains the link to the location of the target's generated page, using forward-slash separators. | >[SRS-093] |
| 7 | The software shall report a cross-document link whose target cannot be resolved to a managed document as a broken reference, naming the linking document, and shall render it as a visibly broken link without aborting the build. | >[SRS-094] |
| 8 | The software shall leave links with an external scheme (such as "http", "https", or "mailto") unchanged and shall not treat them as cross-document targets. | >[SRS-095] |

# References

- SRS-011 and SRS-012 in [srs.md](./../../specifications/srs/srs.md) — existing requirements for internal text links and heading links, which these items extend
- SRS-023 in [srs.md](./../../specifications/srs/srs.md) — existing broken-reference reporting for non-existent `>[ID]` targets, whose behaviour the unresolved-link handling mirrors

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
