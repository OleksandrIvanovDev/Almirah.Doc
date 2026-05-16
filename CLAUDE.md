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

## Test Protocols

Test protocols (`tests/protocols/tp-NNN/tp-NNN.md`) are templates containing test steps with columns: Test Step, Description, Expected Output, Actual Output, Result, Req-ID.

Test runs (`tests/runs/NNN/tp-NNN/tp-NNN.md`) are copies of protocols filled in with actual results (`pass`/`fail`) and evidence images.

The `Req-ID` column in test steps uses the same `>[AAA-NNN]` traceability syntax to link steps back to requirements.

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
