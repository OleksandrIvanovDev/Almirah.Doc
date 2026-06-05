---
title: "ADR-187: Decisions Example in Project Template"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 05-06-2026 | Proposed |
|   | 05-06-2026 | Accepted |
|   | 05-06-2026 | In-Progress |
| * | 05-06-2026 | Implemented |

# Context

`almirah create <project_name>` scaffolds a starter project from `ProjectTemplate` (`lib/almirah/project_template.rb`). Today it writes:

- `specifications/req/req.md` and `specifications/arch/arch.md` (with example controlled items and an `>[REQ-001]` uplink);
- three test protocols under `tests/protocols/` (`tp-001`, `tp-002`, `tq-001`);
- three test runs under `tests/runs/` (`001/tp-001`, `001/tp-002`, `010/tq-001`).

Running `almirah please` on this scaffold produces specifications, test protocols, the traceability matrix, and the coverage matrix — a working tour of those features. It does **not** create a `decisions/` folder.

Decision records (ADRs, issues, enhancements) are a core, first-class feature of the framework — they are parsed from `<project>/decisions/`, rendered as `Decision` documents, linked to specifications via their "Affected Documents" section, and summarised on the generated `build/decisions/overview.html` page (SRS-039, SRS-052; see the `Decision` and `DecisionsOverview` document types). Yet a project created from the template contains no decision record at all. As a consequence:

- A new user has no in-template example of the decision-record format (frontmatter `title`, the `# Status` table with the `*` current-state marker, the section structure, and the "Affected Documents" uplink syntax) and must read the documentation or an existing project to discover it.
- The `decisions/overview.html` page is only generated when at least one decision record exists, so the default `create` → `please` flow never exercises or showcases the decisions overview — one of the framework's distinguishing outputs is invisible in the starter project.

The template is meant to be a self-documenting tour of what Almirah does. Omitting decision records leaves a prominent feature out of that tour.

# Decision

Extend `ProjectTemplate` to also scaffold a `decisions/` directory containing one example decision record, so that a freshly created project demonstrates the decision-record feature end to end and renders the decisions overview page on its first `please` build.

## Scaffolded file

`ProjectTemplate#initialize` gains a `create_decisions` step (alongside the existing `create_requirements`, `create_architecture`, `create_tests`, `create_test_runs`) that writes a single example record:

```
<project>/decisions/adr-001-start-project-decision.md
```

The filename follows the decision-record convention (`<letters>-<digits>-<slug>.md`, ID derived from the `adr-001` prefix; slug kept to three words at most).

## Example content

The example record is minimal but complete enough to teach the format, and it links back to the example requirement the template already creates (`REQ-001`) so the build produces a real decision → specification link and a populated overview page:

```markdown
---
title: "ADR-001: Start Project Decision"
---

# Status

|  | Date | Status |
|:---:|---|---|
| * | <creation date> | Proposed |

# Context

This is an example decision record. It demonstrates how Almirah captures a
decision and links it to the requirements it affects.

# Decision

Describe the decision here. The leading "*" in the Status table marks the
current state of the record.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | To Do | | | |
| Code | To Do | | | |
| Tests | To Do | | | |

# Out of Scope

The ADR contains the most important sections with no detailed content.

# Consequences

## Positive

An ADR example will clearly show the approach Almirah framework use for any software changes.

## Negative

The format of this ADR can be different that is required for the real project.

## Neutral

TBD

# Alternatives Considered

- **Add an empty `decisions/` directory only.** Rejected: an empty folder neither teaches the format nor causes the overview page to render, so it would not exercise the feature.

# Affected Documents

Table below shows a requirement whose text this decision creates or updates.

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | This is a first requirement (controlled paragraph with ID equal to "REQ-001"). | >[REQ-001] |

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | n/a |
| Issue Found in Version | n/a |
| Target Release Version | 0.0.1 |

# References

TBD

# Review Evidences

TBD
```

The creation date is filled at scaffold time via `Time.now.strftime('%d-%m-%Y')`, matching the `DD-MM-YYYY` format used by every decision-record `# Status` table. (Note: this differs from the `%Y-%d-%m` format the existing template uses for the document-history date; the Status table follows the decision-record convention, not the document-history one.)

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | To Do | 05-06-2026 | 05-06-2026 | No new SRS items are added by this ADR; the example demonstrates the existing decision-record requirements (SRS-039, SRS-052). Formal SRS coverage of the `create` command's template contents is tracked separately (see Out of Scope). |
| Code | Done | 05-06-2026 | 05-06-2026 | Added a `create_decisions` method to `lib/almirah/project_template.rb` writing `decisions/adr-001-start-project-decision.md` with the example content above, filling the Status-table date via `Time.now.strftime('%d-%m-%Y')`; invoked it from `ProjectTemplate#initialize`. |
| Tests | Done | 05-06-2026 | 05-06-2026 | Added `spec/e2e/create_template_spec.rb`: `almirah create` produces `decisions/adr-001-start-project-decision.md`, and a subsequent `almirah please` renders `build/decisions/overview.html`, lists the record, renders `adr-001.html`, and links the decision to `REQ-001` from the requirements page. |

# Out of Scope

- Generating a `project.yml` from the `create` command. The scaffold still omits one; that is a separate gap and is not addressed here. Its absence does not block this change: specifications and decisions are discovered by directory scan during `please` independent of `project.yml`, so the example decision still parses, links to `REQ-001`, and renders the overview (the missing file produces only a non-fatal warning).
- Adding example **issue** or **enhancement** records (`issues/`, `enhancements/`). A single ADR is sufficient to demonstrate the format and populate the overview; further example types can be added later.
- Adding formal SRS requirements and a regression suite for the full contents of the `create` template. The command currently has no dedicated SRS coverage; specifying it is a broader effort kept out of this change.

# Consequences

## Positive

- A project created from the template now demonstrates the decision-record feature: the format (frontmatter title, `# Status` table with the `*` marker, sections, "Affected Documents" uplink) is shown by example.
- The default `create` → `please` flow renders `build/decisions/overview.html`, so the decisions overview — a distinguishing framework output — is visible from the very first build.
- The example record links to `REQ-001`, so the generated project also shows a live decision → specification link, reinforcing the traceability story.

## Negative

- One more file to keep in step with the documented decision-record conventions; if the format evolves, the template example must be updated too.

## Neutral

- The addition is purely template content; it has no effect on existing projects or on the `please`/`combine` pipelines.
- A created project gains a `decisions/` directory by default, which a user who does not want decision records would delete — the same as any other example folder the template creates.

# Alternatives Considered

- **Leave the template as-is and document the decision-record format only in prose.** Rejected: the template is intended to be a self-documenting tour; omitting a core feature from it is the gap this ADR closes.
- **Scaffold example records of every type (ADR, issue, enhancement).** Rejected for now: a single ADR is enough to teach the format and populate the overview; multiple example types add clutter to the starter project for little additional teaching value.
- **Add an empty `decisions/` directory only.** Rejected: an empty folder neither teaches the format nor causes the overview page to render, so it would not exercise the feature.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.1 |

# Affected Documents

No requirement text is created or updated by this ADR.

# References

N/A

# Review Evidences

- [Decision Record](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Requirements](https://github.com/OleksandrIvanovDev/Almirah.Doc/pull/29)
- [Code](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
- [Tests](https://github.com/OleksandrIvanovDev/Almirah.Code/pull/48)
