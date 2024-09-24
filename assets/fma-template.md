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

*Italics are used to indicate things that need to be replaced.*

| | |
|--------|--------------|
| Author | *Author Name* |
| Created at | *YYYY-MM-DD* |
| Needs Approval From | *Security Reviewer Name* |
| Other Reviewers | *Reviewer Name 1, Reviewer Name 2* |
| Status | *Draft / In Review / Implementing Actions / Final* |

> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team.
> - Maintain the “Status” property accordingly.

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the  project. The FMA review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers *[project name, high-level summary of the project, and scope of this analysis].*

Below are references for this project:

- *Link 1, e.g. project charter or design doc*
- *Link 2, etc.*

## Failure Modes and Recovery Paths

***Use one sub-header per failure mode, so the full set of failure modes is easily scannable from the table of contents.***

### [Name of Failure Mode 1]

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: *How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?*

### [Name of Failure Mode 2]

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: *How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?*

### Generic items we need to take into account:
See [./fma-generic-hardfork.md](./fma-generic-hardfork.md). 

- [ ] Check this box to confirm that these items have been considered and updated if necessary.


## Audit Requirements

*Will this project require an audit according to the guidance in [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864)? Please explain your reasoning.*

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] *Action item 2 (Assignee: tag assignee)*
- [ ] *Action item 3 (Assignee: tag assignee)*

## Appendix

### Appendix A: This is a Placeholder Title

*Appendices must include any additional relevant info, processes, or documentation that is relevant for verifying and reproducing the above info. Examples:*

- *If you used certain tools, specify their versions or commit hashes.*
- *If you followed some process/procedure, document the steps in that process or link to somewhere that process is defined.*
