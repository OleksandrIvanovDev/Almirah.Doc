---
title: "ADR-188: Escape and Sanitise Generated HTML Output"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Proposed |
|   | 07-06-2026 | Accepted |
| * | 07-06-2026 | In-Progress |
|   |  | Implemented |

# Context

Almirah's purpose is to render author-written Markdown (specifications, test protocols, decision records, test data) into a published, interlinked HTML site.

A review of the HTML rendering path in `Almirah.Code` found that authored Markdown content is interpolated into the generated HTML essentially **raw**. HTML-escaping is applied in only three narrow places — inline code spans and the visible text of wiki links — confirmed by the fact that the only `CGI.escapeHTML` calls in the whole library are in `lib/almirah/doc_items/text_line.rb` (inline code, and the two `wiki_link` branches). There is no global output-encoding pass and no HTML sanitiser dependency.

As a result, any browser-executable code placed in a source `.md` file is reproduced verbatim in the generated page and executes when a reader opens it. This is **stored / persistent cross-site scripting (XSS)**: the payload lives in committed Markdown, is baked into the static site at build time (`almirah please .`), and fires for every visitor.

Confirmed unescaped rendering paths (verified by running the formatter on crafted inputs):

| Vector | Location | Escaped today |
|---|---|---|
| Paragraph / blockquote / table-cell text | plain-text token emit in `TextLine`; `Paragraph#to_html` | No |
| Heading text | `Heading#to_html` (emitted raw; does not even pass through the text formatter) | No |
| Heading anchor (`name`/`href`) | `Heading#get_anchor_text` slug, derived from the free-form heading text and emitted into `name`/`href` attributes (heading self-link and table of contents) | No |
| Fenced code block content | `CodeBlock#to_html` | No |
| Markdown link URL (href) | `TextLine#link` (no scheme check) | No |
| Markdown link text | `TextLine#link` | No |
| Image `src` and `alt` | `Image#to_html` | No |
| Inline code span | `TextLine#inline_code` | Yes |
| Wiki-link display text | `TextLine#wiki_link` | Yes |

Working proof-of-concept payloads that survive rendering:

- A paragraph containing a `script` element is emitted verbatim inside the page.
- An image whose alt text is `" onerror="alert(1)` breaks out of the unescaped `alt` attribute and injects an event handler.
- A Markdown link whose visible text is an `img` element with an `onerror` handler renders that element inside the anchor.

One accidental, non-defensive quirk: the inline parser splits on parentheses, so a naive `javascript:` href is truncated at the first parenthesis — but percent-encoding, attribute breakout, or a `data:` URI bypasses this entirely. It is not a control.

The dynamic, script-rendered surfaces were also reviewed. The client-side full-text search loads a generated JSON index (`specifications_db.json`) and renders each hit; that rendering already uses safe DOM interfaces (`createTextNode` and property assignment), so raw markup in the index is shown as inert text and the search path is not vulnerable as shipped. The image-zoom modal, however, assigned the author-supplied image `alt` text to the caption via `innerHTML` — and because reading the `alt` property in script returns the *decoded* value, the per-context attribute escaping above does not close this: it is a **DOM-based XSS** in which alt markup is re-parsed as live HTML when a reader opens an image. The same class of risk would reappear in the search rendering if it were ever refactored to `innerHTML`.

Threat model: anyone who can land a `.md` file in any input repository — a contributor, a pull-request author — can plant script that runs in every reader's browser on the docs domain, enabling cookie/session theft, credential phishing on a trusted domain, and potential pivoting against authenticated sessions on the co-hosted Redmine. Severity is **High** for the published, multi-author deployment Almirah is built for; Low–Medium only where Markdown is authored by a single trusted operator and never published.

# Decision

Treat all authored Markdown content as untrusted text and encode it for its HTML context at the point of output. Introduce a single, consistently applied escaping mechanism rather than per-item ad-hoc handling, mirroring how `inline_code` already escapes.

## Text content escaping

Every run of literal text rendered into element content shall be HTML-escaped (the five characters: ampersand, less-than, greater-than, double-quote, single-quote) before interpolation. This covers paragraph text, heading text, blockquote text, Markdown table cells, and fenced code block lines. Escaping is applied to the **literal text token only**, after Markdown structure (emphasis, code, links, wiki links) has been recognised, so legitimate formatting tags Almirah itself emits are preserved while author-supplied markup is neutralised.

## Attribute value escaping

Values interpolated into HTML attributes — image `src` and `alt`, link `href` and visible text — shall be attribute-escaped so that a quote or angle bracket in the source cannot break out of the attribute and introduce new attributes or elements.

## URL scheme allow-list

For both Markdown links and images, the URL is admitted only if it is a relative path or carries an allowed scheme: `http`, `https`, or `mailto` (and relative/anchor references). Any other scheme — notably `javascript:`, `data:`, and `vbscript:` — is rejected and the link/image is rendered inert (as escaped text or a disabled reference) rather than emitting the dangerous URL.

## Generated anchor identifiers

Heading anchors are generated identifiers, but they are derived from the free-form heading text and emitted into `name`/`href` attributes. The slug shall therefore have the HTML-significant characters stripped so the anchor cannot break out of its attribute, applied once where the anchor is generated so the heading, its self-link, and the table of contents stay mutually consistent.

## Client-side rendering of search results and the image caption

Author-derived values displayed by client-side scripts shall be inserted using DOM text and attribute interfaces (`textContent` / `createTextNode` / property assignment), never by parsing strings as HTML (`innerHTML`), so a value that survived as inert text on the server cannot become live markup in the browser. Any URL a script assigns to a link shall pass the same scheme allow-list as the server-side renderers. Concretely, the image caption is set as text rather than HTML, and the search result rendering keeps its safe-DOM construction and validates the link target before assigning it. The generated search index continues to store the raw human-readable text, which is correct for a text-node renderer; escaping it would surface entity sequences as visible characters.

## Single mechanism

The escaping and scheme-checking helpers shall be defined once and reused by every doc item, so coverage cannot drift item-by-item the way it has. Adding a new renderer must go through the same helpers.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Done | 07-06-2026 | 07-06-2026 | New SRS items (SRS-096 onward) covering HTML-escaping of all rendered text content; attribute-value escaping for image and link attributes; a URL scheme allow-list for links and images; and safe DOM-based rendering of author-derived values in client-side scripts (SRS-099) |
| Code | Done | 07-06-2026 | 07-06-2026 | Add shared escape/attribute-escape and URL-scheme-check helpers in `Almirah.Code`; apply text escaping to the literal text token in `TextLine` and to `Heading`, `Blockquote`, `MarkdownTable`, and `CodeBlock` rendering; attribute-escape `Image` (`src`, `alt`) and `TextLine#link` (`href`, text); enforce the scheme allow-list in `link` and `Image`; route all of it through the single helper. Client-side: set the image caption with `textContent` instead of `innerHTML`, and keep the search rendering on safe-DOM construction with a client-side URL scheme allow-list on the result link |
| Tests | In-Progress | 07-06-2026 |  | Automated coverage in place: an end-to-end spec in `Almirah.Code` (`spec/e2e/xss_escaping_spec.rb`) asserts the generated HTML contains the escaped, inert form for every vector (paragraph, heading, blockquote, table header/cell, code block, image alt/src breakout, link text, link/image URL scheme) and that legitimate formatting/links/images still render; `HtmlSafe` unit specs; and `spec/client_rendering_safety_spec.rb` guarding the client-side safe-DOM and URL-allow-list invariants. Deferred: XSS fixture documents in `Almirah.TDS` and a referencing test protocol/run in `Almirah.Doc` |

# Out of Scope

- **Content Security Policy and HTTP security headers.** Emitting a CSP (for example `default-src 'self'; script-src 'self'`) and related headers is valuable defence-in-depth.
- **Supporting intentional raw HTML pass-through.** Almirah does not currently treat Markdown as allowing embedded HTML, and this ADR does not add an opt-in "trusted HTML" mode. If embedded HTML is ever wanted it must go through a vetted allow-list sanitiser, decided in its own record.
- **The structured `>[ID]` / `<REQ>…</REQ>` traceability links and their anchors**, which are produced from controlled identifiers, not free-form author text, and are unchanged. (Note: heading anchors *are* derived from free-form heading text and so are **in** scope — they are sanitised to a safe identifier by stripping HTML-significant characters from the slug, keeping the heading, its self-link, and the table of contents mutually consistent.)
- **Re-encoding or auditing already-published content**; the build simply renders safely from this version onward.

# Consequences

## Positive

- Closes the stored-XSS class across every confirmed vector with one mechanism; author Markdown can no longer inject executable code into the published site.
- Behaviour becomes consistent with the existing inline-code escaping instead of safe in two places and unsafe everywhere else.
- A single set of helpers means future renderers inherit the protection rather than reintroducing the gap.
- Closes the DOM-based XSS in the image-zoom caption and establishes a safe-DOM rendering invariant for the client-side search, guarded by a test so a later refactor cannot silently reopen it.

## Negative

- Touches the render path for essentially all text output, so the change has broad regression surface and needs thorough fixture coverage.
- Authors lose the (undocumented, unsafe) ability to drop literal HTML into Markdown; any content that relied on it will now show the markup as text.
- Strict attribute escaping and scheme checks can, in edge cases, alter how an unusual-but-benign URL renders.

## Neutral

- Escaping cost is negligible relative to overall build time.
- No change to the generated directory layout, output filenames, or traceability links.

# Alternatives Considered

- **Escape at the leaf (chosen).** Encode the literal text token and attribute values through shared helpers. Minimal, predictable, preserves Almirah-emitted formatting, and matches the existing inline-code approach.
- **Adopt a dedicated HTML sanitiser gem (such as Loofah or Sanitize) as a final output pass.** A reasonable and robust option; deferred because it adds a dependency and a post-processing stage over output Almirah fully controls, whereas escaping the few leaf points is sufficient and lighter. Kept open should an intentional embedded-HTML feature ever be introduced.
- **Rely on CSP / security headers only.** Rejected as the primary control: it is deployment-specific, does not protect local or differently-served builds, and leaves the output itself unsafe. Retained as complementary defence-in-depth, out of scope here.
- **Do nothing / treat authors as fully trusted.** Rejected: the framework's value proposition is a *published* multi-author site, and the same content also reaches GenAI tooling; silent raw HTML output is an unacceptable default.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.2 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall HTML-escape all author-supplied literal text rendered into element content — including paragraph, heading, blockquote, table-cell, and fenced code block text — so that markup present in the source Markdown is rendered as inert text and cannot introduce HTML elements. | >[SRS-096] |
| 2 | The software shall escape author-supplied values interpolated into HTML attributes — including an image's source and alternate text and a link's address and visible text — so that the value cannot terminate the attribute or introduce additional attributes or elements. | >[SRS-097] |
| 3 | The software shall admit a link or image URL only when it is a relative reference or uses an allowed scheme ("http", "https", or "mailto"), and shall render any other scheme (such as "javascript", "data", or "vbscript") inert rather than emitting it. | >[SRS-098] |
| 4 | The software shall render author-derived values displayed by client-side scripts — including search results and the image caption — using DOM text and attribute interfaces rather than HTML parsing, and shall admit a URL it assigns to a link only when it is a relative reference or uses an allowed scheme, so that indexed or displayed content cannot execute as script. | >[SRS-099] |

# References

- SRS-011 and SRS-012 in [srs.md](./../../specifications/srs/srs.md) — existing requirements for internal text links and heading links, whose rendering these escaping items harden
- [ADR-186](./../release%200.4.1/adr-186-cross-document-links.md) — cross-document linking; defines the `link` and `wiki_link` rendering paths affected here
- ISSUE-180 in [issue-180-inline-code-spans.md](./../release%200.4.0/issues/issue-180-inline-code-spans.md) — inline code span handling, the one text path already escaped (`CGI.escapeHTML`)

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()
