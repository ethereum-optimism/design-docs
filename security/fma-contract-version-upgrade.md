# [Project Name]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [Solidity Bugs Fixed](#solidity-bugs-fixed)
    - [Missing side effect on `.selector` access bug (Bug fixed after 0.8.15)](#missing-side-effect-on-selector-access-bug-bug-fixed-after-0815)
    - [Storage write removal before conditional termination (Bug fixed after 0.8.15)](#storage-write-removal-before-conditional-termination-bug-fixed-after-0815)
    - [Change in Default EVM version between different Solidity versions](#change-in-default-evm-version-between-different-solidity-versions)
  - [Other Failure Modes](#other-failure-modes)
    - [Chosen solidity version is too recent and might contain unknown bugs](#chosen-solidity-version-is-too-recent-and-might-contain-unknown-bugs)
    - [Stack too deep errors during compilation](#stack-too-deep-errors-during-compilation)
    - [Contracts exceeding code size when compiled](#contracts-exceeding-code-size-when-compiled)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |               |
| ------------------- | ------------- |
| Author              | Michael Amadi |
| Created at          | 2024-08-20    |
| Needs Approval From | Matt Solomon  |
| Other Reviewers     | Mark Tyneway  |
| Status              | Draft         |

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
> - The ultimate goal of this document is to identify action items to improve the security of the project. The FMA review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers solidity version upgrade of all OP Stack smart contracts from 0.8.15 code to a more recent version with support for transient storage and mcopy opcodes (i.e versions >=0.8.24).

Below are references for this project:

- [Issue 11527: Smart Contracts: Update all 0.8.15 code to a more recent version](https://github.com/ethereum-optimism/optimism/issues/11527).

## Failure Modes and Recovery Paths

### Solidity Bugs Fixed

This section lists solidity bugs that were fixed as failure modes and important changes to default values used for compilation and explains why these might be a risk for us. This set of bug fixes and changes was chosen because:

- Bug fixes: These are bugs that are only introduced when code is written a specific way, though meant to be correct but the compiler generates incorrect bytecode that can include a vulnerability.
- Changes to default compilation settings: These are changes that if not put into consideration when writing and compiling code can lead to generating non-optimized, non-compiling and/or non-executable code.

#### Missing side effect on `.selector` access bug (Bug fixed after 0.8.15)

- **Description:**
  - Introduced: v0.6.2
  - Fixed: v0.8.21
  - Explanation: The legacy compiler assumes that conditionals used to determine which selector is used/removed have no side effects. So if the result of the conditional is known at compile time the legacy compiler will hardcode the corresponding selector (since selectors are known at compile time). This bug does no exist when the conditional is used with other types of constants.
- **Risk Assessment:**
  - Severity: High (if present)
  - Likelihood: Low (and mostly present in anti-patterns like using conditionals that have side effects)
- **Mitigations:** Using a solidity version above 0.8.20, if lower versions must be used, conditionals should have no side effects.
- **Detection:** Confidence on the absence of this bug can be stronger with robust unit tests that check that all side effects of a call happens and correctly.
- **Recovery Path(s)**: Redeployment and upgrade of the affected contracts

#### Storage write removal before conditional termination (Bug fixed after 0.8.15)

- **Description:**

  - Introduced: v0.8.13
  - Fixed: v0.8.17
  - Explanation: Incorrect assumption that a storage write is redundant so the optimizer removes it. But this assumption can be wrong when the developer intended otherwise. A storage write is considered redundant when the storage slot is written to twice within an execution context without that slot being read before the second write or execution unconditionally terminates before the slot is read again.

    According to the solidity blog, this bug can only occur if a contract contains pattern:

    1. A storage write. Note that the write may still be removed even if it happens only conditionally or within a call to a function that ends up being inlined.
    2. A call to a function that conditionally terminates using inline assembly as described above, but also has a different code path that returns to the caller.
    3. Any continuing control flow path does one of the following:
       - It overwrites the storage write in (1).
       - It reverts.

- **Risk Assessment:**
  - Severity: High (if present)
  - Likelihood: Low
- **Mitigations:** Use a solidity version above 0.8.17.
- **Detection:** If a solidity contract (using solidity version earlier than 0.8.17) follows the pattern described above, its potentially vulnerable to this.
- **Recovery Path(s)**: Redeployment and upgrade of the affected contracts

#### Change in Default EVM version between different Solidity versions

- **Description:** Version 0.8.25 sets the default evm version to cancun
- **Risk Assessment:**
  - Severity: Low/Medium/High
  - Likelihood: Medium
- **Mitigations:** As at 0.8.15, the default EVM version is 'London', if no explicit version override is declared, 'Cancun' will be used rather which if not intended can lead to code that either does not deploy or reverts un-ideally at runtime even though non-fork tests might have passed. A mitigation is explicitly declaring the intended EVM version to use for compilation regardless of if its the same as the default the compiler uses. That way, a new compiler version that changes the default EVM version does not change the one that is intended to be used. In this scenario however, this is not possible because the intended EVM version is the same as the compiler's EVM version default.
- **Detection:** Running unit tests as fork test.
- **Recovery Path(s)**: Redeployment and upgrade of the affected contracts

### Other Failure Modes

#### Chosen solidity version is too recent and might contain unknown bugs

- **Description:** Choosing a very recent solidity version which hasn't stood a relatively long enough test of time can pose the risk of having an unknown bug introduced into the OP Stack smart contracts.
- **Risk Assessment:**
  - Severity: High
  - Likelihood: Low/Medium
- **Mitigations:** Might be safer to use a solidity version 1 or 2 releases earlier than the latest.
- **Detection:** -
- **Recovery Path(s)**: Redeployment and upgrade of the affected contracts

#### Stack too deep errors during compilation

- **Description:** Current OP Stack contracts only compile with the legacy pipeline with the optimizer on. Compiling with the optimizer off or using the via-ir pipeline result in stack too deep errors. When upgrading to a new solidity version, we have to make sure that successful compilation using the current build settings is possible.
- **Risk Assessment:**
  - Severity: -
  - Likelihood: -
- **Mitigations:** -
- **Detection:** During compilation
- **Recovery Path(s)**: Code modification (local variable scoping, local variables in memory etc), build settings modification or/and usage of a different compiler version.

#### Contracts exceeding code size when compiled

- **Description:** There are a few OP Stack contracts that are close to the contract code size limit. We have to ensure that the chosen compiler version to upgrade to compiles all OP Stack contracts successfully and without any exceeding the contract code size limit.
- **Risk Assessment:**
  - Severity: -
  - Likelihood: -
- **Mitigations:** -
- **Detection:** During compilation
- **Recovery Path(s)**: Code modification (runtime code size optimization), build settings modification or/and usage of a different compiler version.

## Audit Requirements

_Will this project require an audit according to the guidance in [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864)? Please explain your reasoning._

This project would not require an audit.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] Contracts compile successfully with upgraded solidity version and without stack too deep error(s)
- [ ] Contracts compile successfully with upgraded solidity version and without any contract intended to be deployed exceeding the contract code size limit (24576 bytes)

## Appendix

- [Solidity 0.8.25 Release Announcement](https://soliditylang.org/blog/2024/03/14/solidity-0.8.25-release-announcement/)
- [Solidity 0.8.24 Release Announcement](https://soliditylang.org/blog/2024/01/26/solidity-0.8.24-release-announcement/)
- [Solidity 0.8.23 Release Announcement](https://soliditylang.org/blog/2023/11/08/solidity-0.8.23-release-announcement/)
- [Solidity 0.8.22 Release Announcement](https://soliditylang.org/blog/2023/10/25/solidity-0.8.22-release-announcement/)
- [Solidity 0.8.21 Release Announcement](https://soliditylang.org/blog/2023/07/19/solidity-0.8.21-release-announcement/)
- [Solidity 0.8.20 Release Announcement](https://soliditylang.org/blog/2023/05/10/solidity-0.8.20-release-announcement/)
- [Solidity 0.8.19 Release Announcement](https://soliditylang.org/blog/2023/02/22/solidity-0.8.19-release-announcement/)
- [Solidity 0.8.18 Release Announcement](https://soliditylang.org/blog/2023/02/01/solidity-0.8.18-release-announcement/)
- [Solidity 0.8.17 Release Announcement](https://soliditylang.org/blog/2022/09/08/solidity-0.8.17-release-announcement/)
- [Solidity 0.8.16 Release Announcement](https://soliditylang.org/blog/2022/08/08/solidity-0.8.16-release-announcement/)
- [Head Overflow Bug in Calldata Tuple ABI-Reencoding](https://soliditylang.org/blog/2022/08/08/calldata-tuple-reencoding-head-overflow-bug/)
- [Storage Write Removal Bug On Conditional Early Termination](https://soliditylang.org/blog/2022/09/08/storage-write-removal-before-conditional-termination/)
- [Bug in Legacy Code Generation When Accessing the .selector Member on Expressions with Side Effects](https://soliditylang.org/blog/2023/07/19/missing-side-effects-on-selector-access-bug/)
