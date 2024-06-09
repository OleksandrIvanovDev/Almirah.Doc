% Requirements for External Components

# Overview

This document defines requirements for external components that could be used by Almirah framework to provide a complete set of functionalities.

# Reference Documents

| Document ID | Document Title |
|---|---|
| ARCH | [Almirah Framework Architecture](./../arch/arch.md) |

# Definitions

| Term | Definition |
|---|---|
| SCM | Source Control Management |

# Requirements

## Source Control Management System

[RFEC-001] The Source Control Management (SCM) system shall allow storing text and binary (image) files. >[ARCH-006]

[RFEC-002] The Source Control Management (SCM) system shall allow to store the reason or summary of changes for already stored files (commit message). >[ARCH-014]

[RFEC-003] The Source Control Management (SCM) system shall allow to read the history of the reasons or summaries of file changes (commits history). >[ARCH-014]

[RFEC-005] The Source Control Management (SCM) system shall allow loading preliminary stored text and binary files with no changes. >[ARCH-007]

[RFEC-006] The Source Control Management (SCM) system shall allow storing multiple versions of the same file (in a chronological order). >[ARCH-008]

[RFEC-007] The Source Control Management (SCM) system shall allow to delete a preliminary store file. >[ARCH-009]

[RFEC-008] Deletion of the file from SCM shall preserve all previously stored version of this file and its history for further access. >[ARCH-011]

## Code Review Tool

The code review tool could be part of the SCM or selected and installed separately.

[RFEC-004] The Code Review Tool shall allow view the difference between two versions of the text file. >[ARCH-014]

[RFEC-009] The Code Review Tool shall allow store review comment for particular file end its line. >[ARCH-012]