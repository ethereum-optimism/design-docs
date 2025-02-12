
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

<!-- _Italics are used to indicate things that need to be replaced._ -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Julian Meyer                                       |
| Created at         | 2025-02-10                                         |
| Initial Reviewers  | refcell                                            |
| Need Approval From |                                                    |
| Status             | In Review 🔎                                       |
<!-- 
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
> - The ultimate goal of this document is to identify action items to improve the security of the project. The FMA review process can be accelerated by proactively identifying action items during the writing process. -->

## Introduction

This document covers implementation of Pectra features in the OP stack. This requires a hard fork due to new transaction types like EIP-7702 transactions. This does not cover generic hardfork related security concerns, which will be addressed in a separate document.

Below is a list of EIPs that are enabled with Pectra, and a quick description of spec changes needed to support the EIP.

- EIP-7702: Set code transaction
  - [Added authorization list to span batch format](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/isthmus/derivation.md#span-batch-updates)
- EIP-2537: BLS12-381 precompiles
  - [Added input limits and specify acceleration for new precompiles](https://github.com/ethereum-optimism/specs/blob/a795bb1f8c34410c47f05d4c3614482361db64b1/specs/protocol/isthmus/exec-engine.md#evm-changes)
- EIP-2935: Block hashes contract predeploy
  - [Added network upgrade transaction](https://github.com/ethereum-optimism/specs/blob/a795bb1f8c34410c47f05d4c3614482361db64b1/specs/protocol/isthmus/derivation.md#eip-2935-contract-deployment)
- EIP-7251: Increase the MAX_EFFECTIVE_BALANCE
  - Only affects execution clients
- EIP-7623: Increase calldata cost
  - Only affects execution clients
- EIP-7002: Execution layer triggerable withdrawals
  - Not applicable to OP
  - Requests sent to this contract are ignored, and the contract is not deployed by default
- EIP-6110: Supply validator deposits on chain
  - Not applicable to OP
  - Requests sent to this contract are ignored, and the contract is not deployed by default
- EIP-7685: General purpose execution layer requests
  - No requests are currently applicable to OP, so the [execution requests array is always empty in Isthmus](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/exec-engine.md#engine_newpayloadv4).

## Failure Modes and Recovery Paths

### FM1: Chain halt due to upgrade transaction failing

- **Description:** The chain could halt if a network upgrade transaction can't be included in the upgrade block.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. end-to-end tests created for network upgrade transactions
      - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
  2. same upgrade path as previous predeploys
      - example: [EIP-4788 deployment transaction](https://github.com/ethereum-optimism/optimism/blob/b6b74290b5502f45daae57946f969327cdb2d383/op-node/rollup/derive/ecotone_upgrade_transactions.go#L130)
    
- **Detection:** Alert for L2 Unsafe liveness should be triggered
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing the network upgrade transaction in rollup node client software (like `op-node`).

### FM2: Chain halt due to request hash mismatch

- **Description:** Requests hash was added to the block header. If it mismatches between the EL client and the rollup node, the chain could halt.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. End-to-end test ensuring no requests are included in blocks
      - https://github.com/ethereum-optimism/optimism/pull/14253
  2. Simplicity of validation rules - always must be an empty hash
- **Detection:** Alert for L2 Unsafe liveness should be triggered
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing requests hash logic in respective clients (e.g. `op-node`, `op-geth`, `op-reth`, ...)

### FM3: EIP-7702 transactions cannot be included on the chain

- **Description:** A client implementation could have a bug that disallows SetCode (EIP-7702) transactions from being included on chain. This could cause an unintended fork by the affected client.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:** 
	1. End-to-end tests of set code transactions before and after the fork
        - https://github.com/ethereum-optimism/optimism/pull/14197
        - https://github.com/ethereum-optimism/optimism/pull/14288
- **Detection:** Sending set code transactions would fail - manual detection is most likely.
- **Recovery Path(s)**: This would require an emergency client update of the implementation with the bug (e.g. `op-reth`, `op-geth`, ...).

### FM4: BLS Precompiles could cause increased FP Program execution time

- **Description:** If implemented in the FP program, BLS precompiles could greatly increase computation time.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program.
        - [Specs](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/isthmus/exec-engine.md#evm-changes)
- **Detection:** op-program trace runner computation time would increase significantly (should trigger an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute
BLS precompiles in sufficient time.

### FM5: BLS Precompiles could fail to execute in the FP program

- **Description:** If implemented in the FP program, BLS precompiles could fail to execute. This could cause certain blocks to be unprovable.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program. This means it should 
  have the same performance characteristics as existing crypto precompiles like `ecrecover`.
        - [Specs](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/isthmus/exec-engine.md#evm-changes)
- **Detection:** op-program trace runner failure (possibly triggering an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute BLS precompiles in sufficient time.

### FM6: Increased call data cost affects network upgrade transactions

- **Description:** Increased calldata cost could mean that network upgrade transactions fail.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Tested that network upgrade transactions don't fail.
        - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
- **Detection:** Manual detection is likely after inspecting the fork block.
- **Recovery Path(s)**: We would have to release an emergency rollup node (e.g. `op-node`) update with fixed network upgrade trnasactions.

### FM7: Early fork if batches containing EIP-7702 transactions could be posted before Pectra

- **Description:** If batches containing EIP-7702 transactions are posted and accepted early, the batcher could fork any nodes that upgrade early. A malicious batcher could cause this to happen if we don't have a check to ensure type 4 (SetCode) txs can only appear in a batch after Isthmus is active.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
  1. Spec disallows type 4 transactions before Isthmus
	1. End-to-end test
        - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
- **Detection:** Upgraded clients would fork causing an alert on L2 unsafe liveness.
- **Recovery Path(s)**: Emergency patch to rollup node clients (like `op-node`).


### Generic items we need to take into account:

<!-- See [generic hardfork failure modes](./fma-generic-hardfork.md) and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document. -->

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] FM6: double check that call data cost doesn't break upgrade transactions - https://github.com/ethereum-optimism/specs/issues/576

## Audit Requirements
<!-- 
_Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning._ -->

**No.**

- The security concerns in this project affect liveness more than safety.
  - None of these failures would cause a loss of user funds.
  - Some could cause a chain halt or an unintended fork for some clients (if unmitigated).
  - Mitigations reduce risk of unintended fork or chain halt to near zero.
- The security concerns are more a reputational threat rather than an existential threat.

This change should be secure through extensive testing and real world usage rather than through extreme caution and auditing.



<!-- ## Appendix

### Appendix A: This is a Placeholder Title

_Appendices must include any additional relevant info, processes, or documentation that is relevant for verifying and reproducing the above info. Examples:_

- _If you used certain tools, specify their versions or commit hashes._
- _If you followed some process/procedure, document the steps in that process or link to somewhere that process is defined._# [Project Name]: Failure Modes and Recovery Path Analysis -->
