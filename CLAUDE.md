# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the documentation repository for the **Almirah framework** — a requirements and test management system for software projects. All content is Markdown-based specification documents and test protocols; there is no application source code here.

The companion code repository is at `../Almirah.Code/lib` (referenced in `project.yml`).

## Building

Almirah processes specification documents using the Ruby `almirah` gem and outputs HTML:

```bash
almirah please <absolute-path-to-project-folder>
```

This generates a `build/` folder containing rendered HTML versions of all specifications.

## Repository Structure

```
specifications/       # Specification documents (one folder per document)
  arch/               # Architecture specification (primary input per project.yml)
    arch.md           # Main document
    img/*.svg         # Rendered diagrams
    src/*.puml        # PlantUML diagram sources
  srs/srs.md          # Software Requirements Specification
  sys/sys.md          # System-level specification
  rfec/rfec.md        # Requirements for External Components
tests/
  protocols/          # Reusable test protocol templates (tp-NNN/)
  runs/               # Test execution records (NNN/tp-NNN/)
decisions/            # Decision records (see "Decision Records" below)
  adr-NNN-*.md        # Architecture Decision Records (top level)
  issues/             # Bug/defect decision records
    issue-NNN-*.md
  enhancements/       # Enhancement proposals
    enh-NNN-*.md
project.yml           # Declares specification inputs and linked repositories
```

## Specification Authoring Conventions

**Controlled Items** are requirement paragraphs identified by a unique ID at the start:

```
[AAA-NNN] Requirement text here.
```

**Traceability links** to items in another specification are placed at the end of a controlled item using `>`:

```
[SRS-001] The software shall allow creating Controlled Items. >[ARCH-005], >[SYS-005]
```

- The prefix letters (`AAA`) identify the owning specification (e.g., `SRS`, `ARCH`, `SYS`, `RFEC`).
- Each controlled item ID must be unique within its specification.
- Uplinks (`>[BBB-NNN]`) point to parent items in higher-level specs; Almirah renders coverage status from these links.

**Document frontmatter** uses either YAML (`---`) or Pandoc-style (`%`) headers. YAML frontmatter supports `title`, `revision`, `date`, and `author` fields.

**Inline code spans** — only **single-backtick** pairs are recognised. Do **not** use double-backtick (or any multi-backtick) syntax to embed a literal backtick character inside a code span (the CommonMark form). Almirah's parser pairs every backtick with the next one and would scramble the rest of the line. To refer to the backtick character in prose, write the word "backtick" instead, or describe the syntax (e.g., "three consecutive backticks" for a fenced code block).

## Test Protocols

Test protocols (`tests/protocols/tp-NNN/tp-NNN.md`) are templates containing test steps with columns: Test Step, Description, Expected Output, Actual Output, Result, Req-ID.

Test runs (`tests/runs/NNN/tp-NNN/tp-NNN.md`) are copies of protocols filled in with actual results (`pass`/`fail`) and evidence images.

The `Req-ID` column in test steps uses the same `>[AAA-NNN]` traceability syntax to link steps back to requirements.

## Decision Records

Decision records capture decisions made about the project — architectural choices, bug-fix decisions, enhancement proposals. They live under `decisions/` and are processed by Almirah alongside specifications.

**Filename convention**: `<letters>-<digits>-<descriptive-slug>.md`. The ID is derived from the `<letters>-<digits>` prefix (e.g., `adr-170-introduce-decision-records.md` → ID `adr-170`). Type is the uppercased letter prefix (`ADR`, `ISSUE`, `ENH`); sequence number is the digits.

**Slug length**: keep the descriptive slug (the part after the ID) to **three words at most**. For example, prefer `adr-181-overview-release-version.md` over `adr-181-overview-target-release-version.md`. The full meaning belongs in the frontmatter `title:` field; the filename is just a stable handle.

**Frontmatter**: every decision record should carry a YAML `title:` field, since this is what the rendered HTML and the overview page display. Without it, the title falls back to `<id>.md`.

```yaml
---
title: "ADR-170: Introduce Decision Records"
---
```

**Status table**: each record starts with a `# Status` section containing a table. The leading column carries `*` in exactly one row to indicate the current state. Future-dated rows may be written upfront for planning.

```markdown
|  | Date | Status |
|:---:|---|---|
|   | 14-05-2026 | Proposed |
|   | 14-05-2026 | Accepted |
| * | 15-05-2026 | In-Progress |
|   | 24-05-2026 | Implemented |
```

**Sections**: follow the same structure as ADR-170 — Status, Context, Decision, Scope, Out of Scope, Consequences (Positive/Negative/Neutral), Alternatives Considered, Software Versions, References.

## project.yml

Declares which specification serves as the top-level input and which external repositories contain linked source code:

```yaml
specifications:
  input:
    - arch          # folder name under specifications/

repositories:
  - name: Almirah.Code
    path: ../Almirah.Code/lib
```
