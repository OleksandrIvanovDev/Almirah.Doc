---
title: "ADR-227: Images in Decisions and Risks"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

Specifications and test protocols support images through one authoring rule: the image files go into an `img` folder next to the Markdown file, and the build copies that folder into the same place under `build/`, so a relative reference like `img/diagram.png` resolves identically in the source Markdown and in the rendered HTML. The copy is keyed on the document-id folder — `specifications/<doc-id>/img` and `tests/protocols/<doc-id>/img` — because those document kinds keep each document in its own folder named after its id.

Decision records and risk records render through the same Markdown pipeline, and the image renderer already emits the author's relative path verbatim into the HTML. Every decision and risk page is also written to the same relative location under `build/` as its source file (`decisions/<subfolder>/adr-NNN.md` renders to `build/decisions/<subfolder>/adr-NNN.html`, and likewise under `risks/`). So a relative image reference in a decision or risk record already renders as a correct relative `src` — but the image file itself is never copied into `build/`, and the picture shows as broken. There is no way today to put a screenshot into an issue record, a sketch into an ADR, or a Fault Tree Analysis or Security Threat Modeling diagram into a risk registry's `overview.md` preface.

The specification rule cannot be reused verbatim, because the folder layout differs: decision records share release folders (many records per folder), and risk records share registry folders, so there is no per-document folder to key the copy on.

# Decision

The build shall copy every folder named `img` found anywhere under `decisions/` and under `risks/` into the same relative location under `build/`, preserving the folder's contents. Nothing else changes: the parser, the linker, and the image renderer already behave correctly; the change closes the file-copy gap only.

The authoring rule therefore becomes uniform across all document kinds: put the image into an `img` folder next to the Markdown file and reference it relatively. For the shared-folder layouts this means one `img` folder per release folder or per registry, shared by the records in it:

```
decisions/release 0.4.6/adr-227-decision-risk-images.md
decisions/release 0.4.6/img/copy-rule-sketch.png
risks/security/overview.md
risks/security/img/threat-model.png
risks/security/r-001-injection.md
```

A registry preface renders to `build/risks/<registry>/overview.html` — the same folder as its source — so diagrams referenced from the preface appear on the registry page with no extra provisions.

The copy runs during rendering, alongside the existing specification and protocol copies. The build folder is deleted at the start of every run, so removed images disappear from the output without any staleness handling.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | State SRS-176: img folders under decisions and risks are copied into the build preserving relative location, so relative image references in decision records, risk records, and registry prefaces resolve in the rendered HTML |
| 2 | Code | DEV | 1 | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | One copy helper in the project pipeline globbing img folders under decisions and risks and copying each to the same relative path under build, called from both build entry points |
| 3 | Tests | TEST | 2 | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | E2e specs with inline fixtures covering every placement: img beside a root-level record, in a shared release folder, in a deeper subfolder, unreferenced, with subfolders and non-image files, in a registry, nested inside a registry, directly under risks, plus the no-img and file-named-img cases |
| 4 | Manual Test Data | TEST | 2 | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | Almirah.TDS demo-risks-fta branch: five risks derived by Fault Tree Analysis of two top events, the fault-tree SVG diagrams shown on the registry overview page from the registry img folder |

# Out of Scope

- **Images on the all-registries risks overview page.** The summary page at `build/risks/overview.html` shows only each registry's frontmatter title, not the preface body, so no diagram can appear there; placing preface content on that page would be a separate decision (the relative depth also differs there).
- **Copying only referenced images.** The whole `img` folder is copied whether or not every file is referenced — the same contract specifications and protocols have today.
- **Other asset kinds.** Only folders named `img` are copied; arbitrary attachments (PDFs and similar) stay unsupported everywhere, as today.
- **Changing the specification and protocol copy rule.** The existing doc-id-keyed copies stay as they are.

# Consequences

## Positive

- The one authoring rule for images becomes uniform across specifications, protocols, decisions, and risks — nothing new to learn.
- Issue records can carry screenshots, ADRs can carry sketches, and risk registries can carry Security Threat Modeling or Fault Tree Analysis diagrams in their prefaces.
- Records in one release folder or registry share one `img` folder, which suits diagrams referenced by several related records.
- No security surface is added: the existing URL scheme filtering and attribute escaping (SRS-097, SRS-098) apply to decision and risk images exactly as to specification images.

## Negative

- A subfolder literally named `img` that itself contains Markdown files would be both parsed for records and copied as images; the convention is documented rather than guarded against.

## Neutral

- The decisions/risks rule is folder-based ("any img folder, preserved relatively") while the specification rule is doc-id-based; from the author's point of view both read as "put images in an img folder next to the Markdown file".
- Build output grows by the size of the image folders; the build folder is recreated on every run, so no cleanup logic is needed.

# Alternatives Considered

- **Reusing the doc-id-keyed rule (`decisions/<record-id>/img`).** Rejected: decision and risk records deliberately share folders; forcing one folder per record would break the established layout of release folders and registries.
- **Copying the whole decisions and risks trees except Markdown files.** Rejected: it would silently publish any stray file into the build output and blur the one clear rule authors already know.
- **Copying only images actually referenced from the Markdown.** Rejected: it requires resolving every image path during parsing and diverges from the existing specification behaviour for no user-visible benefit.
- **Rendering preface bodies (and their diagrams) on the all-registries overview page.** Rejected here: that page aggregates registries by title on purpose, and relative image paths would need depth rewriting; kept out of scope as its own possible future decision.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.5 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.6 |

# Affected Documents

This decision adds SRS-176.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall copy every folder named img located under the decisions folder and under the risks folder into the same relative location under the build folder, so that image files referenced through relative paths from decision records, risk records, and risk registry prefaces resolve in the rendered HTML. | >[SRS-176] |

# References

- [[adr-215-risk-record-collection]] — risk records and their registry folder layout
- [[adr-216-risk-register-table]] — the registry page rendering the preface, where FTA and threat-model diagrams would appear
- [[adr-188-html-output-escaping]] — the URL scheme filtering and escaping that already covers images in every document kind

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 09-07-2026 | Requirements | BA | 0.5 | Initial Proposal |
| 09-07-2026 | Requirements | BA | 0.25 | SRS-176 stated under a new Decision and Risk Images section |
| 09-07-2026 | Code | DEV | 0.5 | copy_decision_and_risk_images helper in the project pipeline, called from both build entry points |
| 09-07-2026 | Tests | TEST | 1 | decision_risk_images_spec e2e added covering all placements, suite 354 green, rubocop clean |
| 09-07-2026 | Manual Test Data | TEST | 1 | TDS demo-risks-fta branch created: FTA register with two fault-tree diagrams on the overview, five cut-set records, verified with the installed gem |
