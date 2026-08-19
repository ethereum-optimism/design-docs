# Sequencer-Defined Metering (SDM) and Post-Execution Transactions: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: PostExec construction or application diverges, causing invalid blocks or chain splits](#fm1-postexec-construction-or-application-diverges-causing-invalid-blocks-or-chain-splits)
  - [FM2: Incorrect refunds cause economic loss](#fm2-incorrect-refunds-cause-economic-loss)
  - [FM3: Refunds bypass block resource limits](#fm3-refunds-bypass-block-resource-limits)
  - [FM4: Fault-proof programs cannot prove an SDM block](#fm4-fault-proof-programs-cannot-prove-an-sdm-block)
  - [FM5: Activation or operator configuration is inconsistent](#fm5-activation-or-operator-configuration-is-inconsistent)
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

This document covers failure modes introduced by Sequencer-Defined Metering (SDM), the version-1 payload carried
by the OP Stack's post-execution transaction (`0x7D`). It covers the PostExec envelope, refund application and
settlement, block production and streaming, derivation, fault proofs, activation, operator controls, and RPC
compatibility.

SDM lets the sequencer reduce the gas charged to a standard Ethereum transaction. The sequencer appends at most
one `0x7D` transaction as the final transaction in a block. Its payload contains the block number and a list of
`(transaction index, gas refund)` entries. Verifiers apply the supplied values when calculating canonical gas,
receipts, fee settlement, and the state root.

The refund **mechanism** is consensus-critical, but the refund **policy** is sequencer-defined. Verifiers enforce
payload structure, `refund <= evmGasUsed`, and settlement validity; they do not recompute whether a refund was
justified by the sequencer's intended policy. The version-1 production policy is block-level warming: it rebates
an EIP-2929 cold-to-warm surcharge only when the transaction actually paid that surcharge. This policy boundary
is an explicit sequencer trust assumption.

SDM activates with the Lagoon network upgrade. Producing refunds additionally requires an operator-controlled
execution-layer opt-in. Verification does not depend on this opt-in: after activation, every verifier must accept
and apply a valid SDM payload if the sequencer did.

The normative specifications are the source of truth:

- [Post-Execution Transactions](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/post-exec.md)
- [Sequencer-Defined Metering](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/sdm.md)
- [Lagoon Network Upgrade](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/lagoon/overview.md)
- [SDM design document](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/sdm.md)

## Failure Modes and Recovery Paths

### FM1: PostExec construction or application diverges, causing invalid blocks or chain splits

- **Description:** The producer's streamed, derived, or sealed block views may contain different PostExec
  payloads; a sealed payload may be malformed; or consensus consumers may apply the same valid payload
  differently. Examples include a stale cumulative payload in an incremental stream, multiple `0x7D`
  transactions, incorrect position or block anchor, invalid transaction indices, applying a refund twice, or
  calculating settlement differently. A stream-only mismatch breaks pending views and preconfirmations. Uniform
  rejection of a malformed sealed block causes a deposits-only replacement and user-transaction reorg. A
  validity or state-root disagreement between clients causes a chain split.
- **Risk Assessment:** High impact, low likelihood. PostExec affects consensus-visible gas accounting, receipts,
  balances, and state roots. The specification, shared execution code, and execution-layer validation reduce the
  likelihood, but producer lifecycle bugs or an independent implementation can still diverge.
- **Existing Mitigations:**
  1. The specifications require at most one final `0x7D`, the correct block-number anchor, an active schema, and
     non-empty, non-zero entries ordered by strictly increasing transaction index.
  2. Entries may target only standard Ethereum transactions. Execution enforces `refund <= evmGasUsed`, rejects
     settlement underflow, and consumes every entry exactly once.
  3. Incremental streams use replace-not-append semantics for the cumulative PostExec payload; the final streamed
     bytes are compared with the sealed transaction, and sealing aborts if PostExec finalization fails.
  4. Derivation enforces activation while the execution layer remains authoritative for post-activation payload
     validity and deposits-only recovery.
  5. Existing unit and acceptance tests cover structural rejection, producer/verifier round trips, and
     streamed-to-sealed payload equality.
- **Possible Mitigations:**
  1. Add a non-empty fixture that compares execution-client and fault-proof-program block hash, receipts root,
     gas used, and state root.
  2. Add deposits-only replacement metrics and cross-client malformed-payload recovery coverage.
  3. Add consumer-side checks and alerts for missing or inconsistent cumulative streamed payloads.
- **Existing Detection:** Execution-engine `INVALID` responses, replica state-root or block-hash disagreement,
  fault-proof disagreement, and differential-test failures.
- **Possible Detection:** Alerts for pending-versus-canonical hash disagreement and deposits-only replacements.
- **Recovery Path(s):** A stream-only mismatch self-heals when the canonical block arrives but invalidates the
  affected preconfirmation. Uniform rejection of a sealed payload results in a deposits-only block and requires
  users to resubmit omitted transactions. A live client split requires an emergency release and coordinated
  choice of canonical behavior.

### FM2: Incorrect refunds cause economic loss

- **Description:** A block may be structurally valid while carrying refunds that do not match the operator's
  intended policy. An over-refund reduces fee-vault or sequencer revenue; an under-refund overcharges users; a
  selectively applied policy may violate operator commitments. Verifiers cannot distinguish an intended refund
  from an erroneous one because refund policy is not part of consensus. Historically, this class included
  rebates for intrinsically warm or otherwise unmetered accesses: the transaction received a cold-to-warm rebate
  even though the EVM never charged the cold surcharge.
- **Risk Assessment:** Medium impact, low likelihood. The likely outcome is bounded economic misallocation rather
  than a chain split. A settlement conservation bug would have higher impact, but is constrained by consensus
  checks and conservation tests.
- **Existing Mitigations:**
  1. Consensus limits each refund to the transaction's `evmGasUsed` and rejects settlement underflow.
  2. Settlement tests assert that sender credits equal recipient debits across transaction types, fee settings,
     and arithmetic boundaries.
  3. Policy tests cover EIP-2929 metered accesses and known intrinsically warm or protocol-generated accesses that
     must not receive a rebate.
  4. Stateful policy data is snapshotted and restored when a candidate transaction fails execution or is not
     committed to the block.
  5. Operators can disable refund production without changing verification behavior.
- **Possible Mitigations:**
  1. Independently audit the production policy and its block-building integration.
  2. Add sampled policy-aware reconciliation and alerts for anomalous refund volume or refund-to-raw-gas ratios.
- **Existing Detection:** Refund totals and fee-vault balance changes are observable through metrics and
  canonical block data.
- **Possible Detection:** Alert on refund-to-raw-gas anomalies and compare sampled payloads with an independent
  policy-aware checker.
- **Recovery Path(s):** Disable refund production, correct the policy, and redeploy. Already finalized refunds are
  consensus-valid and should not be rolled back; material losses require accounting and, if appropriate,
  treasury reconciliation. A true value-conservation failure requires an emergency upgrade.

### FM3: Refunds bypass block resource limits

- **Description:** Canonical gas decreases after refunds. If block admission used only post-refund gas, a producer
  could include substantially more EVM work than the block gas limit intends, causing replicas and challengers to
  fall behind even though the header's `gasUsed` remains low.
- **Risk Assessment:** High impact, low likelihood. Unbounded execution threatens network and fault-proof
  liveness. The current design mitigates this by enforcing the block limit against gas used before SDM refunds.
- **Existing Mitigations:**
  1. Transaction admission and block validation accumulate `evmGasUsed` before SDM refunds and enforce the block
     gas limit against that value.
  2. The accumulator spans the complete block, including builders that construct a block incrementally from
     multiple subblocks.
  3. Tests cover producer and verifier behavior near the gas limit and payloads that attempt to hide pre-refund
     overuse.
- **Possible Mitigations:**
  1. Add an explicit end-of-block assertion that total pre-refund gas remains within the block limit.
  2. Alert on unusually high raw EVM gas relative to canonical gas.
- **Existing Detection:** Block execution time and replica lag are observable through standard node monitoring.
- **Possible Detection:** Expose raw EVM gas alongside canonical gas and alert on an unusually high ratio.
- **Recovery Path(s):** Disable SDM production or reduce sequencing load while patching the affected builder or
  verifier. If deployed clients disagree on the resource-limit rule, recovery follows the chain-split path in
  FM1.

### FM4: Fault-proof programs cannot prove an SDM block

- **Description:** An execution client may validate an SDM block while the deployed fault-proof program fails to
  reproduce it. Causes include different transaction decoding, settlement behavior, configuration, or behavior
  that works in native execution but fails inside the production FPVM. SDM also activates with Lagoon, whose
  dependency-set and proof-program requirements must be satisfied by the target chain.
- **Risk Assessment:** High impact, medium likelihood until production-FPVM coverage exists. An unprovable output
  stalls permissionless withdrawals and may require replacing the fault-proof program under governance.
- **Existing Mitigations:**
  1. Fault-proof execution uses the same PostExec parser and refund application logic as the execution client.
  2. Native proof tests exercise non-empty SDM blocks.
- **Possible Mitigations:**
  1. Prove a non-empty SDM block in the production FPVM rather than only running the program natively.
  2. Add a post-Isthmus, Lagoon-active fixture that compares execution-client and proof-program block hash and
     state root.
  3. Document the compatible proof program and dependency-set requirements for SDM-active chains.
- **Existing Detection:** Proof-runner failures, challenger disagreement, and output-root disagreement in native
  acceptance tests.
- **Possible Detection:** A production-FPVM regression test that detects native-versus-FPVM trace divergence.
- **Recovery Path(s):** Deploy a corrected proof program and publish a new absolute prestate through the approved
  governance and registry process. Use the established fault-proof recovery process while the corrected program
  is prepared.

### FM5: Activation or operator configuration is inconsistent

- **Description:** SDM uses the Lagoon activation timestamp across the consensus client, execution client, and
  fault-proof stack. A timestamp mismatch can make one component produce or accept `0x7D` while another considers
  it inactive. Legacy configuration keys may be silently ignored. An execution client without PostExec support
  cannot follow an SDM-active chain. Missing proof-program prerequisites can prevent proving at activation.
  Separately, the operator opt-in defaults off and may differ across sequencer instances, causing refunds to stop
  or vary after a restart. Exposing the mutating admin method can let an unauthorized caller toggle production.
- **Risk Assessment:** High impact, low likelihood with an activation preflight. Cross-component activation
  disagreement can halt or split the chain. An opt-in-only mismatch has lower impact because verification is
  independent of opt-in and missing refunds remain valid.
- **Existing Mitigations:**
  1. Every component derives SDM activation from the Lagoon schedule.
  2. Refund production is enabled only when both Lagoon is active and the operator opt-in is set.
  3. Verification ignores the local production opt-in.
  4. The operator opt-in defaults off, supports boot-time configuration, and exposes effective state through the
     admin status method.
- **Possible Mitigations:**
  1. Add a cross-component activation preflight covering timestamps, legacy keys, execution-client support,
     proof-program selection, and dependency-set configuration.
  2. Add restart health checks that verify the intended effective opt-in state.
  3. Document and test that mutating admin methods are not exposed on public HTTP or WebSocket endpoints.
- **Existing Detection:** The admin status method exposes protocol activation and effective opt-in state;
  standard replica-divergence and proof-failure alerts detect severe mismatches.
- **Possible Detection:** Cross-node configuration checks and alerts for unexpected opt-in changes or unexpected
  presence or absence of `0x7D`.
- **Recovery Path(s):** Before activation, correct configuration and reschedule if necessary. After activation,
  an opt-in error is fixed by restoring the intended setting. A consensus activation mismatch may require a
  coordinated configuration update, rollback, or emergency hardfork.

### FM6: RPC or receipt handling breaks consumers or misattributes refunds

- **Description:** PostExec introduces a signatureless EIP-2718 transaction exposed through standard RPC methods.
  Strict consumers may reject the unfamiliar `0x7D` shape, assume every transaction has signature fields, or
  mishandle its zero-gas receipt. Standard transaction receipts additionally expose `opGasRefund`, indexed by the
  transaction's block-global position. Filtering or reordering transactions can attribute a valid refund to the
  wrong receipt. Inconsistent auxiliary fee fields on the PostExec receipt can also confuse explorers and
  accounting systems without affecting consensus.
- **Risk Assessment:** Medium impact, low-to-medium likelihood. This is primarily an ecosystem compatibility and
  accounting risk, not a consensus risk.
- **Existing Mitigations:**
  1. The PostExec specification defines the minimal RPC transaction shape, receipt behavior, and
     `opGasRefund` semantics.
  2. Payload validation guarantees ordered, unique, in-range indices before receipt projection.
  3. Unit and acceptance tests cover PostExec serialization and receipt refund projection.
- **Possible Mitigations:**
  1. Add end-to-end coverage for `eth_getBlockByNumber`, `eth_getTransactionByHash`, transaction/receipt
     cross-links, and representative generic Ethereum decoders.
  2. Publish the RPC schema and coordinate with explorer and infrastructure providers before activation.
  3. Make auxiliary receipt fee fields semantically consistent or explicitly document them.
- **Existing Detection:** Unit and acceptance tests detect serialization and receipt-projection regressions; user
  or provider reports expose unsupported consumer behavior.
- **Possible Detection:** Generic-decoder compatibility tests, receipt-to-payload reconciliation, and explorer
  integration testing.
- **Recovery Path(s):** Patch the RPC serialization or consumer integration. Historical refund data remains
  recoverable from the canonical PostExec payload. RPC-only fixes do not require a hardfork unless the underlying
  consensus receipt is incorrect.

### Generic items

The [generic hardfork failure modes](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md)
apply, especially activation coordination, cross-client divergence, chain-split detection, fault-proof prestate
updates, and rollback procedures. SDM adds no L1 or L2 contracts, so the generic smart-contract FMA is not
applicable.

- [x] The applicable generic hardfork failure modes have been reviewed and incorporated above.

## Action Items

The following are candidate follow-ups for Security and engineering review. They are **not commitments or launch
blockers** until an owner accepts them and a tracking issue or PR is created.

| ID | Candidate Follow-up | Failure Modes | Owner | Status |
| --- | --- | --- | --- | --- |
| A1 | Run a production-FPVM proof over a block with a non-empty SDM payload. | FM1, FM4 | _TBD_ | Proposed |
| A2 | Add an action test that compares execution-client and proof-program block hash and state root. | FM1, FM4 | _TBD_ | Proposed |
| A3 | Document the proof-program and dependency-set requirements for every SDM-active chain. | FM4, FM5 | _TBD_ | Proposed |
| A4 | Implement and run the cross-component activation preflight described in FM5. | FM5 | _TBD_ | Proposed |
| A5 | Resolve and specify how SDM canonical gas interacts with the EIP-7623 transaction gas floor. | FM1, FM2 | _TBD_ | Proposed |
| A6 | Add end-to-end RPC schema, receipt cross-link, and generic-decoder compatibility tests; publish the integration schema. | FM6 | _TBD_ | Proposed |
| A7 | Complete an external audit of the public mechanism and production refund policy/builder. | FM1–FM4 | _TBD_ | Proposed |
| A8 | Add deposits-only replacement metrics and an acceptance test for malformed PostExec recovery across supported consensus clients. | FM1 | _TBD_ | Proposed |
| A9 | Add consumer-side cumulative-payload checks and pending-versus-sealed mismatch alerts. | FM1 | _TBD_ | Proposed |
| A10 | Add refund anomaly alerts and sampled policy-aware reconciliation. | FM2, FM3 | _TBD_ | Proposed |
| A11 | Decide and document the PostExec receipt's auxiliary L1 fee fields. | FM6 | _TBD_ | Proposed |
| A12 | Add an operator runbook for opt-in health checks, restarts, and admin-RPC exposure. | FM5 | _TBD_ | Proposed |

- [ ] Resolve all review comments and incorporate accepted decisions into this document (Assignee: document author).

## Audit Requirements

**Recommendation: complete an external audit before production activation.**

The audit should cover:

- PostExec decoding and structural validation;
- SDM canonical-gas calculation and fee settlement, including value conservation and arithmetic boundaries;
- pre-refund block resource-limit enforcement;
- transaction rollback and policy snapshot behavior;
- producer, streamed-subblock, and sealed-block consistency;
- production refund-policy correctness against the EIP-2929 metered-access rules; and
- execution-client versus fault-proof-program parity, including the production FPVM.

No new L1 or L2 contracts are introduced. Re-audit triggers include a new payload schema, a new refund policy,
configurable policy composition, changes to settlement, changes to the pre-refund resource limit, or a new
independent implementation of PostExec verification.

## Appendix

### Appendix A: Review Scope and Method

This analysis was produced by reviewing the public Lagoon PostExec and SDM specifications, the public execution,
derivation, RPC, and fault-proof implementations, and their unit and acceptance tests. The original working notes
included detailed file/line inventories and repository-maintenance observations; those are intentionally omitted
here because they become stale quickly and do not change the durable failure modes.

Implementation-specific audit evidence should use commit-pinned links and be maintained with the audit or
production-readiness records rather than duplicated in this public FMA.
