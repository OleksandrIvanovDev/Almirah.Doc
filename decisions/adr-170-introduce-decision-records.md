---
title: "ADR-170: Introduce Decision Records"
---

# Status

| Date | Status |
|---|---|
| 14-05-2026 | Proposed |
| 14-05-2026 | Accepted |
| 15-05-2026 | In-Progress |

# Context

When using GenAI developer tools like Claude or Copilot, it was observed that the Almirah framework is well-suited to software development with GenAI assistants. It allows for keeping all project documentation in Markdown format, which is native to GenAI tools. So, the requirement/document/test management aspects of the Almirah framework work well and don't need to be changed.

The only project management aspect of the Almirah framework, initially designed to use issue-tracking software (Redmine, for example), requires a lot of tokens for agent interactions when information needs to be extracted.

# Decision

It was decided to introduce an additional entity to the Almirah Ruby gem, responsible for maintaining the required information used in the "decisions" folder, along with the "specifications" and "tests" that already exist. Decisions will be stored in a form of Markdown files with sequential numbers and a special naming convention. It is similar to the Architecture Decision Records (ADR) widely used by software developers.

The key point here is that these records are not tasks. These are decisions that were made and led to the implementation, testing, or other activities. Even a software defect is a decision record because it requires an investigation and a decision at the end about whether it needs to be fixed.

The Almirah Ruby script shall processs decision records, link them to the documents updated per this decision record, and provide seprate HTML pages for each decision record and the summary page.

# Scope

| Item | Status | Start Date | Target Date | Description |
|---|---|---|---|---|
| Requirements | In-progress | 12-05-2026 | 20-05-2026 | SRS specification needs to be updated to reflect basic operations with decision records such as: store decision records, show decision records, show decision records overview |
| Code | In-progress | 13-05-2026 | 24-05-2026 | Decision records to be processed by Almirah Ruby gem in scope of defined in the Requirements |
| Tests | In-progress | 13-05-2026 | 24-05-2026 | End-to-end tests shall be implemented to cover newly created Requirements |

# Out of Scope

To make this ADR feasible the links between decision records and speciifications (and protocols as well) excluded from the scope of implementation.

# Consequences

## Positive

- All the decisions and their context is placed near project documentation and available to GenAI tools
- No additional tools required (no issue tracker)

## Negative

- Project management with no task/issue tracker requires time and effort investments to learning
- The is a risk of decision record ID duplication
- The toolset is not native for non-technical people 

## Neutral

- Starting from this ADR further Almirah framework develompent will be tracked in this repository via ADRs
- There will be no new tasks in the Redmine
- The latest Redmine task number is #169, so this ARD should be #170

# Alternatives Considered

No alternatives were considered

# Software Versions

| Software Version Category | Software Version ID |
|---|---|
| Latest Released Version | 0.3.1 |
| Issue Found in Version | n/a |
| Target Release Version | 0.4.0 |

# References

TBD