# Incident Response Improvements: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Incident Response Improvements: Failure Modes and Recovery Path Analysis](#incident-response-improvements-failure-modes-and-recovery-path-analysis)
  - [Introduction](#introduction)
  - [DelayedWETH Failure Modes](#delayedweth-failure-modes)
    - [FM1: DelayedWETH Initialization Failure](#fm1-delayedweth-initialization-failure)
      - [**Description:**](#description)
      - [**Risk Assessment:**](#risk-assessment)
      - [**Mitigations:**](#mitigations)
      - [**Detection:**](#detection)
      - [**Recovery Path(s):**](#recovery-paths)
      - [Action items:](#action-items)
    - [FM2: DelayedWETH Legacy Version Confusion](#fm2-delayedweth-legacy-version-confusion)
      - [**Description:**](#description-1)
      - [**Risk Assessment:**](#risk-assessment-1)
      - [**Mitigations:**](#mitigations-1)
      - [**Detection:**](#detection-1)
      - [**Recovery Path(s):**](#recovery-paths-1)
      - [Action items:](#action-items-1)
  - [OptimismPortal Failure Modes](#optimismportal-failure-modes)
    - [FM3: Storage Layout Risks in OptimismPortal2](#fm3-storage-layout-risks-in-optimismportal2)
      - [**Description:**](#description-2)
      - [**Risk Assessment:**](#risk-assessment-2)
      - [**Mitigations:**](#mitigations-2)
      - [**Detection:**](#detection-2)
      - [**Recovery Path(s):**](#recovery-paths-2)
      - [Action items:](#action-items-2)
    - [FM4: Game that resolved incorrectly is not blacklisted within airgap delay period](#fm4-game-that-resolved-incorrectly-is-not-blacklisted-within-airgap-delay-period)
      - [**Description:**](#description-3)
      - [**Risk Assessment:**](#risk-assessment-3)
      - [**Mitigations:**](#mitigations-3)
      - [**Detection:**](#detection-3)
      - [**Recovery Path(s):**](#recovery-paths-3)
      - [Action items:](#action-items-3)
  - [AnchorStateRegistry Failure Modes](#anchorstateregistry-failure-modes)
    - [FM5: AnchorStateRegistry Contract Address Mismatch](#fm5-anchorstateregistry-contract-address-mismatch)
      - [**Description:**](#description-4)
      - [**Risk Assessment:**](#risk-assessment-4)
      - [**Mitigations:**](#mitigations-4)
      - [**Detection:**](#detection-4)
      - [**Recovery Path(s):**](#recovery-paths-4)
      - [Action items:](#action-items-4)
    - [FM6: Game Retirement Timestamp Updated Too Frequently or Anchor State Not Updated](#fm6-game-retirement-timestamp-updated-too-frequently-or-anchor-state-not-updated)
      - [**Description:**](#description-5)
      - [**Risk Assessment:**](#risk-assessment-5)
      - [**Mitigations:**](#mitigations-5)
      - [**Detection:**](#detection-5)
      - [**Recovery Path(s):**](#recovery-paths-5)
      - [Action items:](#action-items-5)
  - [FaultDisputeGame Failure Modes](#faultdisputegame-failure-modes)
    - [FM7: FaultDisputeGame Property Reporting](#fm7-faultdisputegame-property-reporting)
      - [**Description:**](#description-6)
      - [**Risk Assessment:**](#risk-assessment-6)
      - [**Mitigations:**](#mitigations-6)
      - [**Detection:**](#detection-6)
      - [**Recovery Path(s):**](#recovery-paths-6)
      - [Action items:\*\*](#action-items-6)
    - [FM8: Bond Distribution Errors](#fm8-bond-distribution-errors)
      - [**Description:**](#description-7)
      - [**Risk Assessment:**](#risk-assessment-7)
      - [**Mitigations:**](#mitigations-7)
      - [**Detection:**](#detection-7)
      - [**Recovery Path(s):**](#recovery-paths-7)
      - [Action items:](#action-items-7)
    - [FM9: Game Participation Disincentivization](#fm9-game-participation-disincentivization)
      - [**Description:**](#description-8)
      - [**Risk Assessment:**](#risk-assessment-8)
      - [**Mitigations:**](#mitigations-8)
      - [**Detection:**](#detection-8)
      - [**Recovery Path(s):**](#recovery-paths-8)
      - [Action items:](#action-items-8)
  - [Cross-Contract Failure Modes](#cross-contract-failure-modes)
    - [FM10: Contract Initialization State Corruption](#fm10-contract-initialization-state-corruption)
      - [**Description:**](#description-9)
      - [**Risk Assessment:**](#risk-assessment-9)
      - [**Mitigations:**](#mitigations-9)
      - [**Detection:**](#detection-9)
      - [**Recovery Path(s):**](#recovery-paths-9)
      - [Action items:](#action-items-9)
    - [FM11: System-Wide Upgrade Coordination](#fm11-system-wide-upgrade-coordination)
      - [**Description:**](#description-10)
      - [**Risk Assessment:**](#risk-assessment-10)
      - [**Mitigations:**](#mitigations-10)
      - [**Detection:**](#detection-10)
      - [**Recovery Path(s):**](#recovery-paths-10)
      - [Action items:](#action-items-10)
    - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
  - [Action Items](#action-items-11)
  - [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Italics are used to indicate things that need to be replaced._

|                    |                |
| ------------------ | -------------- |
| Author             | wildmolasses   |
| Created at         | 2025-01-22     |
| Initial Reviewers  | smartcontracts |
| Need Approval From |                |
| Status             | Draft          |

## Introduction

This document covers the fault dispute game incident response improvements project. The project makes modifications to several key contracts to improve incident response capabilities:

1. DelayedWETH: Simplified hold functionality for emergency response
2. OptimismPortal: Improved game type management and withdrawal validation
3. AnchorStateRegistry: Unified anchor state and improved validation
4. FaultDisputeGame: Added bond refunding for invalidated games

Below are references for this project:

- Design Docs
  - [Anchor State Poisoning Protection](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/anchor-state-poison-protection.md)
  - [DelayedWETH Expansion](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/delayed-weth-expansion.md)
  - [FDG Refund Mode](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/fdg-refund-mode.md)
  - [Withdrawal Invalidation Mitigation](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/withdrawal-invalidation-mitigation.md)
- Specifications
  - [Fault Dispute Game/Bond Incentives - DRAFT](https://github.com/ethereum-optimism/specs/pull/524/files)
  - [Portal - DRAFT](https://github.com/ethereum-optimism/specs/pull/519/files)
  - [AnchorStateRegistry - DRAFT](https://github.com/ethereum-optimism/specs/pull/517/files)

## DelayedWETH Failure Modes

### FM1: DelayedWETH Initialization Failure

#### **Description:**

The `DelayedWETH` contract could fail to initialize properly after deployment, leading to:

- Incorrect hold functionality configuration
- Missing or incorrect permissions
- Improper state initialization
- Broken interaction patterns with WETH

This is particularly critical as improper initialization could affect the contract's ability to handle emergency situations.

#### **Risk Assessment:**

High impact, Medium likelihood

#### **Mitigations:**

1. Comprehensive initialization checklist
2. Automated initialization verification
3. Permission validation system
4. Integration testing with WETH

#### **Detection:**

- Monitor initialization events
- Validate contract permissions
- Test hold functionality post-initialization
- Verify WETH interaction patterns

#### **Recovery Path(s):**

- Emergency reinitialization
- Manual permission correction
- Contract redeployment if necessary

#### Action items:

- [ ] Create initialization verification suite
- [ ] Implement permission validation tests
- [ ] Develop WETH integration tests
- [ ] Create initialization monitoring system

### FM2: DelayedWETH Legacy Version Confusion

#### **Description:**

Operators or guardians familiar with older versions of DelayedWETH might follow outdated procedures during emergencies. For example:

- Calling only `hold()` without subsequent `transfer()` on older versions
- Using deprecated functions or parameters
- Following outdated emergency procedures
- Misunderstanding version-specific behaviors

This risk is heightened during emergency situations when quick action is required.

#### **Risk Assessment:**

High impact, High likelihood

#### **Mitigations:**

1. Clear version documentation
2. Emergency procedure version tracking
3. Operator training on version differences
4. Version-specific runbooks

#### **Detection:**

- Monitor function call patterns
- Track emergency procedure execution
- Alert on deprecated function usage
- Version compatibility checks

#### **Recovery Path(s):**

- Immediate corrective actions
- Emergency procedure updates
- Operator retraining
- Documentation updates

#### Action items:

- [ ] Create version-specific runbooks
- [ ] Implement function call monitoring
- [ ] Develop version verification system
- [ ] Set up operator training program

## OptimismPortal Failure Modes

### FM3: Storage Layout Risks in OptimismPortal2

#### **Description:**

The upgrade involves in-place modifications to the `OptimismPortal2` contract, which introduces storage layout risks. Although we didn't modify the storage layout at all, there is still some small risk that the storage layout is accidentally modified.

#### **Risk Assessment:**

High impact, Low likelihood

#### **Mitigations:**

1. Comprehensive storage layout testing for `OptimismPortal2`
2. Use of storage layout lock files to prevent breaking changes
3. Verification of storage slot alignment post-upgrade

#### **Detection:**

- 3rd party audits
- Storage layout verification tests
- Monitoring of contract state changes post-upgrade

#### **Recovery Path(s):**

- Emergency fix deployment for storage misalignment
- Manual state reconstruction if necessary

#### Action items:

- [ ] Implement storage layout tests for `OptimismPortal2`
- [ ] Set up monitoring for storage changes

### FM4: Game that resolved incorrectly is not blacklisted within airgap delay period

#### **Description:**

Failure to properly handle incorrectly resolved games within the airgap delay period (assumption [aOP-003](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/optimism-portal.md#aop-003-invalid-withdrawals-can-never-be-finalized)) could lead to:

- Invalid withdrawals being finalized
- Security compromise of bridged assets
- System-wide trust failure
- Violation of critical invariant [iOP-001](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/optimism-portal.md#iop-001-invalid-withdrawals-can-never-be-finalized)

This is particularly critical as it could break the fundamental security assumption that invalid withdrawals can never be finalized.

#### **Risk Assessment:**

Critical impact, Low likelihood

#### **Mitigations:**

1. Strict game resolution validation is implemented
2. Games are already monitored: TODO: confirm!
3. Blacklisting procedure can be well-documented or even automated
   - Note: if game is blacklisted while it is the anchor game, the system will require manual intervention.
4. Multiple validation layers for withdrawals

#### **Detection:**

- Monitor game resolution events
- Track withdrawal validation status
- Alert on suspicious resolution patterns
- Monitor airgap period compliance

#### **Recovery Path(s):**

- Immediate withdrawal pause if needed
- Guardian can retire all existing games if needed
- Game can be blacklisted, although that will not reverse any finalized withdrawal
- Manual intervention for affected withdrawals, if possible

#### Action items:

- [ ] Confirm comprehensive game resolution monitoring
- [ ] Create airgap period compliance system
- [ ] Develop blacklisting runbook and possibly automate
- [ ] Create emergency response procedures

## AnchorStateRegistry Failure Modes

### FM5: AnchorStateRegistry Contract Address Mismatch

#### **Description:**

Post-upgrade, there is a risk that components may continue using the old `AnchorStateRegistry` address, leading to inconsistent state tracking.

#### **Risk Assessment:**

High impact, Low likelihood

#### **Mitigations:**

1. Comprehensive address update process
2. Verification of all dependent contracts

#### **Detection:**

- Monitor contract interactions for address mismatches

#### **Recovery Path(s):**

- Emergency update of dependent contracts
- Manual state synchronization if needed

#### Action items:

- [ ] Create address dependency map
- [ ] Implement address validation system

### FM6: Game Retirement Timestamp Updated Too Frequently or Anchor State Not Updated

#### **Description:**

The AnchorStateRegistry's game retirement timestamp could be updated too frequently, or the anchor state might fail to update properly. This could lead to:

- Inability to update the anchor game
- An old anchor game that breaks op-proposer (out of memory)
- Withdrawals can't get finalized, breaking invariant [iOP-002](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/optimism-portal.md#iop-002-valid-withdrawals-can-always-be-finalized-in-bounded-time)
- System-wide progress blockage due to stale anchor state

#### **Risk Assessment:**

Critical impact, Medium likelihood

#### **Mitigations:**

1. Implement internal monitoring and possibly rate limiting for retirement timestamp updates
2. 30-minute calls with each chain on permissionless proofs to walk through changes
3. Scripts to verify contract versions and appropriate actions
4. Clear documentation of legacy behavior and upgrade timeline

#### **Detection:**

- Track retirement timestamp changes
- Monitor game finalization rates
- Alert on frequent retirement timestamp updates
- Monitor anchor state updates

#### **Recovery Path(s):**

- Manual intervention to force game finalization
- Emergency governance action if needed
- Clear escalation path to contact Optimism team

#### Action items:

- [ ] Consider retirement timestamp monitoring
- [ ] Implement anchor state update monitoring
- [ ] Create version verification scripts
- [ ] Schedule training calls with chain operators

## FaultDisputeGame Failure Modes

### FM7: FaultDisputeGame Property Reporting

#### **Description:**

The FaultDisputeGame contract could fail to properly report its important properties (assumption aOP-001), including:

- Game type
- L2 block number
- Root claim value
- Game extra data
- Creation timestamp
- Resolution timestamp
- Resolution result
- Whether the game was the respected game type at creation

This could lead to:

- Incorrect game validation
- Invalid withdrawals being processed
- System-wide security compromise

#### **Risk Assessment:**

Critical impact, Low likelihood

#### **Mitigations:**

1. Unit/integration tests
2. 3rd party audits

#### **Detection:**

- ?

#### **Recovery Path(s):**

- Emergency blacklisting/retirement of affected games

#### Action items:\*\*

- [ ] Create incident response runbook for blacklisting/retirement

### FM8: Bond Distribution Errors

#### **Description:**

The bond distribution system could fail in multiple ways:

1. Bonds not distributed (users can't withdraw):

   - Incorrect refund recipient identification
   - Failed distribution transactions
   - System state preventing withdrawals

2. Excessive bond distribution:
   - Double-counting of bonds
   - Incorrect distribution calculations
   - Race conditions in withdrawal process

This is particularly critical as it affects both system security and user funds.

#### **Risk Assessment:**

High impact, Medium likelihood

#### **Mitigations:**

1. Unit/integration tests
2. 3rd party audits
3. DelayedWETH `hold` functionality
4. Clear documentation of distribution rules
5. 30-minute training calls with chain operators

#### **Detection:**

- Monitor bond deposits and withdrawals
- Track distribution events
- Validate balances
- Alert on unusual distribution patterns

#### **Recovery Path(s):**

- Manual bond redistribution
- Emergency contract fixes
- DelayedWETH emergency holds

#### Action items:

- [ ] Develop recovery procedures
- [ ] Create bond tracking system
- [ ] Implement distribution monitoring
- [ ] Set up automated alerts

### FM9: Game Participation Disincentivization

#### **Description:**

The withdrawal delay directly impacts the economic incentives that secure the system, and we changed the delay to start from game finalization. It doesn't seem like a significant impact.
Still, claimants may stop participating in dispute games if the time to withdraw bonds becomes too long, leading to:

- Reduced game participation
- Weaker security guarantees for the system
- Potential for invalid claims to go unchallenged
- Economic inefficiency in the dispute game mechanism

#### **Risk Assessment:**

High impact, Low likelihood

#### **Mitigations:**

1. Monitor communication channels

#### **Detection:**

- Measure dispute game engagement metrics

#### **Recovery Path(s):**

- New game deployment

#### Action items:

- [ ] Monitor communication channels
- [ ] Set up alerts for reduced participation

## Cross-Contract Failure Modes

### FM10: Contract Initialization State Corruption

#### **Description:**

The upgrade modifies multiple contracts (DelayedWETH, AnchorStateRegistry, OptimismPortal, FaultDisputeGame) that require proper initialization. Initialization failures could occur if:

- The initialization/deployment sequence is incorrect between interdependent contracts

#### **Risk Assessment:**

High impact, Medium likelihood

#### **Mitigations:**

1. OPCM audit, etc

#### **Detection:**

- Monitor initialization events
- Validate contract states post-initialization
- Cross-contract state verification

#### **Recovery Path(s):**

- Emergency upgrade to correct initialization
- Manual state reconciliation if needed
- Redeployment with correct initialization sequence

#### Action items:

### FM11: System-Wide Upgrade Coordination

#### **Description:**

The coordinated upgrade of multiple contracts could fail due to:

- Incorrect upgrade ordering
- Missed dependencies
- Inconsistent contract versions
- Initialization sequence errors

#### **Risk Assessment:**

Critical impact, Medium likelihood

#### **Mitigations:**

1. Detailed upgrade sequence planning
2. Dependency mapping
3. Version compatibility checks
4. Rollback procedures

#### **Detection:**

- Monitor upgrade events
- Track contract versions
- Validate system state

#### **Recovery Path(s):**

- Emergency rollback
- Partial redeployment
- State reconciliation

#### Action items:

- [ ] Create upgrade coordination plan
- [ ] Implement version tracking
- [ ] Develop rollback procedures

### Generic items we need to take into account:

See [./fma-generic-hardfork.md](./fma-generic-hardfork.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- Action items are listed in the relevant failure mode sections above.
- [ ] Validate assumption of game monitoring system
- [ ] Develop and test emergency response procedures
- [ ] Create runbooks for each failure mode
- [x] Implement automated testing for all new functionality
- [ ] Set up alerts for critical state changes

## Audit Requirements

The changes to the following contracts require auditing:

1. DelayedWETH - Changes to `hold` functionality
2. OptimismPortal - Game type management changes
3. AnchorStateRegistry - Unified state model
4. FaultDisputeGame - Bond distribution changes

The OPCM deployment scripts should also be reviewed as part of the audit process.
