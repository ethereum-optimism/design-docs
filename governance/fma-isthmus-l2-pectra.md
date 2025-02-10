
# Pectra Features on Isthmus

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Chain halt due to upgrade transaction failing](#fm1-chain-halt-due-to-upgrade-transaction-failing)
  - [FM2: Chain halt due to request hash mismatch](#fm2-chain-halt-due-to-request-hash-mismatch)
  - [FM3: EIP-7702 transactions cannot be included on the chain](#fm3-eip-7702-transactions-cannot-be-included-on-the-chain)
  - [FM4: BLS Precompiles could cause increased FP Program execution time](#fm4-bls-precompiles-could-cause-increased-fp-program-execution-time)
  - [FM5: BLS Precompiles could fail to execute in the FP program](#fm5-bls-precompiles-could-fail-to-execute-in-the-fp-program)
  - [FM6: Increased call data cost affects network upgrade transactions](#fm6-increased-call-data-cost-affects-network-upgrade-transactions)
  - [FM7: Early fork if batches containing EIP-7702 transactions could be posted before Pectra](#fm7-early-fork-if-batches-containing-eip-7702-transactions-could-be-posted-before-pectra)
  - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Italics are used to indicate things that need to be replaced._

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Julian Meyer                                       |
| Created at         | 2025-02-10                                         |
| Initial Reviewers  |                  |
| Need Approval From |                            |
| Status             | Draft |

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

This document covers implementation of Pectra features in the OP stack. This requires a hard fork due to new transaction types like EIP-7702 transactions. This does not cover generic hardfork related security concerns, which will be addressed in a separate document.

Included EIPs:

-   EIP-7702: Set code transaction
-   EIP-2537: BLS12-381 precompiles
-   EIP-2935: Block hashes contract predeploy
-   EIP-7251: Increase the MAX_EFFECTIVE_BALANCE
-   EIP-7623: Increase calldata cost
-   EIP-7002: Execution layer triggerable withdrawals
-   EIP-6110: Supply validator deposits on chain
-   EIP-7685: General purpose execution layer requests

## Failure Modes and Recovery Paths

### FM1: Chain halt due to upgrade transaction failing

- **Description:** The chain could halt if a network upgrade transaction can't be included in the upgrade block.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. end-to-end tests created for network upgrade transactions
  2. same upgrade path as previous predeploys
- **Detection:** Monitors should trigger an alert immediately
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing the network upgrade transaction and updates to `op-node`.

### FM2: Chain halt due to request hash mismatch

- **Description:** Requests hash was added to the block header. If it mismatches between geth and op-node, the chain could halt
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. End-to-end tests
  2. Simple validation rules - always must be an empty hash
- **Detection:** Monitors should trigger an alert immediately.
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing the network upgrade transaction and updates to `op-node`.

### FM3: EIP-7702 transactions cannot be included on the chain

- **Description:** SetCode (EIP-7702) transactions can't be included on chain.
- **Risk Assessment:** Low impact, low likelihood
- **Mitigations:** 
	1. End-to-end tests of set code transactions before and after the fork
- **Detection:** Sending set code transactions would fail; probably manual detection is most likely.
- **Recovery Path(s)**: This would require another update and fork.

### FM4: BLS Precompiles could cause increased FP Program execution time

- **Description:** If implemented in the FP program, BLS precompiles could greatly increase computation time.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program.
- **Detection:** op-program trace runner computation time will increase (possibly triggering an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute
BLS precompiles in sufficient time.

### FM5: BLS Precompiles could fail to execute in the FP program

- **Description:** If implemented in the FP program, BLS precompiles could fail to execute. This could cause certain blocks to be unprovable.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program. This means it should 
  have the same performance characteristics as existing crypto precompiles like `ecrecover`.
- **Detection:** op-program trace runner failure (possibly triggering an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute BLSprecompiles in sufficient time.

### FM6: Increased call data cost affects network upgrade transactions

- **Description:** Increased calldata cost could mean that network upgrade transactions fail.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Tested that network upgrade transactions don't fail.
- **Detection:** op-program trace runner failure (possibly triggering an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute BLSprecompiles in sufficient time.

### FM7: Early fork if batches containing EIP-7702 transactions could be posted before Pectra

- **Description:** If batches containing EIP-7702 transactions are posted and accepted early, the batcher could fork any nodes that upgrade early.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. End-to-end test
- **Detection:** Upgraded clients would fork
- **Recovery Path(s)**: Emergency patch


### Generic items we need to take into account:

<!-- See [generic hardfork failure modes](./fma-generic-hardfork.md) and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document. -->

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] FM6: double check that call data cost doesn't break upgrade transactions

## Audit Requirements

_Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning._

**No.** All of the code is well-tested and has very low likelihood for any failure modes with reasonably high impact.

<!-- ## Appendix

### Appendix A: This is a Placeholder Title

_Appendices must include any additional relevant info, processes, or documentation that is relevant for verifying and reproducing the above info. Examples:_

- _If you used certain tools, specify their versions or commit hashes._
- _If you followed some process/procedure, document the steps in that process or link to somewhere that process is defined._# [Project Name]: Failure Modes and Recovery Path Analysis -->
