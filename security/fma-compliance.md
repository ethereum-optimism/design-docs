# Compliance Module: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes](#failure-modes)
- [Risks & Uncertainties](#risks--uncertainties)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | tynes                                              |
| Created at         | 2026-02-25                                         |
| Needs Approval From | _TBD_                                             |
| Status             | Draft                                              |

## Introduction

This document covers failure modes for the **Compliance Module**, an optional on-chain compliance screening layer for deposits (`OptimismPortal2`) and withdrawals (`L2ToL1MessagePasser`). The module introduces an abstract `Compliance` contract (inheriting Solady's `Ownable`, using Solady's `EnumerableSetLib` for rule management) with two concrete implementations: `L1Compliance` (deployed on L1) and `L2Compliance` (deployed as a predeploy at `0x420000000000000000000000000000000000002D` on L2). Both use composable `IRule` contracts for configurable screening logic, an owner-controlled approval/rejection flow for flagged transactions, and user-initiated refunds for rejected/blocked transactions.

**References:**

- [Compliance Module Design Doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/compliance-module.md)

## Failure Modes

<details>
<summary>FM1: Owner Key Compromise</summary>

#### Description

The `Compliance` contract inherits Solady's `Ownable`. The owner controls approval and rejection of flagged transactions. If compromised, an attacker could approve sanctioned/malicious transactions that were correctly flagged by rules, or reject legitimate transactions to grief users. The owner can also call `addRule/removeRule` to modify the compliance rules themselves.

#### Risk Assessment

High severity / Low likelihood

#### Mitigations

- Use a multisig or governance-gated address for the owner role to reduce single-key risk
- Key management best practices (HSM, key rotation policies)
- The owner can only act on flagged transactions — compliant transactions that pass all rules are never held

#### Detection

- Monitor `Approved` and `Rejected` events for anomalous patterns (e.g., bulk approvals, approvals of known sanctioned addresses)
- Alert on `addRule/removeRule` calls from the owner

#### Recovery Path

- Call `transferOwnership` to rotate to a new owner address (or governance upgrades the proxy if the owner key is compromised)
- Users whose transactions were incorrectly rejected can still call `settle` to recover their ETH
- If rules were tampered with via `addRule/removeRule`, restore correct rules

#### Action Items

- [ ] Define owner key management requirements (multisig, HSM, etc.)
- [ ] Implement monitoring for owner actions

</details>

<details>
<summary>FM2: Compliance Contract Bug Blocks All Transactions</summary>

#### Description

A bug in the `check()` function or in the configured rules causes all deposits/withdrawals to revert or be incorrectly flagged as `Pending`/`Rejected`. Since the compliance check is in the critical path of `depositTransaction()` and `initiateWithdrawal()`, this would effectively halt all cross-chain transactions for the affected chain.

#### Risk Assessment

High severity / Medium likelihood (new code with external rule calls in the hot path)

#### Mitigations

- Setting `compliance` to `address(0)` disables the module entirely, providing an escape hatch. Note: this requires governance action (proxy upgrade or reinitialization on L1, `setCompliance` via L2ProxyAdmin owner on L2) to maintain stage 1 requirements
- Thorough testing of the `check()` path with various rule configurations

#### Detection

- Monitor for sudden drop in successful deposits/withdrawals
- Monitor for spike in `Pending` or `Rejected` events
- Alert on revert rates in `depositTransaction()` / `initiateWithdrawal()`

#### Recovery Path

- On L1 (`OptimismPortal2`): governance must execute a proxy upgrade or reinitialization to set `compliance` to `address(0)`. There is no direct `setCompliance` setter on `OptimismPortal2`
- On L2 (`L2ToL1MessagePasser`): the `L2ProxyAdmin` owner calls `setCompliance(address(0))` to disable the compliance module
- Recovery requires governance action on both L1 and L2 to maintain stage 1 requirements. This prevents the compliance contract from being changed arbitrarily outside of a security council action

#### Action Items

- [ ] Test the `address(0)` disable path end-to-end
- [ ] Establish runbook for emergency compliance module disable

</details>

<details>
<summary>FM3: Buggy IRule Implementation</summary>

#### Description

The `IRule` interface allows external code to be called during the compliance check. A rule contract could: return unexpected values (e.g., `Refunded` status which is invalid for rules), revert unexpectedly, consume excessive gas causing out-of-gas errors, or contain buggy logic. Since rules are iterated in `check()`, a single bad rule can break the entire compliance flow.

#### Risk Assessment

Medium severity / Low likelihood

The contract owner (an aligned chain operator) controls which rules are configured via `addRule/removeRule`. A malicious rule implementation is out of scope because the owner would be denial-of-servicing their own chain. If a buggy rule does cause issues (e.g., excessive gas consumption or reverts), the owner can simply remove the rule via `removeRule` or disable the compliance module entirely via `setCompliance(address(0))`.

#### Mitigations

- The `Compliance` contract validates that rule return values are only `Approved`, `Pending`, or `Rejected` (reverts if `Refunded` is returned)
- Only the owner (an aligned actor) can add/remove rules via `addRule/removeRule`
- Audit rule implementations before deployment

#### Detection

- Monitor for reverts in `check()` originating from rule calls

#### Recovery Path

- Operator removes the faulty rule via `removeRule`
- If the rule causes `check()` to always revert, fall back to disabling compliance via `setCompliance(address(0))`

#### Action Items

- [ ] Define a rule validation/audit process before rules are added

</details>

<details>
<summary>FM4: ETH Locked in Compliance Contract</summary>

#### Description

When a deposit or withdrawal is flagged, the ETH value is held by the `Compliance` contract. Bugs in the `settle()` or refund logic could prevent users from recovering their ETH, either by reverting on valid refund claims or by failing to forward ETH to the bridge on approval. This is especially critical on L1 where real ETH is at stake.

#### Risk Assessment

Critical severity / Low likelihood

#### Mitigations

- Thorough testing of all `settle()` paths: approved → forward to bridge, rejected → enable refund, pending → no-op
- Test refund claim path independently
- Invariant testing: ETH balance of Compliance contract should equal sum of all pending/rejected transaction values

#### Detection

- Monitor ETH balance of Compliance contracts
- Alert if Compliance contract balance grows unexpectedly or if refund transactions revert

#### Recovery Path

- If a specific refund path is broken: contract upgrade to fix the bug
- If `settle()` is broken: contract upgrade required
- ETH is not lost permanently as long as the contract is upgradeable

#### Action Items

- [ ] Invariant tests for ETH accounting in the Compliance contract
- [ ] Ensure Compliance contract is deployed behind an upgradeable proxy

</details>

<details>
<summary>FM5: Nonce Reservation Issues for Withdrawals</summary>

#### Description

On L2, the withdrawal nonce is reserved in `initiateWithdrawal()` before the compliance check. If nonce accounting is incorrect — e.g., nonce is incremented but the withdrawal is later approved with a different nonce, or the same nonce is reused — withdrawals could become unprocessable on L1 (proving/finalizing fails) or replayable.

#### Risk Assessment

High severity / Low likelihood

#### Mitigations

- The nonce is reserved at the start of `initiateWithdrawal()` and passed to `check()`, which includes it in the hash stored in the status mapping
- When the withdrawal is later approved, the `approved()` function on `L2ToL1MessagePasser` uses the exact reserved nonce
- Careful testing of the nonce lifecycle: reservation → hold → approval → prove → finalize

#### Detection

- Monitor for withdrawal proving failures that correlate with previously-flagged withdrawals
- Compare nonce sequences for gaps or duplicates

#### Recovery Path

- If nonce accounting is fundamentally broken: contract upgrade on L2
- Individual stuck withdrawals may require manual intervention or upgrade

#### Action Items

- [ ] End-to-end test of the full flagged withdrawal lifecycle including L1 proving and finalization
- [ ] Test nonce consistency between reservation and approval

</details>

<details>
<summary>FM6: Owner Key Loss</summary>

#### Description

If the owner key is lost, pending transactions cannot be approved or rejected via owner override. The `settle()` function can still re-evaluate rules (when the owner-override bit is not set), but transactions that require explicit owner approval will be stuck indefinitely. The design doc explicitly calls this out as a single point of failure.

#### Risk Assessment

Medium severity / Low likelihood

#### Mitigations

- Key management practices: multisig, key backup procedures
- Users can still claim refunds for rejected transactions, so ETH is not permanently stuck
- The `settle()` function can still resolve transactions if rules change (rules are re-evaluated when the owner-override bit is not set)

#### Detection

- Alert if owner has not acted on pending transactions within a configurable time window

#### Recovery Path

- The `Compliance` contract inherits Solady's `Ownable`. If the owner key is lost, governance recovers by upgrading the `Compliance` proxy to set a new owner
- For pending transactions: once a new owner is set via proxy upgrade, they can approve/reject the backlog

#### Action Items

- [ ] Document owner key management requirements
- [ ] Define SLA for owner response time to pending transactions

</details>

<details>
<summary>FM7: Access Control Misconfiguration</summary>

#### Description

The `setCompliance`, `transferOwnership`, and `addRule/removeRule` functions control critical security parameters. If access control is misconfigured — e.g., these functions are callable by unauthorized parties — an attacker could disable compliance entirely, install a malicious owner, or add malicious rules.

#### Risk Assessment

Critical severity / Low likelihood

#### Mitigations

- On `OptimismPortal2`, the compliance address is set via `initialize()` — changing it requires a governance upgrade through the L1 proxy admin
- On `L2ToL1MessagePasser`, `setCompliance` is callable only by the `L2ProxyAdmin` owner
- `transferOwnership` and `addRule/removeRule` on the Compliance contract are gated to the owner (via `Ownable`)
- Auth checks verified through unit tests

#### Detection

- Monitor for unexpected calls to `setCompliance`, `transferOwnership`, `addRule/removeRule`
- Alert on any changes to these values

#### Recovery Path

- If compliance is disabled by an unauthorized party: governance action to re-enable
- If malicious owner/rules are set: governance upgrade to restore correct configuration
- Requires governance vote or proxy admin action depending on the access control design

#### Action Items

- [ ] Define and document the access control model for all admin functions
- [ ] Unit tests verifying auth checks on `setCompliance`, `transferOwnership`, `addRule/removeRule`

</details>

<details>
<summary>FM8: Reentrancy via External Rule Calls</summary>

#### Description

The `check()` function calls external `IRule` contracts which could theoretically attempt to re-enter the `Compliance` contract, `OptimismPortal2`, or `L2ToL1MessagePasser`.

#### Risk Assessment

Low severity / Very low likelihood

The contract owner (an aligned chain operator) controls which rules are configured via `addRule/removeRule`. Since only the owner can add rules, a malicious rule designed to exploit reentrancy is out of scope — the owner would be attacking their own chain. If they did, they could simply remove the rule. Reentrancy guards are still applied to `settle()` and the bridge's `approved()` callback as defense in depth, but reentrancy through `IRule` calls is not considered a realistic threat vector.

#### Mitigations

- Only the owner (an aligned actor) can add rules, so malicious rule implementations are out of scope
- Reentrancy guards on `settle()` and the bridge `approved()` callback as defense in depth
- Follow checks-effects-interactions pattern for state updates

#### Detection

- Static analysis tools (Slither, etc.) flag reentrancy vectors
- Audit review of the `settle()` and `approved()` callback paths

#### Recovery Path

- If a buggy rule causes unexpected behavior: owner removes the rule via `removeRule`
- Contract upgrade if needed

#### Action Items

- [ ] Ensure reentrancy guards are applied to `settle()` and bridge `approved()` callback
- [ ] Static analysis scan for reentrancy vectors

</details>

<details>
<summary>FM9: Hash Collision in Status Mapping</summary>

#### Description

The `_status` mapping uses `keccak256(abi.encode(...))` of the transaction parameters as the key. If two different transactions produce the same hash, one transaction's status could be confused with another's — e.g., an attacker's flagged transaction could be resolved by settling a different transaction with the same hash.

#### Risk Assessment

Low severity / Very low likelihood (keccak256 collision resistance)

#### Mitigations

- Use of `abi.encode` (not `abi.encodePacked`) ensures unambiguous encoding and avoids collisions from variable-length fields
- keccak256 provides 256-bit collision resistance, making intentional collisions computationally infeasible

#### Detection

- Not practically detectable since collision would require breaking keccak256

#### Recovery Path

- No recovery needed — this is a theoretical concern only
- If keccak256 were ever broken, a contract upgrade to use a different hash function would be required

</details>

<details>
<summary>FM10: Interaction with Dispute Game / Withdrawal Proving</summary>

#### Description

Flagged withdrawals that are later approved interact with the dispute game timing and withdrawal proving flow. A withdrawal's nonce is reserved at flag time, but the withdrawal message is only emitted when `settle()` approves it (possibly much later). This creates a timing gap where the withdrawal proof's inclusion in an output root may not align with expectations.

Analysis of the flagged withdrawal lifecycle (see [design doc Risks section](../protocol/compliance-module.md#risks--uncertainties) for full details):

1. **Flag time:** Nonce N is reserved in `initiateWithdrawal()`. The nonce counter advances to N+1 for subsequent withdrawals. No `MessagePassed` event is emitted and no message hash is stored in `sentMessages`.
2. **Hold period:** The withdrawal is held in the Compliance contract. Other withdrawals (with nonces N+1, N+2, ...) proceed normally and are included in output roots.
3. **Settle time:** `settle()` calls `approved()`, which emits the `MessagePassed` event with nonce N and stores the message hash in `sentMessages`. The withdrawal is now part of L2 state.
4. **Output root inclusion:** The withdrawal's message hash is included in the next output root proposed after the settle transaction. Any output root proposed before settle will NOT contain this withdrawal.
5. **Prove and finalize:** The user proves the withdrawal against an output root from step 4 (or later), then finalizes after the dispute game window.

The total withdrawal time is extended by the hold duration (step 2). The dispute game window begins when the output root containing the settled withdrawal is proposed, not when the withdrawal was originally initiated. Nonce ordering is preserved — nonce N is deterministic from flag time — so the withdrawal hash computed on L1 during proving matches the one emitted at settle time.

#### Risk Assessment

Medium severity / Low likelihood (the interaction is well-understood and nonce ordering is preserved)

#### Mitigations

- The withdrawal message hash uses the reserved nonce, ensuring consistency between flag-time and settle-time
- Withdrawal proving on L1 depends on the output root containing the withdrawal, which only happens after `settle()` emits the message — this is deterministic and correct by construction
- Nonce ordering is preserved: nonce N is assigned at flag time and reused at settle time, so the L1 proof matches

#### Detection

- Monitor for failed withdrawal proofs that correspond to previously-flagged withdrawals
- Monitor for withdrawals stuck in the proving/finalization pipeline

#### Recovery Path

- If a flagged withdrawal's proof fails on L1, the root cause is likely a nonce accounting bug in the Compliance contract or `L2ToL1MessagePasser.approved()` — this would require a contract upgrade
- Individual stuck withdrawals may require manual intervention or upgrade

#### Action Items

- [ ] Test the full end-to-end flow: flag → hold → settle(approve) → prove → finalize
- [ ] Test nonce consistency between reservation and approval across concurrent withdrawals

</details>

<details>
<summary>FM11: Generic Smart Contract Failure Modes</summary>

#### Description

The Compliance contract is a new contract deployed behind a proxy on both L1 and L2. All generic smart contract upgrade failure modes apply. See [generic smart contract failure modes](./fma-generic-contracts.md) for reference.

Applicable failure modes:
- **Proxy Initialization Failure:** The Compliance contract must be correctly initialized with `owner`, `bridge`, and initial rules. A failed initialization could leave the contract in an unusable state or allow arbitrary reinitialization.
- **Storage Layout Corruption:** New implementations must not overwrite existing storage slots. The `_status` mapping, `EnumerableSetLib.AddressSet` for rules, Solady `Ownable`'s owner slot, and `bridge` variable must be preserved across upgrades.
- **Wrong Initialization Values:** Incorrect `bridge` address would break the caller check in `check()`. Incorrect `owner` address would give the wrong party control.
- **Can't Upgrade Implementation:** If the proxy admin is misconfigured, the Compliance contract cannot be upgraded to fix bugs or recover stuck ETH.
- **Reinitialization:** If `initialize()` is callable again, an attacker could reset the owner, bridge, and rules.
- **ABI Compatibility:** OptimismPortal2 and L2ToL1MessagePasser call into the Compliance contract — ABI changes could break the integration.

#### Risk Assessment

High severity / Low likelihood (standard proxy risks, well-understood mitigations)

#### Mitigations

- Follow standard proxy patterns (TransparentUpgradeableProxy or similar)
- Storage layout snapshot tests
- Initialization tests (`test_initialize_succeeds`)
- Verify `initializer` modifier prevents reinitialization

#### Detection

- Fork tests verifying initialization state
- Simulate upgrades before execution
- ABI snapshot comparisons

#### Recovery Path

- Most issues recoverable via contract upgrade
- If proxy admin is lost: requires governance intervention

</details>

- [x] Check this box to confirm that generic smart contract failure modes have been considered and incorporated above (FM11).

## Risks & Uncertainties

- The exact set of `IRule` implementations needed at launch is still an open question. Rule implementations are a significant part of the attack surface.
- Interaction between the compliance module and the dispute game / withdrawal proving flow needs further analysis (see FM10).
- Governance and access control model has been defined: `setCompliance` on `OptimismPortal2` is set via initializer (governance-gated), `setCompliance` on `L2ToL1MessagePasser` is gated to `L2ProxyAdmin` owner, and `addRule`/`removeRule`/`transferOwnership` on `Compliance` are gated to the owner (via `Ownable`).
- The compliance module holds user ETH in custody, making it a high-value target. The amount of ETH at risk scales with the number of flagged transactions.

## Audit Requirements

**This component requires audit.**

The Compliance module introduces a new contract that:
1. Holds user ETH in custody (flagged deposits/withdrawals)
2. Makes external calls to arbitrary `IRule` contracts
3. Modifies the critical path of `OptimismPortal2.depositTransaction()` and `L2ToL1MessagePasser.initiateWithdrawal()`
4. Introduces an owner role (via `Ownable`) with significant power

All of these factors warrant a thorough security audit before deployment.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] Define owner key management requirements — multisig, HSM, rotation policy (Assignee: chain operations)
- [x] Define and document the access control model for `setCompliance`, `transferOwnership`, `addRule/removeRule` (Assignee: design author)
- [x] Conduct detailed analysis of flagged withdrawal interaction with dispute game / proving flow (Assignee: design author)
- [ ] End-to-end test of the full flagged withdrawal lifecycle: flag → hold → settle → prove → finalize (Assignee: implementation team)
- [ ] Invariant tests for ETH accounting in the Compliance contract (Assignee: implementation team)
- [ ] Ensure Compliance contract is deployed behind an upgradeable proxy (Assignee: implementation team)
- [ ] Implement monitoring and alerting for owner actions, compliance events, and ETH balances (Assignee: chain operations)
- [ ] Reentrancy guards on `settle()` and bridge `approved()` callback, plus static analysis scan (Assignee: implementation team)
- [ ] Define a rule validation/audit process before rules are added to production (Assignee: security team)
- [ ] Storage layout snapshot tests and initialization tests for the Compliance contract (Assignee: implementation team)
- [ ] Security audit of Compliance contract, IRule interface, and modifications to OptimismPortal2 and L2ToL1MessagePasser (Assignee: security team)
