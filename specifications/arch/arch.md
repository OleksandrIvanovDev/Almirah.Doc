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

The Software Testing process can be split into two stages:

* Test Design Process, where Test Cases are created against specifications;
* Test Execution Process, where preliminary created Test Cases are executed on the Software Implementation artifact.

![Splitted Software Testing Process](./img/017.svg)

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

## Software Testing

### Test Design Process

The Software Testing process transforms Detailed Specifications stored with version control to the Test Cases in the (Test Design stage).

![Test Design Process](./img/018.svg)

WBS Item(s) is(are) used to track the scope of transformation Detailed Specifications to the Test Cases. A WBS Item Tracking software service is required to instrument this process.

WBS Item can cover a single or multiple specification transformation to a Test Case, one or several sections, or even a single Detailed Specification paragraph.

The Test Cases need to be:

* Created;
* Stored with version control;
* Reviewed and approved;

with dedicated software services.

### Test Execution Process

When any software artifact is ready for testing (Test Execution stage), these Test Cases are executed to verify that the software artifact implementation meets Detailed Specifications.

![Test Execution Process](./img/019.svg)

WBS Item(s) is(are) used to track the scope of Test Case Execution. A WBS Item Tracking software service is required to instrument this process. WBS Item can include a single or multiple Test Cases for execution.

With dedicated software services, preliminary created, stored, reviewed, and approved Test Cases need to be:

* Marked with pass/fail results;
* Stored with version control with these marks;
* Reviewed and approved;

### Traceability

Almirah framework defines several traceability types for the Software Testing Processes:

* Specification to WBS Item (S2WI) traceability;
* WBS Item to Test Cases (WI2T) traceability;
* Specification to Test Case (S2T) Traceability.

Sections below describe the purpose of each traceability type in the Software Testing Processes.

#### S2WI Traceability

In the Test Design Process, the Specification to WBS Item traceability is used to identify the scope of the WBS Item that will be used further to transform a Detailed Specification into Test Cases.

![S2WI Traceability in Test Design Process](./img/020.svg)

In the Test Execution Process, the Specification to WBS Item traceability defines the scope of Test Cases for testing particular Detailed Specification items.

![S2WI Traceability in Test Execution Process](./img/021.svg)

S2WI Traceability creation and maintenance require the same S2WI Traceability Management software services in both processes. These services are the following:

* S2WI Traceability Creation service;
* S2WI Traceability Review service;
* S2WI Traceability Approval service.

#### WI2T Traceability

WBS Item to Test Case traceability is used to identify the exact list of Test Cases either designed or executed in the scope of this WBS Item.

![WI2T Traceability in Test Design Process](./img/022.svg)

![WI2T Traceability in Test Execution Process](./img/023.svg)

WI2T Traceability creation and maintenance requires the WI2T Traceability Management software service that is a composition of:

* WI2T Traceability Creation service;
* WI2T Traceability Review service;
* WI2T Traceability Approval service.

#### S2T Traceability

Specification to Test Case traceability defines what specification part (section or paragraph) is covered (tested) with the particular test. This type of traceability is applicable for both: Test Design and Test Execution Processes.

![S2T Traceability in Test Design Process](./img/024.svg)

![S2T Traceability in Test Execution Process](./img/025.svg)

S2T Traceability creation and maintenance requires the S2T Traceability Management software service that is a composition of:

* S2T Traceability Creation service;
* S2T Traceability Review service;
* S2T Traceability Approval service.

# Application View

The complete list of software services identified to service all the Software Release Development Processes is the following:

1. S2S Traceability Management;
   * S2S Traceability Creation;
   * S2S Traceability Review;
   * S2S Traceability Approval;
1. S2T Traceability Management;
   * S2T Traceability Creation;
   * S2T Traceability Review;
   * S2WI Traceability Approval;
1. S2WI Traceability Management
   * S2WI Traceability Creation;
   * S2WI Traceability Review;
   * S2WI Traceability Approval;
1. Source Code Creation
1. Source Code Storage w/ Version Control
1. Source Code  Review and Approval
1. Specification Creation;
1. Specification Storage w/ Version Control;
1. Specification Review and Approval;
1. Test Case Creation;
1. Test Case Storage w/ Version Control;
1. Test Case Review and Approval;
1. Test Marking;
1. WBS Item Creation;
1. WBS Item Tracking;
1. WI2C Traceability Management
   * WI2C Traceability Creation;
   * WI2C Traceability Review;
   * WI2C Traceability Approval;
1. WI2S Traceability Management
   * WI2S Traceability Creation;
   * WI2S Traceability Review;
   * WI2S Traceability Approval;
1. WI2T Traceability Management
   * WI2T Traceability Creation;
   * WI2T Traceability Review;
   * WI2T Traceability Approval;

## WBS Software Services

The WBS Item Creation and WBS Item Tracking services are realized by the Task/Issue Tracking Software.

![WBS Software Services Realization](./img/026.svg)

Since there are different processes to be tracked with WBS Items (Software Design, Software Implementation, Test Design, Test Execution) the Task/Issue Tracking Software shall support the following item types:

* Document Task;
* Code Task;
* Test Design Task;
* Test Execution Task.

## Content Creation Services

Almirah framework defines markdown files as the primary format for creating Detailed Specifications and Test Cases. The Test Marking is defined as a Test Case update process in markdown format, which inserts pass/fail marks into the file.

A Markdown Files Editor realizes the following software services:

* Specification Creation service;
* Test Case Creation service;
* Test Marking service.

![Content Creation Software Services Realization](./img/027.svg)

The Source Code Creation is realized by a Source Code Editor. The Source Code Editor and Markdown Files Editor can be the same Integrated Development Environment if it supports both formats.

## Storage with Version Control

Since Specifications and Test Cases are created as markdown files, it is possible to store them in a Source Control in the same way as the Source Code.

![Storage w/ Version Control Software Services Realization](./img/028.svg)

It is recommended to maintain separate repositories for:

1. Source Code;
2. Specifications (High-Level Requirement Specification and Detailed Specifications) and Test Cases (both executed and not executed).

## Content Review and Approval Services

A Code Review Software is used to review and approve the content of:

* Detailed Specifications;
* Source Code;
* Test Cases;
* Executed Test Cases.

![Storage w/ Version Control Software Services Realization](./img/029.svg)

The Source Control may Realize these content review services as well.

## Traceability Services

### S2S Traceability

In Almirah framework traceability between specifications is implemented with a custom extension to the markdown syntax.

This extension consists of two tags:

* Paragraph ID tag - "[SPID-001]" placed at the start of the paragraph, where
  * SPID - is a current specification ID letters;
  * 001 - is a unique paragraph number in this specification (can be non-sequential);
* Reference to paragraph ID tag - ">[SPID-001]" placed at the end of the paragraph that refers to a paragraph in another specification;
  * SPID - is another(external) specification ID letter;
  * 001 - is a unique paragraph number in the external specification;

These Markdown extensions are added manually to the specification body with the Markdown Files Editor and processed by a custom Ruby script (Almirah Ruby gem). The processing includes:

1. Parsing all the specifications;
1. Identification of S2S links;
1. S2S link errors check:
   * against duplicated paragraph ID tags;
   * against links to non-existing paragraph ID tag; 
1. Rendering all the specifications to HTML files where the S2S links are replaced with hyperlinks.

Specifications converted to HTML with hyperlinks allows to review S2S links for correctness.

![S2S Traceability Software Services Realization](./img/030.svg)

When S2S links are reviewed for correctness with Code Review Software in raw and HTML format, specifications containing these links are approved in the Code Review Software.

### S2T Traceability

In Almirah framework traceability between Specifications and Test Cases is implemented with custom extension to the markdown syntax and several additional rules.

These rules are the following:

* Test Case markdown file name defines the Test Case ID;
* Test Case markdown file name consists of two parts combined with dash ("tc-001"):
  * several letters as a prefix (for example "tc");
  * sequential number of the test case (for example "001");
* Test Case steps are defined as rows in the markdown table, where:
  * The first column indicates the test step number (1, 2, 3, e.t.c);
  * The last column is reserved for the reference to a specification paragraph ID (reference to paragraph ID tag - ">[SPID-001]") if the step is intended to verify some statement from the specification (see [S2S Traceability](#s2s-traceability-1) section for details);
  * The column before the last is reserved for the test step result. The only "pass" and "fail" marks are defined for this purpose. All other values are ignored.

These Markdown extensions are added manually to the test case body with the Markdown Files Editor and processed by a custom Ruby script (Almirah Ruby gem). The processing includes:

1. Parsing all the specifications;
1. Parsing all the test cases;
1. Identification of S2T links;
1. S2T link errors check:
   * against duplicated test steps;
   * against links to non-existing paragraph ID tag;
1. Rendering all the specifications to HTML files where the S2T links are replaced with hyperlinks;
1. Rendering all the test cases to HTML files where the S2T links are replaced with hyperlinks;

Specifications and Test Cases converted to HTML with hyperlinks in both directions allow the review of S2T links for correctness.

![S2T Traceability Software Services Realization](./img/031.svg)

When S2T links are reviewed for correctness with Code Review Software in raw and HTML format, test cases containing these links are approved in the Code Review Software.

### S2WI Traceability

As it was stated in the [Business View](#s2wi-traceability) section, the Specification to WBS Item traceability is used to define the scope of the transformation of either:

* High-Level Requirements Specification to Detailed Specification;
* Or Detailed Specification to Detailed Specification.

The Task/Issue Tracking Software ultimately realizes this type of traceability.

![S2WI Traceability Realization](./img/032.svg)

If a WBS Item requires traceability to specification, it is done by mentioning the specification ID or specification paragraph ID in the Description field of the WBS Item.

S2WI Traceability Review and Approval services are realized via custom WBS Item State field values:

![S2WI Traceability Review and Approval Flow](./img/033.svg)

1. When WBS Item is created it has the Draft state;
2. After the scope description in the WBS Item, the author moves the item to the "Scope Review" state;
3. If the reviewer is OK with the scope defined (including specified paragraph IDs), they move the item to the "Scope Approved" state;
4. In the case scope requires correction, the reviewer will move the item back to the Draft state;

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
