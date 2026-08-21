# Sequencer-Defined Metering (SDM) and Post-Execution Transactions: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1a: Structurally invalid or inconsistent PostExec blocks](#fm1a-structurally-invalid-or-inconsistent-postexec-blocks)
  - [FM1b: PostExec application diverges across clients](#fm1b-postexec-application-diverges-across-clients)
  - [FM2a: Incorrect policy refunds cause economic loss](#fm2a-incorrect-policy-refunds-cause-economic-loss)
  - [FM2b: Settlement violates value conservation](#fm2b-settlement-violates-value-conservation)
  - [FM2c: EIP-7623 calldata-floor interaction is inconsistent](#fm2c-eip-7623-calldata-floor-interaction-is-inconsistent)
  - [FM3: Refunds bypass block resource limits](#fm3-refunds-bypass-block-resource-limits)
  - [FM4: Fault-proof programs cannot prove an SDM block](#fm4-fault-proof-programs-cannot-prove-an-sdm-block)
  - [FM5a: Lagoon activation is inconsistent](#fm5a-lagoon-activation-is-inconsistent)
  - [FM5b: Sequencer SDM production configuration drifts](#fm5b-sequencer-sdm-production-configuration-drifts)
  - [FM6: RPC or receipt handling breaks consumers or misattributes refunds](#fm6-rpc-or-receipt-handling-breaks-consumers-or-misattributes-refunds)
  - [Generic items](#generic-items)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)
- [Appendix](#appendix)
  - [Appendix A: Review Scope and Method](#appendix-a-review-scope-and-method)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                   |
| ------------------ | ----------------- |
| Author             | Anton Evangelatov |
| Created at         | 2026-08-14        |
| Initial Reviewers  | _TBD_             |
| Need Approval From | _TBD (Security)_  |
| Status             | Draft 📝          |

## Introduction

This document covers failure modes introduced by Sequencer-Defined Metering (SDM), the version-1 payload carried by the OP Stack's post-execution transaction (`0x7D`). It covers the PostExec envelope, refund application and settlement, block production and subblock streaming, derivation, fault proofs, activation, operator controls, and RPC compatibility.

SDM lets the sequencer reduce the gas charged to a standard Ethereum transaction. The sequencer appends at most one `0x7D` transaction as the final transaction in a block. Its payload contains the block number and a list of `(transaction index, gas refund)` entries. Verifiers apply the supplied values when calculating canonical gas, receipts, fee settlement, and the state root.

The refund **mechanism** is consensus-critical, but the refund **policy** is sequencer-defined. Verifiers enforce payload structure, `refund <= evmGasUsed`, and settlement validity; they do not recompute whether a refund was justified by the sequencer's intended policy. The version-1 production policy is block-level warming: it rebates an EIP-2929 cold-to-warm surcharge only when the transaction actually paid that surcharge. This policy boundary is an explicit sequencer trust assumption.

SDM activates with the Lagoon network upgrade. Producing refunds additionally requires an operator-controlled execution-layer opt-in. Verification does not depend on this opt-in: after activation, every verifier must accept and apply a valid SDM payload if the sequencer did.

The normative specifications are the source of truth:

- [Post-Execution Transactions](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/post-exec.md)
- [Sequencer-Defined Metering](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/sdm.md)
- [Lagoon Network Upgrade](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/overview.md)
- [SDM design document](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/sdm.md)

## Failure Modes and Recovery Paths

### FM1a: Structurally invalid or inconsistent PostExec blocks

- **Description:** A producer may seal a malformed PostExec block whether it uses standard block building or incremental subblocks. Examples include multiple `0x7D` transactions, an incorrect position or block anchor, and invalid transaction indices. A batcher may omit, corrupt, or reorder `0x7D`; omission can produce a valid derived block without refunds that differs from the unsafe block, while corruption or reordering can make the derived payload invalid. With subblock streaming enabled, a producer may additionally emit a stale cumulative PostExec payload or a final subblock-stream payload that differs from the sealed transaction. A subblock-stream-only mismatch breaks pending views and preconfirmations. Uniform rejection of a malformed sealed block causes a deposits-only replacement and user-transaction reorg.
- **Risk Assessment:** High impact, low likelihood. Producer lifecycle bugs can invalidate preconfirmations or omit user transactions from the derived chain. The specification, execution-layer validation, and lifecycle tests reduce the likelihood.
- **Existing Mitigations:**
  1. The specifications require at most one final `0x7D`, the correct block-number anchor, an active schema, and non-empty, non-zero entries ordered by strictly increasing transaction index.
  2. Incremental subblock streams use replace-not-append semantics for the cumulative PostExec payload; the final subblock-stream payload is compared with the sealed transaction, and sealing aborts if PostExec finalization fails.
  3. Before Lagoon activation, blocks containing PostExec transactions are invalid. After activation, the execution layer validates PostExec payloads, while derivation handles deposits-only recovery when a payload is invalid.
  4. Existing unit and acceptance tests cover structural rejection, subblock-stream-to-sealed payload equality, and PostExec derivation through singular and span batches.
- **Possible Mitigations:**
  1. Add deposits-only replacement metrics and cross-client malformed-payload recovery coverage.
- **Existing Detection:** Execution-engine `INVALID` responses and differential-test failures.
- **Possible Detection:** Alerts for deposits-only replacements.
- **Recovery Path(s):** A subblock-stream-only mismatch self-heals when the canonical block arrives but invalidates the affected preconfirmation. Batcher omission can reorg the unsafe block to a valid derived block without refunds. Uniform rejection of a malformed sealed or batched payload results in a deposits-only block and requires users to resubmit omitted transactions.

### FM1b: PostExec application diverges across clients

- **Description:** Native op-reth execution and kona-executor's fault-proof execution path may apply the same valid PostExec payload differently. Examples include applying a refund twice or calculating canonical gas, receipts, or settlement differently. A validity, block-hash, or state-root disagreement between the two paths causes a chain split or an invalid fault-proof result.
- **Risk Assessment:** High impact, low likelihood. PostExec affects consensus-visible gas accounting, receipts, balances, and state roots. The two paths share core execution code, reducing the likelihood, but their parsing, configuration, and integration paths can still diverge.
- **Existing Mitigations:**
  1. Entries may target only standard Ethereum transactions. Execution enforces `refund <= evmGasUsed`, rejects settlement underflow, and consumes every entry exactly once.
  2. Existing tests cover producer/verifier round trips, settlement, and structural validity.
- **Possible Mitigations:**
  1. Add op-reth and kona-executor parity coverage for a block containing an SDM refund.
  2. Add differential coverage across native and fault-proof execution for valid and malformed PostExec payloads.
- **Existing Detection:** Replica block-hash or state-root disagreement, fault-proof disagreement, and differential-test failures.
- **Possible Detection:** Alerts for cross-client block-hash disagreement.
- **Recovery Path(s):** A live client split requires an emergency release and coordinated choice of canonical behavior.

### FM2a: Incorrect policy refunds cause economic loss

- **Description:** A block may be structurally valid while carrying refunds that do not match the operator's intended policy. An over-refund reduces fee-vault or sequencer revenue; an under-refund overcharges users; a selectively applied policy may violate operator commitments. Verifiers cannot distinguish an intended refund from an erroneous one because refund policy is not part of consensus. Historically, this class included rebates for intrinsically warm or otherwise unmetered accesses: the transaction received a cold-to-warm rebate even though the EVM never charged the cold surcharge.
- **Risk Assessment:** Medium impact, low likelihood. The economic misallocation is bounded per transaction by its `evmGasUsed` and does not cause a chain split.
- **Existing Mitigations:**
  1. Consensus limits each refund to the transaction's `evmGasUsed`.
  2. Policy tests cover EIP-2929 metered accesses and known intrinsically warm or protocol-generated accesses that must not receive a rebate.
  3. Stateful policy data is snapshotted and restored when a candidate transaction fails execution or is not committed to the block.
  4. Operators can disable refund production without changing verification behavior.
- **Possible Mitigations:**
  1. Independently audit the production policy and its block-building integration.
  2. Add sampled policy-aware reconciliation and alerts for anomalous refund volume or refund-to-raw-gas ratios.
- **Existing Detection:** Refund totals and fee-vault balance changes are observable through metrics and canonical block data.
- **Possible Detection:** Alert on refund-to-raw-gas anomalies and compare sampled payloads with an independent policy-aware checker.
- **Recovery Path(s):** Disable refund production, correct the policy, and redeploy. Already finalized refunds are consensus-valid and should not be rolled back; material losses require accounting and, if appropriate, treasury reconciliation.

### FM2b: Settlement violates value conservation

- **Description:** A settlement implementation bug may credit the sender without applying equal debits, debit the wrong fee recipient, or otherwise violate value conservation. If deployed consistently across clients, this could mint ETH or drain fee-vault or beneficiary balances without causing an immediate chain split.
- **Risk Assessment:** Very high impact, very low likelihood. A value-conservation failure can corrupt canonical balances, but the settlement equations, underflow checks, and conservation tests substantially reduce the likelihood.
- **Existing Mitigations:**
  1. Settlement requires sender credits to equal the sum of recipient debits and rejects debit underflow.
  2. Settlement tests cover transaction types, fee settings, and arithmetic boundaries.
  3. Operators can disable refund production without changing verification behavior.
- **Possible Mitigations:**
  1. Independently audit settlement and add invariant or property tests for value conservation.
  2. Reconcile sender credits and fee-recipient debits from sampled canonical blocks.
- **Existing Detection:** Conservation and differential tests detect known settlement discrepancies before deployment.
- **Possible Detection:** Alert on reconciliation failures or anomalous fee-vault and beneficiary balance changes.
- **Recovery Path(s):** Disable refund production immediately and deploy corrected clients. Canonical balance corruption may require an emergency upgrade and the established incident-response process.

### FM2c: EIP-7623 calldata-floor interaction is inconsistent

- **Description:** [EIP-7623](https://eips.ethereum.org/EIPS/eip-7623) computes a calldata-dependent minimum charge after the EVM's native gas refund. For calldata token count `T`, let `F = 21,000 + 10T` be that floor and let `G_evm` be the gas the EVM reports before SDM; therefore `G_evm >= F`. The current SDM specification then applies the sequencer-defined refund `R` outside the EVM:

  ```text
  G_canonical = G_evm - R
  ```

  This ordering lets `G_canonical` fall below `F`. By contrast, a floor-preserving interpretation would apply or clip the rebate before the final floor:

  ```text
  G_canonical = max(G_evm - R, F)
  ```

  or, equivalently for a payload that records the amount actually credited, require `R <= G_evm - F`. The distinction is reachable: a calldata-heavy transaction can be floor-bound while touching an account or slot warmed by an earlier transaction, and the production block-warming policy does not currently inspect or clip against the calldata floor.

  The current public executor mostly follows the first interpretation. Its explicit `canonical_gas_used`, receipt cumulative gas, block `gasUsed`, settlement, and builder-facing committed-gas output subtract the full SDM refund. However, the underlying EVM `ResultGas` retains its EIP-7623 floor, so its generic `tx_gas_used()` accessor remains clamped at `F`. A floor-bound refunded transaction can therefore have two different post-execution gas values inside the same process. This is not currently used to validate the block, but it is a fragile interface for tracing, builder policy, metrics, and future execution consumers.
- **Risk Assessment:** High impact, low likelihood for consensus divergence; medium impact for economics and compatibility if all clients make the same choice. A producer, verifier, or fault-proof implementation that clamps at `F` will disagree with one that subtracts the full refund, changing receipts, header `gasUsed`, fee settlement, and the state root. If every client follows the current full-refund behavior, there is no chain split or value-conservation failure, but the sequencer can subsidize gas below Ethereum's calldata floor and lower the gas signal used for base-fee adjustment. This may be intended for an out-of-EVM sequencer rebate, but it is not equivalent to implementing block-level warming directly in the EVM gas schedule, where EIP-7623's final `max` would continue to bind.
- **Existing Mitigations:**
  1. Block admission and validation use pre-SDM `G_evm`, which already includes the EIP-7623 floor. Consequently, an SDM refund cannot use this ambiguity to fit extra calldata or EVM work under the block gas limit.
  2. The current producer and verifier share the explicit `canonical_gas_used` accounting path, reducing the chance that those two roles accidentally choose different orderings.
  3. Settlement debits exactly match the sender credit for the applied `R`; allowing gas below `F` does not by itself mint value.
  4. Production is operator-gated and can be disabled while the behavior is resolved.
- **Possible Mitigations:**
  1. Before production activation, make the ordering normative in the SDM specification. State whether the EIP-7623 floor constrains only transaction validity and pre-refund block admission, or also constrains canonical gas and the amount that SDM may credit.
  2. If the floor remains binding, encode only the effective, clipped refund and enforce `R <= G_evm - F` in producers and verifiers. If SDM may cross the floor, make every post-execution gas view report `G_evm - R` consistently rather than retaining a conflicting generic EVM floor-clamped value.
  3. Add a floor-bound block-warming vector to the public executor, receipt/block validation, fault-proof program, standard and subblock premium producers, replay tooling, and RPC tests. Cover values below, exactly at, and above the floor, including a transaction with a native EVM refund as well as an SDM refund.
  4. Document that gas estimation must still reserve the EIP-7623 floor even if the eventual SDM charge may be lower; an SDM rebate does not make a transaction with `gasLimit < F` valid.
- **Existing Detection:** A disagreement is visible as a receipt-root, state-root, or block-hash mismatch. The current implementation comment records the open question, but there is no dedicated floor-bound SDM test.
- **Possible Detection:** Assert equality between receipt-derived gas, block accumulation, settlement gas, generic execution-result gas, replay output, and builder-policy inputs for the floor-bound vector. Alert or report when canonical gas is below the transaction's EIP-7623 floor so the chosen behavior is observable in deployment.
- **Recovery Path(s):** Before activation, settle the specification and patch all implementations. After activation, an operator can stop producing new refunds while clients are patched. Already accepted blocks must be interpreted under the activated rule; changing the ordering retroactively would require a coordinated consensus upgrade.

### FM3: Refunds bypass block resource limits

- **Description:** Canonical gas decreases after refunds. If block admission used only post-refund gas, a producer could include substantially more EVM work than the block gas limit intends, creating a resource-exhaustion denial of service as replicas and challengers fall behind even though the header's `gasUsed` remains low.
- **Risk Assessment:** High impact, low likelihood. Unbounded execution threatens network and fault-proof liveness. The current design mitigates this by enforcing the block limit against gas used before SDM refunds.
- **Existing Mitigations:**
  1. Transaction admission and block validation accumulate `evmGasUsed` before SDM refunds and enforce the block gas limit against that value.
  2. The accumulator spans the complete block, including builders that construct a block incrementally from multiple subblocks.
  3. Tests cover producer and verifier behavior near the gas limit and payloads that attempt to hide pre-refund overuse.
- **Possible Mitigations:**
  1. Add an explicit end-of-block assertion that total pre-refund gas remains within the block limit.
  2. Alert on unusually high raw EVM gas relative to canonical gas.
- **Existing Detection:** Block execution time and replica lag are observable through standard node monitoring.
- **Possible Detection:** Expose raw EVM gas alongside canonical gas and alert on an unusually high ratio.
- **Recovery Path(s):** Disable SDM production or reduce sequencing load while patching the affected builder or verifier. If deployed clients disagree on the resource-limit rule, recovery follows the chain-split path in FM1b.

### FM4: Fault-proof programs cannot prove an SDM block

- **Description:** An execution client may validate an SDM block while the deployed fault-proof program fails to reproduce it. Causes include different transaction decoding, settlement behavior, configuration, or behavior that works in native execution but fails inside the production FPVM. The deployed proof program and prestate must support Lagoon and SDM.
- **Risk Assessment:** High impact, medium likelihood until production-FPVM coverage exists. An unprovable output stalls permissionless withdrawals and may require replacing the fault-proof program under governance.
- **Existing Mitigations:**
  1. Fault-proof execution uses the same PostExec parser and refund application logic as the execution client.
  2. Native proof tests exercise non-empty SDM blocks.
- **Possible Mitigations:**
  1. Prove a non-empty SDM block in the production FPVM rather than only running the program natively.
  2. Add a post-Isthmus, Lagoon-active fixture that compares execution-client and proof-program block hash and state root.
- **Existing Detection:** Proof-runner failures, challenger disagreement, and output-root disagreement in native acceptance tests.
- **Possible Detection:** A production-FPVM regression test that detects native-versus-FPVM trace divergence.
- **Recovery Path(s):** Deploy a corrected proof program and publish a new absolute prestate through the approved governance and registry process. Use the established fault-proof recovery process while the corrected program is prepared.

### FM5a: Lagoon activation is inconsistent

- **Description:** SDM uses the Lagoon activation timestamp across the consensus client, execution client, and fault-proof stack. A timestamp mismatch can make one component produce or accept `0x7D` while another considers it inactive. An execution client without PostExec support cannot follow an SDM-active chain. Missing proof-program prerequisites can prevent proving at activation.
- **Risk Assessment:** High impact, low likelihood with an activation preflight. Cross-component activation disagreement can halt or split the chain or make its outputs unprovable.
- **Existing Mitigations:**
  1. The Superchain Registry defines the expected Lagoon activation schedule, and every component derives SDM activation from that schedule.
  2. Before Lagoon activation, blocks containing PostExec transactions are invalid; after activation, every verifier must accept and apply valid PostExec payloads.
- **Possible Mitigations:**
  1. Add a cross-component activation preflight covering timestamps, execution-client support, and proof-program selection.
- **Existing Detection:** Standard replica-divergence and proof-failure alerts detect severe activation mismatches.
- **Possible Detection:** Compare the effective Lagoon schedule reported by every consensus, execution, and fault-proof component before activation.
- **Recovery Path(s):** Before activation, correct configuration and reschedule if necessary. After activation, a consensus mismatch may require a coordinated configuration update, rollback, or emergency hardfork.

### FM5b: Sequencer SDM production configuration drifts

- **Description:** The operator opt-in defaults off and may differ across sequencer instances, causing refunds to stop or vary after a restart. The chain continues because verification is independent of the production opt-in, but the resulting refund behavior may not match the operator's intent. Different valid refund policies likewise do not halt the chain; their economic consequences are covered by FM2a. Exposing the mutating admin method can let an unauthorized caller toggle production.
- **Risk Assessment:** Medium impact, low likelihood with configuration monitoring. Drift does not invalidate blocks or halt the chain, but it can create inconsistent user charges, revenue, and operator-policy compliance.
- **Existing Mitigations:**
  1. Refund production is enabled only when both Lagoon is active and the operator opt-in is set.
  2. Verification ignores the local production opt-in and does not recompute the sequencer's refund policy.
  3. The operator opt-in defaults off, supports boot-time configuration, and exposes effective state through the admin status method.
- **Possible Mitigations:**
  1. Add restart and cross-sequencer health checks that verify the intended effective opt-in.
  2. Document and test that mutating admin methods are not exposed on public HTTP or WebSocket endpoints.
- **Existing Detection:** The admin status method exposes protocol activation and effective opt-in state.
- **Possible Detection:** Cross-sequencer configuration checks and alerts for unexpected opt-in changes or unexpected presence or absence of `0x7D`.
- **Recovery Path(s):** Restore the intended opt-in on every active sequencer and restrict the admin endpoint if it was exposed. Blocks already accepted under the drifted configuration remain consensus-valid.

### FM6: RPC or receipt handling breaks consumers or misattributes refunds

- **Description:** PostExec introduces a signatureless EIP-2718 transaction exposed through standard RPC methods. Strict consumers may reject the unfamiliar `0x7D` shape, assume every transaction has signature fields, or mishandle its zero-gas receipt. Standard transaction receipts additionally expose `opGasRefund`, indexed by the transaction's block-global position. Filtering or reordering transactions can attribute a valid refund to the wrong receipt. Inconsistent auxiliary fee fields on the PostExec receipt can also confuse explorers and accounting systems without affecting consensus.
- **Risk Assessment:** Medium impact, low-to-medium likelihood. This is primarily an ecosystem compatibility and accounting risk, not a consensus risk.
- **Existing Mitigations:**
  1. The PostExec specification defines the minimal RPC transaction shape, receipt behavior, and `opGasRefund` semantics.
  2. Payload validation guarantees ordered, unique, in-range indices before receipt projection.
  3. Unit and acceptance tests cover PostExec serialization and receipt refund projection.
- **Possible Mitigations:**
  1. Add end-to-end coverage for `eth_getBlockByNumber`, `eth_getTransactionByHash`, transaction/receipt cross-links, and representative generic Ethereum decoders.
  2. Publish the RPC schema and coordinate with explorer and infrastructure providers before activation.
  3. Make auxiliary receipt fee fields semantically consistent or explicitly document them.
- **Existing Detection:** Unit and acceptance tests detect serialization and receipt-projection regressions; user or provider reports expose unsupported consumer behavior.
- **Possible Detection:** Generic-decoder compatibility tests, receipt-to-payload reconciliation, and explorer integration testing.
- **Recovery Path(s):** Patch the RPC serialization or consumer integration. Historical refund data remains recoverable from the canonical PostExec payload. RPC-only fixes do not require a hardfork unless the underlying consensus receipt is incorrect.

### Generic items

The [generic hardfork failure modes](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md) apply, especially activation coordination, cross-client divergence, chain-split detection, fault-proof prestate updates, and rollback procedures. SDM adds no L1 or L2 contracts, so the generic smart-contract FMA is not applicable.

- [x] The applicable generic hardfork failure modes have been reviewed and incorporated above.

## Action Items

The following are candidate follow-ups for Security and engineering review. They are **not commitments or launch blockers** until an owner accepts them and a tracking issue or PR is created.

| ID | Candidate Follow-up | Failure Modes | Owner | Status |
| --- | --- | --- | --- | --- |
| A1 | Decide whether SDM subblock-stream behavior needs a normative specification. | FM1a | _TBD_ | Proposed |
| A2 | Agree on required proof coverage for an SDM block: client/program parity, FPVM execution, or an end-to-end dispute game. | FM1b, FM4 | _TBD_ | Proposed |
| A3 | Before production activation, choose and specify the EIP-7623/SDM ordering, make all gas views consistent, and add floor-bound cross-stack vectors. | FM1b, FM2a, FM2c | _TBD_ | Proposed |
| A4 | Complete an external audit of the public mechanism and production refund policy/builder. | FM1a–FM4 | _TBD_ | Proposed |
| A5 | Add deposits-only replacement metrics and an acceptance test for malformed PostExec recovery across supported consensus clients. | FM1a | _TBD_ | Proposed |
| A6 | Add refund anomaly alerts. | FM2a, FM3 | _TBD_ | Proposed |
| A7 | Decide and document the PostExec receipt's auxiliary L1 fee fields. | FM6 | _TBD_ | Proposed |
| A8 | Add an operator runbook for opt-in health checks, restarts, and admin-RPC exposure. | FM5b | _TBD_ | Proposed |

- [ ] Resolve all review comments and incorporate accepted decisions into this document (Assignee: document author).

## Audit Requirements

**Recommendation: complete an external audit before production activation.**

The audit should cover:

- PostExec decoding and structural validation;
- SDM canonical-gas calculation and fee settlement, including value conservation and arithmetic boundaries;
- EIP-7623 floor-bound transactions with SDM and native EVM refunds, including consistency across receipts, settlement, block gas, generic execution results, builders, replay, and fault proofs;
- pre-refund block resource-limit enforcement;
- transaction rollback and policy snapshot behavior;
- producer, subblock-stream, and sealed-block consistency;
- production refund-policy correctness against the EIP-2929 metered-access rules; and
- execution-client versus fault-proof-program parity, including the production FPVM.

No new L1 or L2 contracts are introduced. Re-audit triggers include a new payload schema, a new refund policy, configurable policy composition, changes to settlement, changes to the pre-refund resource limit, or a new independent implementation of PostExec verification.

## Appendix

### Appendix A: Review Scope and Method

This analysis was produced by reviewing the public Lagoon PostExec and SDM specifications, the public execution, derivation, RPC, and fault-proof implementations, and their unit and acceptance tests. The original working notes included detailed file/line inventories and repository-maintenance observations; those are intentionally omitted here because they become stale quickly and do not change the durable failure modes.

Implementation-specific audit evidence should use commit-pinned links and be maintained with the audit or production-readiness records rather than duplicated in this public FMA.
