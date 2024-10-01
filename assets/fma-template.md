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
| Initial Reviewers | *Reviewer Name 1, Reviewer Name 2* |
| Need Approval From | *Security Reviewer Name* |
| Status | *Draft / In Review / Implementing Actions / Final* |

> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team. 
> - Maintain the “Status” property accordingly. An FMA document can have the following statuses:
>   - **Draft 📝:** Doc is created but not yet ready for review.
>   - **In Review 🔎:** Security is reviewing, and Engineering is iterating on the design. A checklist of action items will be created during this phase.
>   - **Implementing Actions 🛫:** Security has signed off on the content of the document, including the resulting action items. Engineering is responsible for implementing the the action items, and updating the checklist.
>   - **Final 👍:** Security will transition the status of the document to Final once all action items are completed.

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

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] *Action item 2 (Assignee: tag assignee)*
- [ ] *Action item 3 (Assignee: tag assignee)*

## Audit Requirements

*Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning.*

## Appendix

### Appendix A: This is a Placeholder Title

*Appendices must include any additional relevant info, processes, or documentation that is relevant for verifying and reproducing the above info. Examples:*

- *If you used certain tools, specify their versions or commit hashes.*
- *If you followed some process/procedure, document the steps in that process or link to somewhere that process is defined.*
