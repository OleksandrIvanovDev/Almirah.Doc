---
title: "ADR-184: Concise Console Progress Output"
---

# Status

|  | Date | Status |
|:---:|---|---|
|   | 31-05-2026 | Proposed |
|   | 31-05-2026 | Accepted |
| * | 31-05-2026 | In-Progress |
|   | 31-05-2026 | Implemented |

# Context

Running `almirah please <project>` printed one block of output per processed document. Each `Specification` emitted its title plus a four-row statistics table (Number of Controlled Items, Items w/ Up-links, Items w/ Down-links, …); every `Index`, `Traceability`, `Coverage`, `Implementation`, `Decision`, `DecisionsOverview`, and `SourceFile` printed its own coloured line via a `to_console` method. In addition, the `SourceFile` constructor carried a stray `puts @specifications_path` debug statement that printed an internal relative path for **every** source file discovered in a linked repository.

The result was console output whose volume scaled with the size of the project **and** with the number of files in any linked source repository. On a real project the actual outcome — did it finish, where is the output, were there problems — was buried under hundreds of per-document lines. A clean, fixed-size summary is far easier for a human to scan and far cheaper for an agent to read.

The detailed per-specification statistics that the verbose output carried are **not** lost: they are already rendered on the HTML index page (the requirements behind them, SRS-005 through SRS-008, are satisfied by the index, not by the console). So the console dump was largely redundant with the generated HTML.

# Decision

Introduce a dedicated `ConsoleReporter` module and replace the verbose per-document console dump with a single concise progress summary keyed by **pipeline phase**.

## ConsoleReporter

A new `lib/almirah/console_reporter.rb` defines a small stateless module:

- `count(label, number)` — prints an aligned line `"<label> ....... <number> ok"`.
- `result(label, value)` — prints an aligned line `"<label> ....... <value>"` (used for the output path).
- Both pad the label with dots to a fixed column (28) so the values line up.
- Colour is applied **only when `$stdout.tty?`** is true (bright green for counts, bright cyan for the result), so piped or captured output stays free of ANSI escape codes.

## Phase reporting in the pipeline

`project.rb` emits exactly one line per phase, at the end of that phase's method:

```
parsing specifications ..... 4 ok
parsing test protocols ..... 2 ok
parsing decisions .......... 14 ok
traceability matrices ...... 4 ok
coverage matrices .......... 1 ok
implementation matrices .... 1 ok
decision links ............. 5 ok
rendering HTML ............. <project>/build/index.html
```

`link_all_decisions` now tallies the number of decision→specification links it creates so it can report a meaningful count. A final `report_rendered` line is emitted at the end of both `specifications_and_protocols` and `specifications_and_results` (the `--run` variant), so the index path is always the last thing printed.

## Index path normalization

The index path printed by `report_rendered` was assembled by interpolating the raw project-directory argument directly into `"…/build/index.html"`. Because the argument is stored verbatim as the user typed it, the result depended on whether that argument carried a trailing slash: a literal separator before `build` doubled it for trailing-slash inputs (`./` → `.//build`, `/a/b/c/` → `/a/b/c//build`), while omitting the separator dropped it for no-trailing-slash inputs (`Almirah.Doc` → `Almirah.Docbuild`). No single literal handled both forms.

This is resolved in two places:

- `ProjectConfiguration#initialize` normalizes the argument once, at the single point of entry, via `File.expand_path`. Every downstream consumer (the render output paths, the `FileUtils` calls, the `project.yml` loader) inherits a clean, absolute, trailing-slash-free root. Repository paths declared in `project.yml` are resolved against the current working directory and are unaffected.
- `report_rendered` composes the path with `File.join` (separator-safe) and, when the resolved project directory equals the current working directory, displays it relative to the current directory as `./build/index.html`; otherwise it prints the absolute path. So any argument that points at the current directory — `./`, `.`, `./../<cwd-name>/`, or the matching absolute path with or without a trailing slash — yields `./build/index.html`, and any other directory yields an absolute path.

## Suppressing the verbose output

The five per-document `doc.to_console` invocations in the render methods (`render_all_specifications`, `render_all_source_files`, `render_index`, `render_decisions_overview`, `render_all_decisions`) are removed from the pipeline, and the stray `puts @specifications_path` debug line is removed from the `SourceFile` constructor. The `to_console` methods themselves are left defined on the document types; they are simply no longer called during a build.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | Done | 31-05-2026 | 01-06-2026 | New SRS items (SRS-079 onward) covering: a concise per-phase progress summary on standard output; the generated index path as the final progress line; ANSI colour only on an interactive terminal; index-path normalisation (SRS-082) — relative `./build/index.html` when the project directory is the current directory, absolute otherwise, free of duplicated or missing separators |
| Code | Done | 31-05-2026 | 01-06-2026 | Added `lib/almirah/console_reporter.rb`; added `ConsoleReporter.count` calls to `parse_all_specifications`, `parse_all_protocols`, `parse_decisions`, `link_all_specifications`, `link_all_protocols`, `link_all_decisions` (now tallying links), and `link_all_source_files`; added `report_rendered` to both pipeline entry points; removed the five `doc.to_console` calls from the render methods; removed the debug `puts @specifications_path` from `SourceFile`; normalised the project directory once via `File.expand_path` in `ProjectConfiguration#initialize`; rewrote `report_rendered` to build the path with `File.join` and print it relative (`./build/index.html`) when the project directory is the current directory, absolute otherwise |
| Tests | Done | 31-05-2026 | 01-06-2026 | End-to-end (`aruba`) assertions in `spec/e2e/console_output_spec.rb` that `almirah please` prints the phase summary lines (SRS-079) and the index path as the final line (SRS-080), that captured (non-TTY) output contains no ANSI escape codes (SRS-081), and that the index path is shown relative (`./build/index.html`) for current-directory inputs and absolute otherwise, with no duplicated or missing separators for trailing-/no-trailing-slash argument forms (SRS-082). Each example names the SRS item it covers |

# Out of Scope

- Removing the per-document `to_console` methods themselves. They remain defined on the document types and are simply no longer invoked by the build pipeline; deleting them is a separate clean-up.
- A `--verbose` / `--quiet` flag or configurable log levels. The concise summary is the single, default behaviour.
- A logging framework or structured (JSON) output. Plain aligned text matches the existing tool style.
- Progress bars, spinners, per-phase timing, or total-duration metrics.
- Changing the per-specification statistics on the HTML index (Number of Controlled Items, Up-/Down-links, Test Coverage). Those are unchanged and remain the canonical home for that data (SRS-005 through SRS-008).
- The format of broken-link / wrong-reference warnings produced during checking.
- Exit codes and error handling.

# Consequences

## Positive

- Output is fixed-size and scannable regardless of project size or the number of files in linked source repositories; the generated index path is always the last line.
- Cleaner, lower-volume logs for CI and for GenAI-agent runs — fewer tokens to capture and read, complementing the GenAI-native direction.
- TTY-gated colour means piped or redirected output is plain text with no escape-code noise.
- Removes a leftover debug statement that leaked an internal relative path once per source file.
- The printed index path is now correct and stable for every form of the project-directory argument (trailing slash or not, relative or absolute): no duplicated or missing separators, a tidy `./build/index.html` when run against the current directory, and an unambiguous absolute path otherwise. Normalising the directory once in `ProjectConfiguration` also makes every other path built from the project root consistent.

## Negative

- The per-specification statistics (controlled items, up-/down-links, coverage) no longer appear on the console. Mitigation: they are already rendered on the HTML index page (SRS-005 through SRS-008), so no information is actually lost — only console redundancy is removed.
- The progress counts are not yet specified in the SRS nor covered by tests (tracked above as To Do); until then the summary is unverified behaviour.
- Counting is performed as a side effect inside the parse/link methods, lightly coupling "do the work" with "report the count".

## Neutral

- The `to_console` methods remain in the codebase, unused by the pipeline.
- No change to the generated HTML, the build directory layout, or exit codes.
- Negligible performance impact (a handful of length lookups and `puts` calls per build).

# Alternatives Considered

- **Keep the verbose output and add a `--quiet` flag.** Rejected: concise-by-default is the better baseline for the common case, for CI, and for agents; a verbosity flag can be added later if a real need appears.
- **Adopt a logging framework or emit structured JSON.** Rejected: over-engineered for a one-shot build tool; plain aligned text matches the existing simple console style.
- **Leave the per-document `to_console` output as-is.** Rejected: its volume scales with project size and with linked-repository file counts, which is exactly what obscured the result.
- **Print per-document lines but drop only the statistics tables.** Rejected: still scales linearly with document count; a per-phase summary is constant-size and more informative at a glance.

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.4.0 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.1 |

# Affected Documents

| # | Proposed Text | Req-ID |
|---|---|---|
| 1 | While processing a project, the software shall emit a concise progress summary to standard output consisting of one line per processing phase, each line pairing a phase label with the number of items processed in that phase. | >[SRS-079] |
| 2 | The software shall print the path of the generated index page as the final line of the progress summary. | >[SRS-080] |
| 3 | The software shall apply ANSI colour to its progress output only when standard output is an interactive terminal, and shall emit uncoloured text otherwise. | >[SRS-081] |
| 4 | When composing the generated index path for the progress summary, the software shall resolve the project directory argument to a normalised path free of duplicated or missing path separators, regardless of the form of the argument. The path shall be shown relative to the current directory as "./build/index.html" when the project directory resolves to the current working directory, and as an absolute path otherwise. | >[SRS-082] |

# References

- SRS-005 through SRS-008 in [srs.md](./../../specifications/srs/srs.md) — the per-specification statistics that the removed verbose console output duplicated; they remain satisfied by the HTML index page

# Review Evidences

- Decision Record:
- Requirements:
- Code:
- Tests:
