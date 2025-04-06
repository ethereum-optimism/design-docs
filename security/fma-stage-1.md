# [Project Name]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[Name of Failure Mode 1]](#name-of-failure-mode-1)
  - [[Name of Failure Mode 2]](#name-of-failure-mode-2)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)
  - [Appendix A: This is a Placeholder Title](#appendix-a-this-is-a-placeholder-title)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | JosepBove                                          |
| Created at         | 2025-04-06                                         |
| Initial Reviewers  |                                                    |
| Need Approval From |                                                    |
| Status             |  Draft                                             |

## Introduction

This document covers _[project name, high-level summary of the project, and scope of this analysis]._

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/202/)
- [Specs](https://github.com/ethereum-optimism/specs/pull/625)
- [Implementation](https://github.com/ethereum-optimism/optimism/pull/15174)

## Failure Modes and Recovery Paths

### FM1: [Name of Failure Mode 1]

- **Description:** _Details of the failure mode go here. What the causes and effects of this failure?_
- **Risk Assessment:** _Simple low/medium/high rating of impact (severity) + likelihood._
- **Mitigations:** _What mechanisms are in place, or what should we add, to:_
  1. _reduce the chance of this occurring?_
  2. _reduce the impact of this occurring?_
- **Detection:** _How do we detect if this occurs?_
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### FM2: [Name of Failure Mode 2]

- **Description:** _Details of the failure mode go here. What the causes and effects of this failure?_
- **Risk Assessment:** _Simple low/medium/high rating of impact (severity) + likelihood._
  **Mitigations:** _What mechanisms are in place, or what should we add, to:_
  1. _reduce the chance of this occurring?_
  2. _reduce the impact of this occurring?_
- **Detection:** _How do we detect if this occurs?_
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### Generic items we need to take into account:

See [generic hardfork failure modes](./fma-generic-hardfork.md) and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document.

- [ ] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)


## Audit Requirements

## Appendix
