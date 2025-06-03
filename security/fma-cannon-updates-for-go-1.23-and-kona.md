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

_Italics are used to indicate things that need to be replaced._

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Paul Dowman                                        |
| Created at         | 2025-05-02                                         |
| Initial Reviewers  | Meredith Baxter                                    |  
| Need Approval From | Matt Solomon                                       |  
| Status             | Implementing Actions                               |  

> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team.
> - Maintain the “Status” property accordingly. An FMA document can have the following statuses:
>   - **Draft 📝:** Doc is created but not yet ready for review.
>   - **In Review 🔎:** Security is reviewing, and Engineering is iterating on the design. A checklist of action items will be created during this phase.
>   - **Implementing Actions 🛫:** Security has signed off on the content of the document, including the resulting action items. Engineering is responsible for implementing the action items, and updating the checklist.
>   - **Final 👍:** Security will transition the status of the document to Final once all action items are completed.

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the project. The FMA review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers updates to Cannon (Solidity and Go versions) to support Go 1.23 and to support running Kona

Below are references for this project:

- [Go 1.23 PR](https://github.com/ethereum-optimism/optimism/pull/14692)
- [New instructions for Kona PR](https://github.com/ethereum-optimism/optimism/pull/15601)
- [Add feature toggling to MIPS VM contracts PR](https://github.com/ethereum-optimism/optimism/pull/15487)

## Failure Modes and Recovery Paths

**_Use one sub-header per failure mode, so the full set of failure modes is easily scannable from the table of contents._**

### FM1: Toggles are incorrectly deployed or implemented causing features to be incorrectly toggled off

- **Description:** A [feature toggle](https://github.com/ethereum-optimism/optimism/pull/15487) was added. The contract could be deployed with the wrong version.
- **Risk Assessment:** low
- **Mitigations:**
  1. The version number is checked in the constructor, and currently it's required to be 7 (the latest version) so we shouldn't be able to deploy MIPS64.sol with the wrong version.
  2. This logic is fairly simple, it's just a check against the version number to enable features, so it's easy to reason about and low risk of being implemented incorrectly.
- **Detection:** We have manually reviewed for this.
- **Recovery Path(s)**: This would require a contract upgrade.

### FM2: Stack depth-related refactoring with new dclo/dclz instructions introduced a bug

- **Description:** Arguments were consolidated into a struct to avoid "stack too deep" issues. 
- **Risk Assessment:** low
- **Mitigations:** 
  1. We have comprehensive differential testing on all VM instructions between go and solidity, which should catch any potential refactoring-related bugs. In this case, the solidity code was changed but the go code was unchanged, therefore we have confidence a bug was not introduced from the refactor.  
  2. This is a trivial refactoring
- **Detection:** We rely on our tests.
- **Recovery Path(s)**: It would require fixing the bug and upgrading the contract.

### FM3: New dclo/dclz instructions are incorrectly implemented

- **Description:** There are two new instructions, there could be a bug in the implementation. They aren't used by op-program, but would be used if we ever deployed Kona on Cannon.
- **Risk Assessment:** low
- **Mitigations:** 
  1. These instructions aren't emitted by the Go compiler, so behavior should not affect the VM when running op-program
  2. If we ever do deploy Kona on Cannon we will do more testing, including running it on mainnet data for weeks in VM Runner.
- **Detection:** The program would crash if it used those instructions and they were incorrectly implemented.
- **Recovery Path(s)**: It would require fixing the bug and upgrading the contract.

### FM4: Incomplete Go 1.23 support (missing syscalls)

- **Description:** It's possible that the Go 1.23 compiler uses additional syscalls that we haven't noticed and they aren't implemented.
- **Risk Assessment:** low
- **Mitigations:**
  1. We have been running `op-challenger-runner` on production data for several weeks with the new VM
  2. We used `vm-compat`, a tool that runs in CI and detects new syscalls referenced in the op-program binary
- **Detection:**: we will continue to watch `op-challenger-runner` and will be alerted if any mainnet blocks fail.
- **Recovery Path(s)**: It would require fixing the bug and upgrading the contract.

### FM5: eventfd or mprotect noop insufficient for Go 1.23 suppport

- **Description:** the eventfd and mprotect syscalls were implemented as a noop, because it was determined that it won't be used by op-program even though there is a reference to it in the binary.
- **Risk Assessment:** medium
- **Mitigations:** 
  1. We have been running op-challenger-runner on production data for several weeks with the new VM
- **Detection:** We rely on our tests.
- **Recovery Path(s)**: It would require fixing the bug and upgrading the contract.

### Generic items we need to take into account:

See [generic hardfork failure modes](./fma-generic-hardfork.md) and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document.

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)

## Audit Requirements

_Given the failure modes and action items, will this project require an audit? See [FMAs in the SDLC](https://github.com/ethereum-optimism/pm/blob/main/src/fmas.md#determine-audit-requirements) for a reference decision making framework. Please explain your reasoning._
