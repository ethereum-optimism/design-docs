# [Project Name]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [Missing side effect on `.selector` access bug (Bug fixed after 0.8.15)](#missing-side-effect-on-selector-access-bug-bug-fixed-after-0815)
  - [Storage write removal before conditional termination (Bug fixed after 0.8.15)](#storage-write-removal-before-conditional-termination-bug-fixed-after-0815)
  - [Deprecation of early EVM versions](#deprecation-of-early-evm-versions)
  - [Set default EVM version to cancun](#set-default-evm-version-to-cancun)
  - [Deprecation of block.difficulty](#deprecation-of-blockdifficulty)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Italics are used to indicate things that need to be replaced._

|                     |                                    |
| ------------------- | ---------------------------------- |
| Author              | Michael Amadi                      |
| Created at          | 2024-08-20                         |
| Needs Approval From | _Security Reviewer Name_           |
| Other Reviewers     | _Reviewer Name 1, Reviewer Name 2_ |
| Status              | Draft                              |

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

This document covers solidity version upgrade of all OP Stack smart contracts from 0.8.15 code to 0.8.25.

Below are references for this project:

- [Issue 11527: Smart Contracts: Update all 0.8.15 code to 0.8.25](https://github.com/ethereum-optimism/optimism/issues/11527).

## Failure Modes and Recovery Paths

### Missing side effect on `.selector` access bug (Bug fixed after 0.8.15)

- **Description:**
  - Introduced: v0.6.2
  - Fixed: v0.8.21
  - Explanation: The legacy compiler assumes that conditionals used to determine which selector is used/removed have no side effects. So if the result of the conditional is known at compile time the legacy compiler will hardcode the corresponding selector (since selectors are known at compile time). This bug does no exist when the conditional is used with other types of constants.
- **Risk Assessment:**
  - Severity: High (if present)
  - Likelihood: Low (and mostly present in anti-patterns like using conditionals that have side effects)
- **Mitigations:** Using a solidity version above 0.8.20, if lower versions must be used, conditionals should have no side effects.
- **Detection:** On the rare scenario (when using solidity version earlier than 0.8.21) where a conditional results in a constant boolean, has a side effect and accesses `.selector` method on function types, that conditional is prone to this bug.
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### Storage write removal before conditional termination (Bug fixed after 0.8.15)

- Introduced: v0.8.13
- Fixed: v0.8.17
- Explanation: Incorrect assumption that a storage write is redundant so the optimizer removes it. But this assumption can be wrong when the developer intended otherwise. A storage write is considered redundant when the storage slot is written to twice within an execution context without that slot being read before the second write or execution unconditionally terminates before the slot is read again.
  This bug can only occur if a contract contains pattern:
  - A storage write. Note that the write may still be removed even if it happens only conditionally or within a call to a function that ends up being inlined.
  - A call to a function that conditionally terminates using inline assembly as described above, but also has a different code path that returns to the caller.
  - Any continuing control flow path does one of the following:
    - It overwrites the storage write in (1).
    - It reverts.
- **Risk Assessment:**
  - Severity: High (if present)
  - Likelihood: Low
- **Mitigations:** Use a solidity version above 0.8.17
- **Detection:** If a solidity contract (using solidity version earlier than 0.8.17) follows the pattern described above, its potentially vulnerable to this.
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### Deprecation of early EVM versions

- **Description:** Version 0.8.22 of solidity deprecated EVM versions "homestead", "tangerineWhistle", "spuriousDragon" and "byzantium". This would mean that compiling with these versions but on solidity 0.8.25 would not be possible
- **Risk Assessment:** _Simple low/medium/high rating of impact (severity) + likelihood._
- **Mitigations:** _What mitigations are in place, or what should we add, to reduce the chance of this occurring?_
- **Detection:** Compilation fails.
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### Set default EVM version to cancun

- **Description:** Version 0.8.25 sets the default evm version to cancun
- **Risk Assessment:** _Simple low/medium/high rating of impact (severity) + likelihood._
- **Mitigations:** As at 0.8.15, the default EVM version is 'London', if no explicit version override is declared, 'Cancun' will be used rather. A mitigation is explicitly declaring the intended EVM version to use for compilation regardless of if its the same as the default the compiler uses. That way, a new compiler version that changes the default EVM version does not change the one that is intended to be used. In this scenario, this is not possible because the intended EVM version is the same as the compiler versions default.
- **Detection:** _How do we detect if this occurs?_
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

### Deprecation of block.difficulty

- **Description:** Version 0.8.18 deprecated the use of block.difficulty.
- **Risk Assessment:** No instance of this in current contract.
- **Mitigations:** _What mitigations are in place, or what should we add, to reduce the chance of this occurring?_
- **Detection:** Compilation fails.
- **Recovery Path(s)**: _How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?_

## Audit Requirements

_Will this project require an audit according to the guidance in [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864)? Please explain your reasoning._

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)

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
