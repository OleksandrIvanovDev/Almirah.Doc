# Almirah Software Requirements Specification

## Overview

This document defines software requirements applicable for Almirah software.

## Definitions

| Term | Definition |
|---|---|
| Item Id | Unique identifier for a paragraph in the specification document defined in the following form "[AAA-NNN]", where AAA - is any combination of letters and NNN - is the number. |
| Controlled Item | Paragraph in a specification document that is started with the Item Id. |
| External Item Id | Controoled Item Id from another specification that is also managed by Almira system. It is defined as ">[BBB-NNN]" symbols at the end of Controlled Item, where BBB - is any combination of leters different from AAA and NNN - is the number. |

## Requirements

[SRS-001] The software shall allow to create a Controlled Items. >[ARCH-005]

>Example: "[ITM-001] This is a controlled item"

[SRS-004] The software shall allow to create a non-controlled items.

>Example: "This is a non-controlled item"

[SRS-002] The software shall allow to create a reference from a Controlled Item to external Controlled Item. >[ARCH-001]

>Example: "[ITM-001] This is a controlled item with the reference to the external controlled item >[EXT-004]"

[SRS-003] The software shall indicate whether a Controlled Item is referenced in another specification via External Item Id >[ARCH-004]
