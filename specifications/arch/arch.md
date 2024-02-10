# Almirah Framework Architecture

## Overview

This document defines the architecture of the Almirah framework as a system as a set of processes (work instructions) and software components used for:

* Project Management
* Requirements Management
* Test Management
* Change Management

## Use Case View

### User Roles

![User Roles](./img/002.svg)

[ARCH-015] There are 4 user roles Almirah framework defines: Analyst, Developer, Tester, and Process Engineer.

### Requirement Management

![Requirement Management Use Cases](./img/001.svg)

#### Create Specification

[ARCH-005] Specifications are created by Analys as a plain text documents in Markdown format. Markdown set of links is extended by specific tags. >[SYS-005]

"[AAA-NNN]" tag at the beggining of paragraph indicates paragraph id. This id can be used for a reference from other specificaions.

">[AAA-NNN]" tag at the end of the paragraph indicated the reference to a paragraph outside of this specification.

#### Store Specification

[ARCH-006] Created specifications are saved and stored in a source control tool for further review and updates. >[SYS-005]

#### Load Specification

[ARCH-007] Created and stored specifications are loaded from a source control tool. >[SYS-006]

#### Change Specification

[ARCH-008] Loaded from a source control tool specifications are changed in the Markdown format and stored in a source control tool for further review and updates. >[SYS-007]

#### Delete Specification

[ARCH-009] Created and stored specifications can be removed from a source control tool with a next commit. >[SYS-008]

[ARCH-011] However, all the content and history of changes well be preserved in a source control tool.

#### Review Specification

[ARCH-012] Created or updated specifications are reviewed with a code review tool as a files in the Markdown format >[SYS-009]

[ARCH-013] This code review tool can be part of the source control (pull requests, merge requests, etc.) or can be established separately.

#### View Change History

[ARCH-014] The history of specification changes is a history of Markdown file changes provided by a source control tool. >[SYS-010]

#### Traceability Management

[ARCH-001] Traceability is established on the level of paragraphs. A paragraph from one specifaction may contain a reference to a paragraph from another specification. >[SYS-001]

##### Create Rreference

[ARCH-002] The reference between paragraphs in two different specifications is created by adding an item id from another specificaiton at the end of the paragraph. >[SYS-002]

##### Remove Rreference

[ARCH-003] The reference between paragraphs in two different specifications can be removed by deletion an item id at the end of the paragraph. >[SYS-002]

##### Review Traceability

[ARCH-004] Information about traceability between different specifications is a result of running a custom scrip. This script parce all the specifications and extracts all the references. Based on this infomration the script generates html representation of each specification extended with external references. Traceability report in html format is also gerenated by this script. >[SYS-002]
