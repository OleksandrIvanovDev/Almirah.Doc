% Almirah Framework Architecture

# Overview

This document defines the architecture of the Almirah framework as a set of processes (work instructions) and software components used for:

* Project Management
* Requirements Management
* Test Management
* Change Management

# Reference Documents

| Document ID | Document Title                         |
|---|---|
| SYS | [Almirah System Specification](sys.md) |

# Business Process View

Almirah framework is designed to support software development process that is a composition of the following high-level processes:

* Software release planning;
* Software design;
* Software implementation;
* Software testing;
* Final software release process.

![Business Process View](./img/003.svg)

## Software Release Planning

On the release planning stage a High-Level Requirements (an a process input) are transformed to the Work Breakdown Structure (process output).

![Software Release Planning Process](./img/004.svg)

The Work Breakdown Structure elements (WBS Items) are used to track planned activities through all the processes.

![WBS Items](./img/007.svg)

The transformation of High-Level Requirements to the Work Breakdown Structure requires the following software services:

* High-Level Requirements to be captured in a form of specification and stored with version control;
* WBS Items to be created with appropriate planning  attributes, such as: effort estimation, start date, end date, item type, sub-items.

## Software Design

Software design process transforms High-Level Requirement Specification stored with version control to one or several Detailed Specification(s).

Several examples of Detailed Specification are:

* Software Requirements Specification;
* Functional Requirements Specification;
* Non-functional Requirements Specification;
* Software Architecture Specification;
* System Architecture Specification;
* Software Detailed Design Specification;
* e.t.c.

![Software Design Process](./img/005.svg)

WBS Item(s) is(are) used to track the scope of transformation High-Level Requirements to Detailed Specifications. A WBS Item Tracking software service is required to instrument this process.

For example, WBS Item can cover the transformation of entire specification, one or several sections, or even a single High-Level Requirement paragraph.

All the Detailed Specifications need to be:

* Created;
* Stored with version control;
* Reviewed and approved;

with dedicated software services.

### Traceability

Almirah framework defines several traceability types for the Software Design Process:

* Specification to WBS Item (S2WI) traceability;
* WBS Item to Specification (WI2S) traceability;
* Specification to Specification (S2S) traceability;

Sections below describe the purpose of each traceability type in the Software Design Process.

#### S2WI Traceability

Specification to WBS Item traceability is used to identify the scope of WBS Item. It can be applied to the High-Level Requirement Specification transformation to a Detailed Specification.

![S2WI Traceability in Software Design Process. Option 1](./img/008.svg)

Or S2WI traceability can be applied to the transformation of one Detailed Specification to another.

![S2WI Traceability in Software Design Process. Option 2](./img/009.svg)

S2WI Traceability creation and maintenance requires the S2WI Traceability Management software service that is a composition of:

* S2WI Traceability Creation service;
* S2WI Traceability Review service;
* S2WI Traceability Approval service.

#### WI2S Traceability

WBS Item to Specification traceability is used to identify exact output of Software Design process obtained in scope of WBS Item. It can be applied to both: the High-Level Requirement Specification transformation to a Detailed Specification, and Detailed Specification to Detailed Specification transformations.

![WI2S Traceability in Software Design Process. Option 1](./img/010.svg)

![WI2S Traceability in Software Design Process. Option 2](./img/011.svg)

WI2S Traceability creation and maintenance requires the WI2S Traceability Management software service that is a composition of:

* WI2S Traceability Creation service;
* WI2S Traceability Review service;
* WI2S Traceability Approval service.

#### S2S Traceability

Specification to Specification traceability is used:

* To identify the logical connection between two specifications;
* To show how the high-level specification item was designed on the low-level;
* To verify the completeness of the Software Design Process.

S2S Traceability is applied to both: the High-Level Requirement Specification transformation to a Detailed Specification, and Detailed Specification to Detailed Specification transformations.

![S2S Traceability in Software Design Process. Option 1](./img/006.svg)

![S2S Traceability in Software Design Process. Option 2](./img/012.svg)

S2S Traceability creation and maintenance requires the S2S Traceability Management software service that is a composition of:

* S2S Traceability Creation service;
* S2S Traceability Review service;
* S2S Traceability Approval service.

## Software Implementation

Software Implementation process transforms Detailed Specifications stored with version control to the Source Code. The Source Code is created, stored with version control, reviewed and approved during the Software Implementation process.

![Software Implementation Process](./img/013.svg)

WBS Item(s) is(are) used to track the scope of transformation Detailed Specifications to the Source Code. A WBS Item Tracking software service is required to instrument this process.

For example, WBS Item can cover the transformation of entire specification, one or several sections, or even a single Detailed Specification paragraph.

The Source Code need to be:

* Created;
* Stored with version control;
* Reviewed and approved;

with dedicated software services.

### Traceability

Almirah framework defines several traceability types for the Software Implementation Process:

* Specification to WBS Item (S2WI) traceability;
* WBS Item to Code (WI2C) traceability;

Sections below describe the purpose of each traceability type in the Software Implementation Process.

#### S2WI Traceability

Specification to WBS Item traceability is used to identify the scope of WBS Item that will be used further for the transformation of a Detailed Specification to the Source Code.

![S2WI Traceability in Software Implementation Process. Option 2](./img/014.svg)

S2WI Traceability creation and maintenance in scope of Software Implementation Process requires the same S2WI Traceability Management software services used for S2WI Traceability creation and maintenance in Software Design Process. These services are the following:

* S2WI Traceability Creation service;
* S2WI Traceability Review service;
* S2WI Traceability Approval service.

#### WI2C Traceability

WBS Item to Code traceability is used to identify the exact output of the Software Implementation process obtained in the scope of WBS Item.

![WI2C Traceability in Software Implementation Process](./img/015.svg)

WI2C Traceability creation and maintenance requires the WI2C Traceability Management software service that is a composition of:

* WI2C Traceability Creation service;
* WI2C Traceability Review service;
* WI2C Traceability Approval service.

# Use Case View

## User Roles

![User Roles](./img/002.svg)

[ARCH-015] There are 4 user roles Almirah framework defines: Analyst, Developer, Tester, and Process Engineer.

## Requirement Management

![Requirement Management Use Cases](./img/001.svg)

### Create Specification

[ARCH-005] Specifications are created by Analyst as a plain text documents in Markdown format. Markdown set of links is extended by specific tags. >[SYS-005]

"[AAA-NNN]" tag at the beginning of paragraph indicates paragraph ID. This ID can be used for a reference from other specifications.

">[AAA-NNN]" tag at the end of the paragraph indicated the reference to a paragraph outside this specification.

### Store Specification

[ARCH-006] Created specifications are saved and stored in a source control tool for further review and updates. >[SYS-005]

### Load Specification

[ARCH-007] Created and stored specifications are loaded from a source control tool. >[SYS-006]

### Change Specification

[ARCH-008] Loaded from a source control tool specifications are changed in the Markdown format and stored in a source control tool for further review and updates. >[SYS-007]

### Delete Specification

[ARCH-009] Created and stored specifications can be removed from a source control tool with a next commit. >[SYS-008]

[ARCH-011] However, all the content and history of changes well be preserved in a source control tool.

### Review Specification

[ARCH-012] Created or updated specifications are reviewed with a code review tool as files in the Markdown format >[SYS-009]

>Note: The code review tool can be part of the source control (pull requests, merge requests, etc.) or can be established separately.

### View Change History

[ARCH-014] The history of specification changes is a history of Markdown file changes provided by a source control tool. >[SYS-010]

### Traceability Management

[ARCH-001] Traceability is established on the level of paragraphs. A paragraph from one specification may contain a reference to a paragraph from another specification. >[SYS-001]

#### Create Reference

[ARCH-002] The reference between paragraphs in two different specifications is created by adding an item ID from another specification at the end of the paragraph. >[SYS-002]

#### Remove Reference

[ARCH-003] The reference between paragraphs in two different specifications can be removed by deletion an item ID at the end of the paragraph. >[SYS-002]

#### Review Traceability

[ARCH-004] Information about traceability between different specifications is a result of running a custom script. This script parses all the specifications and extracts all the references. Based on this information the script generates HTML representation of each specification extended with external references. Traceability report in HTML format is also generated by this script. >[SYS-002]
