---
title: "ISSUE-183: Orama Search Broken on Index Page"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 26-05-2026 | Proposed |
|   | 27-05-2026 | Accepted |
|   | 27-05-2026 | In-Progress |
| * | 27-05-2026 | Implemented |

# Context

The search box on the rendered Index page does nothing. Typing in the field neither opens the dropdown nor returns results.

The root cause is a broken module import. [orama_search.js:2](./../../../Almirah.Code/lib/almirah/templates/scripts/orama_search.js#L2) imports `create`, `search`, and `insert` from:

```
https://unpkg.com/@orama/orama@latest/dist/index.js
```

The `@latest` tag now resolves to Orama 3.1.18, which reorganised its build output into per-target subfolders (`dist/browser/index.js`, `dist/esm/index.js`, `dist/commonjs/index.js`, `dist/deno/index.js`). The flat `dist/index.js` entry no longer exists, so unpkg's redirect lands on a 404. The module fails to load, which means the `addEventListener` calls at the bottom of the script ([orama_search.js:124](./../../../Almirah.Code/lib/almirah/templates/scripts/orama_search.js#L124) and [orama_search.js:127](./../../../Almirah.Code/lib/almirah/templates/scripts/orama_search.js#L127)) never run and the search input is inert.

The proximate trigger is that the script pins to `@latest`. The same code worked while Orama 1.x and 2.x were resolving on that tag; an upstream major-version release broke every previously-rendered Almirah project retroactively.

Two further defects in the same script will surface once the import is restored:

1. **Schema vs. payload field mismatch.** The schema in [orama_search.js:7](./../../../Almirah.Code/lib/almirah/templates/scripts/orama_search.js#L7) declares a `doc_title` property, but [specifications_db.rb:19](./../../../Almirah.Code/lib/almirah/search/specifications_db.rb#L19) emits the document title under the key `document`, and the JS insert at [orama_search.js:21-27](./../../../Almirah.Code/lib/almirah/templates/scripts/orama_search.js#L21-L27) re-passes `document`. The schema field stays empty, and Orama 3.x is strict about unknown properties — inserts can throw, or at minimum the title column is never indexed.
2. **Possible API drift in Orama 3.x.** Between 1.x and 3.x the `create({ schema })` configuration shape and `search(...)` options changed (notably `properties` and `exact`). The current call sites are unlikely to be valid against 3.x without adjustment.

# Decision

Restore search by pinning Orama to a known-good version, importing it from a path that exists in that version, and aligning the schema and payload field names. Then validate the `create` and `search` call shapes against the pinned Orama version's API.

## Import path and version pinning

Replace the `@latest` import with an explicit version and the new browser-bundle path:

```
import { create, search, insert } from 'https://unpkg.com/@orama/orama@3.1.18/dist/browser/index.js'
```

Pinning prevents a future upstream major from breaking already-rendered projects. The `dist/browser/index.js` path is the entry the package exposes for in-browser ESM consumption in 3.x.

## Field name alignment

Pick a single name for the document-title field and use it in three places consistently:

- The JSON producer in `lib/almirah/search/specifications_db.rb` (the hash key currently named `document`).
- The Orama schema in `lib/almirah/templates/scripts/orama_search.js`.
- The insert payload and the result-rendering accessor in the same JS file.

`doc_title` (already in the schema) is the natural choice; renaming the producer key is the smaller change in line count, and avoids broadening the JSON's notion of "document" beyond the title string.

## API compatibility

After the import is fixed, the `create({ schema })`, `insert(db, ...)`, and `search(db, { term, properties, exact })` call sites are verified against the 3.x API and adjusted in place if any option has been renamed or restructured. No new search features are introduced; the goal is parity with the previously-working behaviour.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Code | Implemented | 26-05-2026 | 27-05-2026 | Update the import URL in `orama_search.js` to a pinned `@orama/orama@3.1.18` and the `dist/browser/index.js` path; rename the title field to `doc_title` in `specifications_db.rb`, the schema, the insert payload, and the result-rendering accessor; reconcile the `create`/`insert`/`search` call shapes with the 3.x API |
| Tests | Implemented | 26-05-2026 | 27-05-2026 | Manual verification on the rendered Index page of `Almirah.Doc`: search returns results for a known term, the dropdown opens on focus, results render with title, color, heading link, and snippet |

# Out of Scope

- **Bundling Orama into the gem.** Hosting the library at a fixed path inside the rendered project (instead of fetching from a CDN) would remove the upstream-availability dependency entirely. Worth doing, but a larger change than this fix needs; tracked as a follow-up.
- **Replacing Orama with a different search library.** The defect is operational, not architectural.
- **Indexing additional document types** (e.g., decision records, test protocols) in the search DB. The current build only indexes specification paragraphs and table rows; broadening that is independent of restoring the existing behaviour.
- **CSS or UX changes to the search dropdown.**

# Consequences

## Positive

- Search on the Index page works again for every newly rendered Almirah project.
- Pinning the version eliminates the class of regression where an upstream release breaks already-deployed sites that nobody is re-rendering.
- The field-name alignment closes a latent inconsistency that would have caused subtle indexing problems even before the 3.x layout change.

## Negative

- Pinning a CDN-hosted dependency means version upgrades become a deliberate change in the gem rather than an automatic pickup. This is the desired trade-off here, but it does add a small maintenance surface (someone has to remember to refresh the pin when a meaningful Orama upgrade lands).

## Neutral

- The user-facing search UI, result layout, and styling are unchanged.
- The JSON producer's output shape changes only by renaming one key. Any external tooling that consumed `specifications_db.json` and read the `document` key would need a matching rename, but there is no such known consumer.

# Alternatives Considered

- **Keep the `@latest` import and live with the breakage until the upstream layout stabilises.** Rejected: the layout is unlikely to revert, and continuing to track `@latest` re-creates the same risk on the next major.
- **Pin to the last Orama 1.x or 2.x version where `dist/index.js` still exists.** Rejected: pins old security fixes out and trades one stale-dependency problem for another. Pinning to a current 3.x release keeps the project on a supported line.
- **Inline the Orama library into the gem's templates and serve it locally from each rendered project.** Rejected for this fix as out of scope (see above); reconsidered later if CDN availability proves unreliable.
- **Rename the schema field to `document` instead of renaming the payload key to `doc_title`.** Rejected: the schema name `doc_title` is more descriptive than `document` (which reads as if it might be the whole document body) and is the form already present in the in-tree script, so renaming the producer is the smaller and clearer change.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | 0.4.0 |
| Target Release Version | 0.4.1 |

# References

- [ISSUE-180](./issue-180-inline-code-spans.md) — most recent issue decision record, used as the structural template for this one
- [ADR-170](./../adr-170-introduce-decision-records.md) — the decision-record convention this ISSUE follows

# Review Evidences

- Decision Record:
- Code:
- Tests:
