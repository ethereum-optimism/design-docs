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

This document covers failure modes for the **Compliance Module**, an optional on-chain
compliance screening layer for L1 deposits via `OptimismPortal2`. The module exposes a single
`ICompliance` interface (`check(from, to) returns (bool)`) and ships a single concrete
implementation, `ChainalysisCompliance`, which wraps the Chainalysis Sanctions Oracle. The
contract is deployed behind an upgradeable proxy on L1.

This iteration is L1-only. L2 → L1 withdrawals are not screened; the failure-mode analysis below
is scoped to deposits.

`ChainalysisCompliance` has no admin role and no ETH-egress function. ETH `msg.value` that
arrives with a screened-out deposit (sanctioned or oracle-unavailable) is permanently locked in
the contract. There is no `recover`, `settle`, `refund`, `withdraw`, owner override, or any other
mechanism that releases locked funds. The contract's only externally callable entry point is
`check`, gated to the bridge. The proxy admin (governance) retains the ability to upgrade the
implementation.

**References:**

- [Compliance Module Design Doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/compliance-module.md)

## Failure Modes

<details>
<summary>FM1: Compliance Contract Bug Blocks Deposits</summary>

#### Description

A bug in `check()` — for example, an unconditional revert, a misconfigured `bridge` address
(failing the caller check), or a mistake in the `try/catch` wrapping of the oracle calls — could
cause every deposit to revert. Because the compliance check is on the `depositTransaction` hot
path, this would halt all L1 → L2 deposits for the affected chain.

A milder failure mode is `check()` returning `false` for compliant addresses (false positives
caused by an implementation bug rather than a bad oracle response), which does not halt
deposits but causes legitimate deposits to be permanently locked. See FM2 for the locked-ETH
consequences.

#### Risk Assessment

High severity / Medium likelihood (new code in the deposit hot path)

#### Mitigations

- Setting `OptimismPortal2.compliance` to `address(0)` disables the module entirely. This
  requires a governance-gated proxy upgrade or reinitialization of `OptimismPortal2`. Stage 1
  requirements are preserved.
- Thorough testing of `check()` with the oracle in each of: returns `false`, returns `true`,
  reverts, returns malformed data
- Fork tests against the actual Chainalysis Sanctions Oracle on Ethereum mainnet

#### Detection

- Monitor for a sudden drop in successful deposits
- Monitor revert rates in `depositTransaction()`
- Monitor the rate of `Sanctioned` and `OracleUnavailable` events for unexpected spikes

#### Recovery Path

- Governance executes a proxy upgrade or reinitialization to set `OptimismPortal2.compliance`
  to `address(0)`. ETH locked by the buggy `check()` before the disable is **not** recoverable
  — the contract has no egress function. A governance-introduced egress function via proxy
  upgrade would release locked funds, but doing so eliminates the property that makes the
  contract safe to leave unattended (see FM2).

#### Action Items

- [ ] Test the `address(0)` disable path end-to-end on L1
- [ ] Establish a runbook for emergency compliance module disable
- [ ] Fork tests against the live Chainalysis Sanctions Oracle on Ethereum mainnet

</details>

<details>
<summary>FM2: Compliant ETH Permanently Locked by Buggy `check()` or Bad Oracle</summary>

#### Description

The screened-out path of `check()` retains `msg.value` and never releases it. By design, this
applies to genuinely sanctioned addresses and to genuine oracle outages — that is the
deliberate behaviour of the contract.

The failure-mode-flavoured version of this is when the screened-out path is hit by a
*compliant* user. There are two ways this can happen:

1. **Bug in `check()`**. An incorrect oracle address, a misuse of `try/catch`, an off-by-one
   in the `from` / `to` arguments, or any other logic error that returns `false` on a
   compliant transaction. The user's ETH is locked.
2. **False positive from the oracle**. The Chainalysis Sanctions Oracle (or whichever oracle
   the deployment configures) flags an address that should not have been flagged. The
   contract correctly emits `Sanctioned`; the user's ETH is locked.

Both cases produce the same outcome: a compliant user's deposit `msg.value` is permanently
locked in the compliance contract with no on-chain remediation. The user's only recourse is
off-chain (complaint to the oracle provider, governance proposal, etc.). This is the central
trade-off of the no-recovery design and is the most user-visible failure mode of the module.

#### Risk Assessment

High severity / Medium likelihood (oracle false positives are routine; `check()` bugs are a
new-code risk)

#### Mitigations

- Conservative review and audit of every line of `check()`; this is the single function whose
  bugs cause permanent user loss
- Fork tests of `check()` against the live oracle on Ethereum mainnet, including known
  sanctioned addresses (true positives), random clean addresses (true negatives), addresses at
  the transition points of the oracle's flagging (boundary cases)
- Operators should publish clear UX guidance to users that screened-out deposits result in
  irreversible loss of `msg.value`, so users do not accidentally deposit after being flagged
- Operators may choose to deploy with `OptimismPortal2.compliance = address(0)` for an initial
  period to collect telemetry on what `check()` *would* have returned, before flipping the
  bridge to trust those decisions

#### Detection

- Cross-reference `Sanctioned` events against external sanctions lists for false positives
- Operators should monitor user complaints (Discord, support channels) for reports of
  unexpected deposit "successes" with no L2 effect
- Track the cumulative ETH locked in the compliance contract; an unexplained jump or a
  mismatch with the sum of `Sanctioned.value + OracleUnavailable.value` indicates a bug
  somewhere in the path

#### Recovery Path

- **There is no on-chain recovery.** Locked ETH stays locked.
- A proxy upgrade can introduce an egress function, but doing so changes the contract's trust
  model fundamentally: anyone with upgrade authority gains the ability to release every
  historical locked deposit, including ones from genuinely sanctioned users. Operators should
  treat such a proposal as an exceptional governance event, not routine remediation.
- Off-chain remediation (e.g., the operator credits the affected user out-of-band) is
  possible but is outside the scope of the on-chain protocol.

#### Action Items

- [ ] Audit `check()` with explicit attention to the locked-funds blast radius
- [ ] Define the operational decision criteria for whether and when to introduce an egress
      function via governance upgrade
- [ ] Document the user-visible loss in SDK error messages and operator-facing guidance
- [ ] Decide whether the v1 launch should ship with an alternative `msg.value` disposition
      (auto-bounce-to-sender, burn-to-zero) — both alternatives are documented in the design
      doc's Alternatives section

</details>

<details>
<summary>FM3: Access Control Misconfiguration</summary>

#### Description

The compliance module's access control surface is small in this design: the `compliance`
address on `OptimismPortal2`, and the proxy admin authority over `ChainalysisCompliance`
itself. There is no owner role on `ChainalysisCompliance`, no `transferOwnership`, no
`recover`, and no rule-management functions.

There is no `setCompliance` setter on `OptimismPortal2`; the compliance address is set via
`initialize()` and changed only by governance proxy action.

If access control is misconfigured — e.g. the proxy admin authority on `OptimismPortal2` or
`ChainalysisCompliance` is misassigned — an attacker could point the bridge at a malicious
`ICompliance` implementation or upgrade `ChainalysisCompliance` to a version that introduces
an ETH-egress function and drains historical locked funds.

#### Risk Assessment

Critical severity / Low likelihood

#### Mitigations

- The `OptimismPortal2.compliance` address is set via `initialize()`, governance-gated through
  the L1 proxy admin
- The proxy admin for `ChainalysisCompliance` should be governance, not an EOA
- Auth checks verified through unit tests
- Storage-layout snapshot tests catch upgrades that would introduce a new state-mutating
  function in an unexpected slot

#### Detection

- Alert on any change to the `compliance` storage slot on `OptimismPortal2`
- Alert on any upgrade of the `OptimismPortal2` or `ChainalysisCompliance` proxy implementation

#### Recovery Path

- If `OptimismPortal2.compliance` is changed by an unauthorized party: governance action to
  restore the correct address. ETH already passed to a malicious implementation is not
  recoverable.
- If the `ChainalysisCompliance` proxy is upgraded to a malicious implementation: governance
  action to revert the upgrade. ETH already moved by the malicious implementation is not
  recoverable.

#### Action Items

- [ ] Define and document the access control model for the `OptimismPortal2` and
      `ChainalysisCompliance` proxy admins
- [ ] Unit tests verifying auth checks on the proxy admin gate
- [ ] Confirm that the proxy admin authority for `ChainalysisCompliance` matches governance
      expectations

</details>

<details>
<summary>FM4: Chainalysis Sanctions Oracle Dependency</summary>

#### Description

`ChainalysisCompliance` calls the Chainalysis Sanctions Oracle on every deposit. The oracle is a
third-party dependency outside the OP Stack governance perimeter. Failure modes include:

- **Oracle unreachable** — calls revert (e.g., contract self-destructed, paused, or address
  changed). `check()` catches this and emits `OracleUnavailable`. The associated ETH is
  permanently locked.
- **Oracle returns malformed data** — caught the same way; locked the same way.
- **Oracle returns false negatives** — sanctioned addresses pass the check. Not detectable
  on-chain; the chain operator inherits the false-negative rate of the Chainalysis list.
- **Oracle returns false positives** — legitimate addresses are flagged. ETH is permanently
  locked. See FM2 for the user-impact analysis.
- **Sanctions list update lag** — Chainalysis updates the on-chain list on a delay relative
  to OFAC announcements. A user that becomes sanctioned during the lag window can still
  deposit.
- **Oracle compromise** — if the Chainalysis oracle owner is compromised and an attacker can
  flip the list arbitrarily, the chain inherits whatever the attacker writes. False
  positives at scale would result in mass permanent loss.

The interaction between this dependency and the no-recovery property of the contract is the
defining risk profile of the module: routine oracle failures (outages, false positives)
translate directly into routine permanent user loss. This is significantly more punitive than
a held-and-recoverable design and should be a primary topic of audit and operator review.

#### Risk Assessment

High severity / Medium likelihood (oracle outages and update lag are routine; oracle
compromise is rare but high-impact)

#### Mitigations

- `check()` wraps both `isSanctioned(from)` and `isSanctioned(to)` calls in `try/catch` so a
  reverting or invalid-returndata oracle does not propagate a revert to the bridge — instead
  the deposit is screened out under `OracleUnavailable`
- Governance-gated proxy reinitialization of `OptimismPortal2` to set `compliance` to
  `address(0)` provides an escape hatch if the oracle is determined to be untrustworthy. This
  stops new losses but does not unlock historical ones.
- Operators should monitor Chainalysis status feeds and OFAC announcement channels
- The oracle address is set at initialization and changing it requires governance action;
  swapping to a different oracle implementation is a deliberate decision, not an attacker
  capability
- Operators may choose to gate the bridge launch with `OptimismPortal2.compliance = address(0)`
  and a shadow-mode deployment first, to surface the false-positive and outage rate before
  flipping the bridge to act on the oracle's verdict

#### Detection

- Monitor the ratio of `Allowed` / `Sanctioned` / `OracleUnavailable` events for anomalies
- Cross-reference `Sanctioned` event addresses against external sanctions lists for false
  positives
- Alert on sustained `OracleUnavailable` events suggesting an oracle outage

#### Recovery Path

- Oracle outage: there is no on-chain recovery for ETH already locked by `OracleUnavailable`.
  Going forward, the operator can disable compliance until the oracle is healthy.
- Oracle compromise: governance disables compliance via a proxy reinitialization of
  `OptimismPortal2` and/or upgrades the `ChainalysisCompliance` proxy to point at a different
  oracle. Historical losses remain locked.

#### Action Items

- [ ] Document operational runbook for oracle outage and oracle compromise scenarios
- [ ] Implement monitoring on the rate of `OracleUnavailable` events and the
      `Sanctioned` / `Allowed` ratio
- [ ] Decide whether shadow-mode deployment is required before live operation

</details>

<details>
<summary>FM5: Generic Smart Contract Failure Modes</summary>

#### Description

`ChainalysisCompliance` is a new contract deployed behind an upgradeable proxy on L1, and
`OptimismPortal2` is modified to add the `compliance` configuration variable. All generic
smart contract upgrade failure modes apply. See [generic smart contract failure
modes](./fma-generic-contracts.md) for reference.

Applicable failure modes:

- **Proxy Initialization Failure:** the contract must be correctly initialized with `bridge`
  and `sanctionsOracle`. A failed initialization could leave the contract unusable or allow
  arbitrary reinitialization.
- **Storage Layout Corruption:** new implementations must not overwrite the existing layout
  (`bridge`, `sanctionsOracle`, `_nextId`).
- **Wrong Initialization Values:** an incorrect `bridge` address breaks the caller check in
  `check()`. An incorrect `sanctionsOracle` causes false positives or negatives.
- **Can't Upgrade Implementation:** if the proxy admin is misconfigured, the contract cannot
  be upgraded to fix bugs. Bugs that lock compliant ETH (FM2) become irrecoverable in
  practice.
- **Reinitialization:** if `initialize()` is callable again, an attacker could reset the
  bridge and oracle.
- **ABI Compatibility:** `OptimismPortal2` calls into the compliance contract — ABI changes to
  `check(address,address)` would break the integration. The `check(address,address)` selector
  is therefore stable forever; richer screening is added via additional overloads, never by
  changing this one.
- **Upgrade introducing an egress function:** a governance proxy upgrade could add a function
  that drains the contract's locked balance. This is intentional capability — it is the only
  remediation path for FM2-class losses — but it is also a single governance action that
  controls every historical locked deposit. It should be treated as a high-scrutiny upgrade,
  not routine.

#### Risk Assessment

High severity / Low likelihood (standard proxy risks, well-understood mitigations; the
egress-via-upgrade case is unique to this design)

#### Mitigations

- Standard proxy patterns (TransparentUpgradeableProxy or similar)
- Storage layout snapshot tests
- Initialization tests (`test_initialize_succeeds`)
- `initializer` modifier prevents reinitialization
- Treat the `check(address,address)` selector as a permanent ABI commitment
- Treat any proposed upgrade that adds ETH-egress capability as an exceptional governance
  event with explicit user notification

#### Detection

- Fork tests verifying initialization state
- Simulate upgrades before execution
- ABI snapshot comparisons
- Governance review process flags any upgrade introducing new state-mutating functions

#### Recovery Path

- Most issues recoverable via contract upgrade
- If proxy admin is lost: requires governance intervention

</details>

- [x] Check this box to confirm that generic smart contract failure modes have been considered
      and incorporated above (FM5).

## Risks & Uncertainties

- The Chainalysis Sanctions Oracle is a third-party dependency on the deposit hot path. Its
  availability, integrity, and update cadence directly affect deposit UX when compliance is
  enabled, and unlike the earlier draft, oracle failures translate into permanent user loss
  rather than a recoverable hold (see FM2 and FM4).
- There is no on-chain remediation for screened-out deposits. The contract is intentionally
  designed without an admin role or egress function. The trade-off is contract simplicity and
  the elimination of an admin-key compromise vector at the cost of harsher user impact on
  false positives and oracle outages. Operators must decide whether this trade-off is
  appropriate for their user base; the design doc's Alternatives section documents
  auto-bounce-to-sender and burn-to-zero as drop-in alternatives.
- The proxy admin retains the theoretical ability to introduce an egress function via upgrade.
  This is the only remediation path for FM2-class losses. Governance practices around such an
  upgrade should be defined ahead of time so the question is not litigated under pressure.
- L2 → L1 withdrawals are not screened in this iteration. Chain operators that need
  withdrawal-side sanctions screening must address it off-chain or wait for a later iteration
  that introduces an L2 module.

## Audit Requirements

**This component requires audit.**

The compliance module introduces a new contract that:

1. Sits in the critical path of `OptimismPortal2.depositTransaction()`
2. Permanently locks user ETH on screened-out deposits, with no on-chain remediation
3. Makes external calls to a third-party oracle on every deposit

The audit should pay particular attention to `check()` — this is the only function in the
contract that touches user funds, and bugs in it directly cause irreversible user loss.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes
occurring, and to ensure they can be detected:

- [ ] Resolve all comments on this document and incorporate them into the document itself
      (Assignee: document author)
- [x] Define and document the access control model for the `OptimismPortal2` and
      `ChainalysisCompliance` proxy admins (Assignee: design author)
- [ ] Audit `check()` with explicit attention to the locked-funds blast radius (Assignee:
      security team)
- [ ] Define the operational decision criteria for whether and when to introduce an egress
      function via governance upgrade (Assignee: chain operations + governance)
- [ ] Decide whether the v1 launch should ship with an alternative `msg.value` disposition
      (auto-bounce-to-sender, burn-to-zero) — both alternatives are documented in the design
      doc's Alternatives section (Assignee: design author)
- [ ] Decide whether shadow-mode deployment is required before live operation (Assignee:
      chain operations)
- [ ] Implement monitoring for the cumulative locked balance and reconcile against the sum of
      `Sanctioned.value + OracleUnavailable.value` events (Assignee: chain operations)
- [ ] Implement monitoring for the rate of `OracleUnavailable` events and the
      `Sanctioned` / `Allowed` ratio (Assignee: chain operations)
- [ ] Storage layout snapshot tests and initialization tests (Assignee: implementation team)
- [ ] Fork tests of `check()` against the live Chainalysis Sanctions Oracle on Ethereum
      mainnet (Assignee: implementation team)
- [ ] Operational runbook for emergency compliance module disable (proxy reinitialization)
      (Assignee: chain operations)
- [ ] Operational runbook for oracle outage and oracle compromise scenarios (Assignee: chain
      operations)
- [ ] Update SDK/tooling documentation to describe the screened-out deposit user experience,
      including the irreversibility of the loss (Assignee: ecosystem)
- [ ] Security audit of `ChainalysisCompliance`, the `ICompliance` interface, and modifications
      to `OptimismPortal2` (Assignee: security team)
