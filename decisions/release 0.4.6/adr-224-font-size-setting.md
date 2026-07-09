---
title: "ADR-224: Base Font Size Setting"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

Every rendered page carries its base text at a hardcoded 12px, set on the body element in the single static stylesheet main.css that the build copies verbatim into build/css. Projects aimed at different audiences or read on different displays have no way to make the text larger or smaller without hand-editing the generated CSS after every build.

Two facts of the current stylesheet shape any solution:

- All heading sizes are declared in em units — h1 at 2em down to h5 and h6 at 1em — and em resolves against the inherited body size. So does the top menu bar with the document title (1.5em) and the footer (0.9em). Changing the body size today would scale all of these along with it.
- Everything else — table cells, code blocks, blockquotes, the left navigation pane tree — declares no font size at all and simply inherits from the body. The secondary stylesheet search.css and the Rouge theme injected into source pages set no font sizes either.

The desired behaviour is a single number in project.yml that scales the reading text — paragraphs, tables, code, and the left navigation pane — while the headings and the page chrome (top menu bar, footer) keep their current fixed sizes.

# Decision

An optional top-level `font_size` key in project.yml, one integer, the base text size in pixels. When absent, pages render exactly as today.

- **A CSS custom property carries the setting.** In main.css the body rule becomes `font-size: var(--almirah-font-size, 12px)` — the fallback keeps the stylesheet self-contained and the default rendering unchanged. When `font_size` is configured, each generated page emits one inline style line after the main.css link, setting `--almirah-font-size` on the root element. Both page writers emit it: the shared base-document writer and the separate source-file writer, which duplicates the head-assembly logic (as ADR-223 already found when renaming the menu link there).
- **Headings are pinned to their current pixel sizes.** The six heading rules change from em to the pixel values they compute to today against the 12px body: h1 24px, h2 21.6px, h3 18px, h4 14.4px, h5 12px, h6 12px. The exact fractional values are kept so the default rendering stays pixel-identical. This pinning is what makes the setting body-text-only: with em sizes the headings would scale along with the body.
- **The page chrome is pinned the same way.** The two top-menu link rules change from 1.5em to 18px and the footer from 0.9em to 10.8px, so the menu bar, document title and footer keep their size whatever the setting says.
- **The left navigation pane scales with the setting.** The pane's document tree declares no font size and inherits from the body; that inheritance is kept deliberately, so the navigation text follows the configured base size just like the content it points at. Tables, code blocks, blockquotes and the image-modal caption scale the same way, for the same reason — no new rules needed for any of them.

Example configuration:

```yaml
font_size: 14
```

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | State SRS-174: an optional project.yml font size number scaling body text and the navigation pane while headings, top menu bar and footer keep their sizes |
| 2 | Code | DEV |  | 2 | 3 | Done | 09-07-2026 | 09-07-2026 | Read `font_size` in the project configuration; emit the root-element custom-property override from both page writers; convert the body rule to the custom property and pin the heading, top-menu and footer rules to pixels in main.css |
| 3 | Tests | TEST |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | Extend the `Almirah.Code` e2e specs: a project with `font_size` set carries the inline override on regular and source pages; a project without it emits no override; the copied main.css pins headings in pixels |

# Out of Scope

- **Font family.** Only the size is configurable; the Verdana stack stays hardcoded.
- **Heading, menu and footer sizes.** The setting is one number for the reading text; the pinned sizes are not configurable.
- **Per-document overrides.** The setting is project-wide; frontmatter cannot change it.
- **The search dropdown and the Rouge highlight theme.** Neither declares font sizes today and neither is touched.

# Consequences

## Positive

- One number in project.yml sizes the reading text for the project's audience, and the left navigation pane follows it, so the tree stays proportionate to the content it navigates.
- Projects without the setting render pixel-identical to today — the custom-property fallback and the exact pixel pins preserve the current output.

## Negative

- Heading sizes no longer track the body size. A future decision to scale headings too would revisit the pinned values.
- The em-to-pixel pinning bakes today's 12px-derived sizes into main.css; the fractional values (21.6px, 14.4px, 10.8px) look odd in the stylesheet, though they are exactly what browsers compute today.

## Neutral

- The inline override is one style line per page, emitted only when the setting exists; the stylesheet itself remains a static copied asset.

# Alternatives Considered

- **Templating main.css during the resource copy.** Rejected: it turns a static asset into a template and makes every build's CSS differ per project, complicating caching and comparison; the inline override keeps main.css identical everywhere and valid on its own.
- **Scoping the setting to the content area instead of the body.** Rejected: the navigation pane sits outside the content element and would stop scaling — the opposite of what is wanted — while the headings, being em-sized inside the content, would still need pixel pinning anyway.
- **Rounding the pinned sizes to whole pixels.** Rejected: 22px and 14px would visibly change the default rendering for every existing project; keeping the computed fractional values makes the change invisible by default.
- **A body-level em or percentage setting.** Rejected: a bare number of pixels is the simplest contract for a one-number setting; relative units would reintroduce the coupling to ancestor sizes the pinning removes.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.5 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.6 |

# Affected Documents

This decision adds SRS-174. The pixel pinning of headings, menu and footer is presentation styling carried by the same requirement's "unchanged" clause.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall support an optional font_size number in project.yml setting the base text size in pixels of every rendered page, scaling the body text and the navigation pane while heading, top menu bar and footer sizes remain unchanged; when the setting is absent the pages shall render as without the feature. | >[SRS-174] |

# References

- [[adr-223-documents-menu-widths]] — the previous change touching both page writers, where the source-file renderer's duplicated head assembly was noted

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 09-07-2026 | Requirements | BA | 0.5 | Initial Proposal |
| 09-07-2026 | Requirements | BA | 0.25 | SRS-174 stated under a new Appearance section |
| 09-07-2026 | Code | DEV | 1 | Configuration getter, class-level setting, both page writers, main.css var and pixel pinning |
| 09-07-2026 | Tests | TEST | 0.5 | font_size e2e spec added, suite 337 green, rubocop clean |
