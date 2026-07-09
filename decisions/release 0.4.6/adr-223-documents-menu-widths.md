---
title: "ADR-223: Documents Menu and Risks Overview Column Widths"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 09-07-2026 | Proposed |
|   | 09-07-2026 | Accepted |
|   | 09-07-2026 | In-Progress |
| * | 09-07-2026 | Implemented |

# Context

Release 0.4.6 opens with three cosmetic rough edges on the rendered UI, all at the top level of a build:

- The risks summary page ([[adr-219-risks-menu-page]], polished by [[enh-221-risk-table-titles]]) renders its four numeric columns — Total Risks, Open Risks, Highest RPN, Average RPN — with no declared width, so the browser distributes the table unevenly per content. The Index page already solved this layout: every count column after the leading name column carries a fixed 7% width, leaving the name column the remainder.
- The top-menu link to the Index page is labelled "Index" — a term that reads as an implementation detail, while what the page actually lists is the project's documents.
- The same menu slot is unstable: on every page it is the "Index" item with the info icon, but on the Index page itself it swaps to a "Home" item with a home icon. The one button that should anchor the menu changes identity depending on where the reader stands, unlike the Decision Records and Risks entries, which render the same everywhere and simply self-link on their own page.

# Decision

Three presentation changes, no behavioural ones.

- **Equal count-column widths on the risks summary.** The Risks summary table gives every column after the leading Risk Registry column a fixed 7% width, following the Index page's pattern of inline widths on the data cells. The Risk Registry column keeps the remaining width; the column set, order, contents, links and colouring are untouched.
- **Rename the Index menu item to "Documents".** The top-menu entry leading to the Index page reads "Documents" on every rendered page, carrying the home icon — the anchor-of-the-menu icon the Index-page variant already used. The element id stays `index_menu_item`, so existing styling and tests keyed on the id are unaffected. The hardcoded copy of the link in the source-file page renderer is renamed with it.
- **One stable menu item, no Home swap.** The Index page renders the same "Documents" item as every other page, self-linking to the Index — matching how the Decision Records and Risks entries behave on their own overview pages. The separate "Home" item and its renderer are removed; its home icon lives on in the Documents item.

# Scope

| # | Item | Owner | Depends On | Est (focused) | Est (safe) | Status | Start Date | Target Date | Description |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Requirements | BA |  | 0.5 | 1 | Done | 09-07-2026 | 09-07-2026 | State SRS-173: a "Documents" link with the home icon in the top navigation bar of every rendered page, including the Index page itself, leading to the Index page |
| 2 | Code | DEV |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | Add the 7% inline widths to the risks summary count cells; rename the index link label to Documents with the home icon in the base document and the source-file renderer; drop the Home variant and render the Documents item unconditionally |
| 3 | Tests | TEST |  | 1 | 2 | Done | 09-07-2026 | 09-07-2026 | Extend the `Almirah.Code` e2e specs: the summary count cells carry the 7% width; every page including the Index shows the Documents item with the home icon and correct href; no page shows a Home item |

# Out of Scope

- **The element id.** `index_menu_item` stays; renaming it would break user styling for a label change.
- **The Index page content and its own 7% columns.** The page whose pattern is being copied is not touched.
- **Other menu entries.** Decision Records and Risks keep their labels, icons and conditional presence.
- **The index.html filename.** The page keeps its name and URL; only the menu label changes.

# Consequences

## Positive

- The risks summary reads like the Index page: one wide name column, evenly slim count columns, regardless of content length.
- The menu is stable — the same items, in the same form, on every page — and "Documents" says what the page holds instead of how it is built.

## Negative

- Readers used to the "Index"/"Home" labels must relearn one word; screenshots in older material show the old label.

## Neutral

- On the Index page the Documents item links to the page itself, exactly as the Risks item does on the risks summary — an established pattern, not a new one.

# Alternatives Considered

- **A CSS class instead of inline widths.** Rejected for consistency: the Index page sets these widths inline on the data cells; introducing a second mechanism for the same visual rule would leave two ways to do one thing.
- **Renaming the element id to match the new label.** Rejected: the id is an interface for styling and tests; the label is presentation. Changing both couples a cosmetic rename to downstream breakage.
- **Hiding the menu item on the Index page instead of self-linking.** Rejected: a menu that loses an entry depending on the current page is the instability this decision removes; the collection entries already self-link on their own pages.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.5 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.6 |

# Affected Documents

This decision adds SRS-173. The column widths are presentation styling with no requirement, following the ENH-214 precedent.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | The software shall provide a clickable "Documents" link, carrying the home icon, in the top navigation bar of every rendered page, including the Index page itself, leading to the Index page. | >[SRS-173] |

# References

- [[adr-219-risks-menu-page]] — the risks summary table whose count columns get the fixed widths; the self-linking menu precedent
- [[enh-221-risk-table-titles]] — the previous presentation polish of the same page
- [[adr-170-introduce-decision-records]] — the decisions menu entry whose stable form the Documents item adopts

# Review Evidences

- [Decision Record]()
- [Requirements]()
- [Code]()
- [Tests]()

# Effort

| Date | Item | Owner | Hours | Note |
|---|---|---|---|---|
| 09-07-2026 | Requirements | BA | 0.5 | Initial Proposal |
| 09-07-2026 | Requirements | BA | 0.25 | SRS-173 stated |
| 09-07-2026 | Code | DEV | 0.5 | Widths, Documents label, Home variant dropped |
| 09-07-2026 | Tests | TEST | 0.5 | e2e extended, suite 331 green |
| 09-07-2026 | Code | DEV | 0.25 | Documents item icon switched from info to home |
