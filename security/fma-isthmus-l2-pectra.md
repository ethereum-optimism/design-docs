
# Pectra Features on Isthmus

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Chain halt due to L1 block deployment upgrade transaction failing](#fm1-chain-halt-due-to-l1-block-deployment-upgrade-transaction-failing)
  - [FM2: Chain halt or fork due to request hash mismatch](#fm2-chain-halt-or-fork-due-to-request-hash-mismatch)
  - [FM3: Increased call data cost affects network upgrade transactions](#fm3-increased-call-data-cost-affects-network-upgrade-transactions)
  - [FM4: EIP-7702 transactions cannot be included on the chain](#fm4-eip-7702-transactions-cannot-be-included-on-the-chain)
  - [FM5: BLS Precompiles could cause increased FP Program execution time](#fm5-bls-precompiles-could-cause-increased-fp-program-execution-time)
  - [FM6: BLS Precompiles could fail to execute in the FP program](#fm6-bls-precompiles-could-fail-to-execute-in-the-fp-program)
  - [FM7: Early fork if batches containing EIP-7702 transactions could be posted before Pectra](#fm7-early-fork-if-batches-containing-eip-7702-transactions-could-be-posted-before-pectra)
  - [FM8: Smart contracts relying on sender check no longer ensures sender does not have code](#fm8-smart-contracts-relying-on-sender-check-no-longer-ensures-sender-does-not-have-code)
  - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)
- [Appendix](#appendix)
  - [Appendix A: Required Code Changes by EIP](#appendix-a-required-code-changes-by-eip)
  - [Appendix B: Block Header Changes](#appendix-b-block-header-changes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- _Italics are used to indicate things that need to be replaced._ -->
|  Related References       |     Links                                   |
| ----------------------- | ------------------------------------------- |
|   Operator Fee FMA        | https://github.com/succinctlabs/optimism-design-docs/blob/main/security/fma-operator-fee.md |
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

See [Appendix A](#appendix-a-required-code-changes-by-eip) for details on which EIPs required new code and where.

## Failure Modes and Recovery Paths

### FM1: Chain halt due to L1 block deployment upgrade transaction failing

- **Description:** The chain could halt if the [L1 block deployment](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/isthmus/derivation.md#l1block-deployment) upgrade transaction can't be included in the upgrade block.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. end-to-end tests created for the L1 block deployment upgrade transaction
      - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
  2. same upgrade path as previous predeploys
      - example: [EIP-4788 deployment transaction](https://github.com/ethereum-optimism/optimism/blob/b6b74290b5502f45daae57946f969327cdb2d383/op-node/rollup/derive/ecotone_upgrade_transactions.go#L130)
    
- **Detection:** Alert for L2 Unsafe liveness should be triggered
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing the network upgrade transaction in rollup node client software (like `op-node`).

### FM2: Chain halt or fork due to request hash mismatch

- **Description:** Requests hash was added to the block header. If it mismatches between EL clients, or the rollup node, the chain could halt or fork for the affected clients.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:**
  1. End-to-end test ensuring requests hash is always `sha256('')` indicating no requests are included in blocks.
      - https://github.com/ethereum-optimism/optimism/pull/14253
  2. Simplicity of validation rules - always must be an empty hash
- **Detection:** Alert for L2 Unsafe liveness should be triggered
- **Recovery Path(s)**: This would require an emergency update if it occurred fixing requests hash logic in respective clients (e.g. `op-node`, `op-geth`, `op-reth`, ...)
- **References**:
    - [Appendix B](#appendix-b-block-header-changes) shows the changes to the block header.
    - [specs: Header validity rules](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/isthmus/exec-engine.md#header-validity-rules)
    - [op-e2e: test ensuring empty hash even if requests created](https://github.com/ethereum-optimism/optimism/blob/9df1fc15d0bf0dc9464db249ce06424607d5f399/op-e2e/actions/proofs/isthmus_requests_test.go#L19)

### FM3: Increased call data cost affects network upgrade transactions

- **Description:** Increased calldata cost as the result of [EIP-7623](https://eips.ethereum.org/EIPS/eip-7623) could mean that network upgrade transactions fail.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Tested that network upgrade transactions don't fail.
        - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
  2. The EIP results in the same gas behavior as long as most of the gas is not spent on call data.
        - Most upgrade transactions create a new contract, and contract creation gas remains the same.
- **Detection:** Manual detection is likely after inspecting the fork block.
- **Recovery Path(s)**: We would have to release an emergency rollup node (e.g. `op-node`) update with fixed network upgrade transactions.

### FM4: EIP-7702 transactions cannot be included on the chain

- **Description:** A client implementation could have a bug that disallows SetCode (EIP-7702) transactions from being included on chain. This could cause an unintended fork by the affected client.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:** 
	1. End-to-end tests of set code transactions before and after the fork
        - https://github.com/ethereum-optimism/optimism/pull/14197
        - https://github.com/ethereum-optimism/optimism/pull/14288
- **Detection:** Sending set code transactions would fail - manual detection is most likely.
- **Recovery Path(s)**: This would require an emergency client update of the implementation with the bug (e.g. `op-reth`, `op-geth`, ...).

### FM5: BLS Precompiles could cause increased FP Program execution time

- **Description:** If implemented in the FP program, BLS precompiles could greatly increase computation time.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program.
        - [Specs](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/isthmus/exec-engine.md#evm-changes)
- **Detection:** op-program trace runner computation time would increase significantly (should trigger an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute
BLS precompiles in sufficient time.

### FM6: BLS Precompiles could fail to execute in the FP program

- **Description:** If implemented in the FP program, BLS precompiles could fail to execute. This could cause certain blocks to be unprovable.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
	1. Fully mitigated by using an accelerated precompile that calls out to L1 instead of calculating inside the program. This means it should 
  have the same performance characteristics as existing crypto precompiles like `ecrecover`.
        - [Specs](https://github.com/ethereum-optimism/specs/blob/b5e0fa98881171f658f782597a46b641e8f3dfd0/specs/protocol/isthmus/exec-engine.md#evm-changes)
- **Detection:** op-program trace runner failure (possibly triggering an alert)
- **Recovery Path(s)**: If this occurred, which it can't with the mitigations, we'd have to make op-program efficient enough to execute BLS precompiles in sufficient time.

### FM7: Early fork if batches containing EIP-7702 transactions could be posted before Pectra

- **Description:** If batches containing EIP-7702 transactions are posted and accepted early, the batcher could fork any nodes that upgrade early. A malicious batcher could cause this to happen if we don't have a check to ensure type 4 (SetCode) txs can only appear in a batch after Isthmus is active.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
  1. Spec disallows type 4 transactions before Isthmus
	1. End-to-end test
        - [Action test created here](https://github.com/ethereum-optimism/optimism/blob/01ddb2e6a09edf55a7cb2130e0a5b6acd0c2d2fa/op-e2e/actions/upgrades/isthmus_fork_test.go#L290)
- **Detection:** Upgraded clients would fork causing an alert on L2 unsafe liveness.
- **Recovery Path(s)**: Emergency patch to rollup node clients (like `op-node`).


### FM8: Smart contracts relying on sender check no longer ensures sender does not have code

- **Description:** As part of EIP-7702, `msg.sender == tx.origin` no longer implies that the caller has no code. If unaddressed, this could cause
some contracts to incorrectly assume the caller has no code.
- **Risk Assessment:** Medium impact, low likelihood
- **Mitigations:**
  1. `isAddress` functions that are used as part of our predeployed contracts check code size, not the condition `msg.sender == tx.origin`.
      - [OpenZeppelin reference](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/0a2cb9a445c365870ed7a8ab461b12acf3e27d63/contracts/utils/AddressUpgradeable.sol#L41)
  2. The checks, `msg.sender == tx.origin` and `tx.origin == msg.sender`, do not appear in the contracts currently used.
- **Detection:** Some contracts that use the `msg.sender == tx.origin` check would allow delegated accounts to execute certain actions they weren't able to before.
- **Recovery Path(s)**: Fix and upgrade the contracts.

### Generic items we need to take into account:

<!-- See [generic hardfork failure modes](./fma-generic-hardfork.md) and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document. -->

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] (NON-BLOCKING)  Make a way to enforce the gas limit of a block can't be lower of 30M (cf this [discord thread](https://discord.com/channels/1244729134312198194/1334603521454247946/1351651773755949117) with @tynes & @sebastianst) 
- [x] (NON-BLOCKING):  Add the description of the new block header with the new field `request_hash` in this FMA (@meyer9)
- [ ] (BLOCKING): Add a test that Validate a block with BLS accelerated precompile (@meyer9 ref to this [comment](https://github.com/ethereum-optimism/design-docs/pull/201/files#r1985623601))
- [ ] (BLOCKING): Add a E2E test that Validate that minting to EOA that contains code (with Type4) is not an issue (@meyer9 assignee for now but could be someone else).
- [ ]  (NON-BLOCKING): Identify path from the sequencer during the deposit transaction on the L2 that can cause unexpected behavior when an EOA has some code.
- [ ]  (NON-BLOCKING): Indicate in this FMA reference the kurtosis devnet that allow the perform testing on Isthmus (L2 Pectra).
- [ ]  (NON-BLOCKING): Tests to make sure that `ressources_limit` and `gas_limit` for each Superchain are not failling by overflowing from the config on L1 (assignee: @geoknee)

## Audit Requirements
<!-- 
_Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning._ -->

**This project should not require an audit.**

- The security concerns in this project affect liveness more than safety.
  - None of these failures would cause a loss of user funds.
  - Some could cause a chain halt or an unintended fork for some clients (if unmitigated).
  - Mitigations reduce risk of unintended fork or chain halt to near zero.
- The security concerns are more a reputational threat rather than an existential threat.

This change should be secure through extensive testing and real world usage rather than through extreme caution and auditing.

## Appendix

### Appendix A: Required Code Changes by EIP


| EIP      | Description                                                | Impact on L2 Consensus Rules                                                                                                                  | Scope of Changes (new code)                   |
| -------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| EIP-2537 | Precompile for BLS12-381 curve operations                  | Support precompile in op-geth and add support for precompile to FP programs.                                                                  | op-geth, FP programs (op-program, Kona, etc.) |
| EIP-2935 | Save historical block hashes in state                      | Support this predeploy and deploy it by default as a network upgrade transaction.                                                             | op-geth, op-node (upgrade tx)                 |
| EIP-6110 | Supply validator deposits on chain                         | Affects the L1 block header via the `requests_hash` field (see EIP 7685).                                                                     | none                                          |
| EIP-7002 | Execution layer triggerable withdrawals                    | Affects the L1 block header via the `requests_hash` field (see EIP 7685).                                                                     | none                                          |
| EIP-7251 | Increase the MAX_EFFECTIVE_BALANCE                         | Affects the L1 block header via the `requests_hash` field (see EIP 7685).                                                                     | none                                          |
| EIP-7549 | Move committee index outside Attestation                   | None, since it only affects beacon chain.                                                                                                     | none                                          |
| EIP-7623 | Increase calldata cost                                     | Support change in op-geth; no extra changes are required.                                                                                     | none                                          |
| EIP-7685 | General purpose execution layer requests                   | Adds a new field `requests_hash` to the L1 block  which always must reflect an empty requests array.                                          | op-node (engine API, block header)            |
| EIP-7691 | Blob throughput increase                                   | None, since OP chains do not support blob txs.                                                                                                | none                                          |
| EIP-7702 | Set EOA account code                                       | Support the new SetCode tx type on L2.                                                                                                        | op-batcher, op-node (span batch format)       |
| EIP-7840 | Add blob schedule to EL config files                       | None, since OP chains do not support blob txs.                                                                                                | none                                          |

### Appendix B: Block Header Changes

```diff
  // Header represents a block header in the Ethereum blockchain.
  type Header struct {
  	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
  	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
  	Coinbase    common.Address `json:"miner"`
  	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
  	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
  	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
  	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
  	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
  	Number      *big.Int       `json:"number"           gencodec:"required"`
  	GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
  	GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
  	Time        uint64         `json:"timestamp"        gencodec:"required"`
  	Extra       []byte         `json:"extraData"        gencodec:"required"`
  	MixDigest   common.Hash    `json:"mixHash"`
  	Nonce       BlockNonce     `json:"nonce"`
  
  	// BaseFee was added by EIP-1559 and is ignored in legacy headers.
  	BaseFee *big.Int `json:"baseFeePerGas" rlp:"optional"`
  
  	// WithdrawalsHash was added by EIP-4895 and is ignored in legacy headers.
  	WithdrawalsHash *common.Hash `json:"withdrawalsRoot" rlp:"optional"`
  
  	// BlobGasUsed was added by EIP-4844 and is ignored in legacy headers.
  	BlobGasUsed *uint64 `json:"blobGasUsed" rlp:"optional"`
  
  	// ExcessBlobGas was added by EIP-4844 and is ignored in legacy headers.
  	ExcessBlobGas *uint64 `json:"excessBlobGas" rlp:"optional"`
  
  	// ParentBeaconRoot was added by EIP-4788 and is ignored in legacy headers.
  	ParentBeaconRoot *common.Hash `json:"parentBeaconBlockRoot" rlp:"optional"`
  
+ 	// RequestsHash was added by EIP-7685 and is ignored in legacy headers.
+ 	RequestsHash *common.Hash `json:"requestsHash" rlp:"optional"`
 }

```