# Incident Response Improvements: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Incident Response Improvements: Failure Modes and Recovery Path Analysis](#incident-response-improvements-failure-modes-and-recovery-path-analysis)
  - [Introduction](#introduction)
  - [FM1: Contract deployment and/or initialization failures](#fm1-contract-deployment-andor-initialization-failures)
    - [Description](#description)
    - [Risk Assessment](#risk-assessment)
    - [Mitigations](#mitigations)
    - [Detection](#detection)
    - [Recovery Path(s)](#recovery-paths)
  - [FM2: Legacy runbook/response confusion](#fm2-legacy-runbookresponse-confusion)
    - [Description](#description-1)
    - [Risk Assessment](#risk-assessment-1)
    - [Mitigations](#mitigations-1)
    - [Detection](#detection-1)
    - [Recovery Path(s)](#recovery-paths-1)
    - [Action items](#action-items)
  - [FM3: Anchor state fails to progress within a time bound](#fm3-anchor-state-fails-to-progress-within-a-time-bound)
    - [Description](#description-2)
    - [Risk Assessment](#risk-assessment-2)
      - [Mitigations](#mitigations-2)
      - [Detection](#detection-2)
      - [Recovery Path(s)](#recovery-paths-2)
      - [Action items:](#action-items-1)
  - [FM4: Invalid anchor state](#fm4-invalid-anchor-state)
    - [Description](#description-3)
    - [Risk Assessment](#risk-assessment-3)
    - [Mitigations](#mitigations-3)
    - [Detection](#detection-3)
    - [Recovery Path(s)](#recovery-paths-3)
    - [Action items](#action-items-2)
  - [FM5: Bug in `FaultDisputeGame.wasRespectedGameTypeWhenCreated()` or Portal logic allows game that wasn't respected to be used for withdrawals](#fm5-bug-in-faultdisputegamewasrespectedgametypewhencreated-or-portal-logic-allows-game-that-wasnt-respected-to-be-used-for-withdrawals)
    - [Description](#description-4)
    - [Risk Assessment](#risk-assessment-4)
    - [Mitigations](#mitigations-4)
    - [Detection](#detection-4)
    - [Recovery Path(s)](#recovery-paths-4)
  - [FM6: Errors in bond distribution](#fm6-errors-in-bond-distribution)
    - [Description](#description-5)
    - [Risk Assessment](#risk-assessment-5)
    - [Mitigations](#mitigations-5)
    - [Detection](#detection-5)
    - [Recovery Path(s)](#recovery-paths-5)
    - [Action items](#action-items-3)
    - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
  - [Action Items](#action-items-4)
  - [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Italics are used to indicate things that need to be replaced._

|                    |                          |
| ------------------ | ------------------------ |
| Author             | wildmolasses             |
| Created at         | 2025-01-22               |
| Initial Reviewers  | smartcontracts           |
| Need Approval From | Matt Solomon, Josep Bové |
| Status             | Implementing Actions     |

## Introduction

This document covers the fault dispute game incident response improvements project. The project makes modifications to several key contracts to improve incident response capabilities:

- **FaultDisputeGame**: Has changes but almost any failure mode here is equivalent to a bug in the dispute game which we already have runbooks for. The main new consideration is potential accounting issues in the new bond refund functionality (documented in [FM4](#fm4-errors-in-bond-distribution)).

- **DelayedWETH**: Contract is basically unchanged and the diff is so minor there's really no genuine concern for any sort of failure.

- **AnchorStateRegistry**: Since this change doesn't actually make the ASR a critical dependency yet, the only real failure modes are:

  - The anchor state not being updated correctly (documented in [FM3](#fm3-anchor-state-fails-to-progress-within-a-time-bound))
  - Incorrect anchor state being set (documented in [FM4](#fm4-invalid-anchor-state))
  - (We're already planning to add monitoring for anchor state getting too old and we can easily add monitoring for anchor state being invalid)

- **OptimismPortal**: Very minor changes, with two main failure modes:
  - The new incident response functionality being misused (documented in [FM2](#fm2-legacy-runbookresponse-confusion))
  - A critical issue that allows a game that isn't respected to be used (either a bug in the changes or a bug in `FDG.wasRespectedGameTypeWhenCreated`) (documented in [FM5](#fm5-bug-in-faultdisputegamewasrespectedgametypewhencreated-or-portal-logic-allows-game-that-wasnt-respected-to-be-used-for-withdrawals))

Below are references for this project:

- Design Docs
  - [Anchor State Poisoning Protection](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/anchor-state-poison-protection.md)
  - [DelayedWETH Expansion](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/delayed-weth-expansion.md)
  - [FDG Refund Mode](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/fdg-refund-mode.md)
  - [Withdrawal Invalidation Mitigation](https://github.com/ethereum-optimism/design-docs/blob/b807dc5880f48a3631222e7f519c02a7668f2493/protocol/proofs/withdrawal-invalidation-mitigation.md)
- Specifications
  - [Fault Dispute Game/Bond Incentives](https://github.com/ethereum-optimism/specs/pull/560/files)
  - [Portal](https://github.com/ethereum-optimism/specs/blob/ab35e8181f34614503a03e18a282cce69ea634fe/specs/fault-proof/stage-one/optimism-portal.md)
  - [AnchorStateRegistry](https://github.com/ethereum-optimism/specs/blob/7706b68c6f8f4172f9396c03175e6c8cb299bfbc/specs/fault-proof/stage-one/anchor-state-registry.md)
  - [DelayedWETH](https://github.com/ethereum-optimism/specs/blob/a0c94920a3c3b6b3527e4f9c3f07b787cd9fe2ad/specs/fault-proof/stage-one/bond-incentives.md#delayedweth)

## FM1: Contract deployment and/or initialization failures

### Description

Any of the modified contracts could be deployed or initialized incorrectly leading to a variety of failure modes, depending on the exact contract impacted.

### Risk Assessment

- Impact: HIGH/CRITICAL
  - Reasoning: Highly depends on the exact failure mode but can be critical in the worst case.
- Likelihood: LOW
  - Reasoning: Very likely to be covered by both OPCM, testing, and auditing.

### Mitigations

- Validation via OPCM
- Integration testing
- 3rd party audit
- Testnet deployment

### Detection

Detection for this failure mode is highly dependent on the exact manner in which the mode occurs. Given the complexity in detecting this failure mode and the overlap with existing OPCM validation, we do not propose any new detection mechanisms for this mode.

### Recovery Path(s)

- Contract redeployment if necessary
- Superchain-wide pause mechanism

## FM2: Legacy runbook/response confusion

### Description

Operators or Guardians familiar with older versions of these contracts may follow outdated procedures during an incident.

For example:

- Calling only `DelayedWETH.hold()` without subsequent `DelayedWETH.transferFrom()` on a legacy `DelayedWETH`
- Failing to call `OptimismPortal.setRespectedGameType(type(uint32).max)` to invalidate games
- Following outdated emergency procedures / misunderstanding version-specific behaviors

This risk is heightened during emergency situations when quick action is required.

### Risk Assessment

- Impact: HIGH/CRITICAL
  - Reasoning: Exact risk depends on the specific incident response mechanism impacted. For instance, if the chain operator calls `OptimismPortal.setRespectedGameType` assuming that it will invalidate all existing games, the operator may not understand that existing games can _still_ be used to finalize withdrawals (critical risk). Other failure modes within this category are generally MEDIUM/HIGH.
- Likelihood: LOW
  - Reasoning: Only applies during the first ~2 weeks after the upgrade when existing `FaultDisputeGame` contracts may still refer to the legacy `DelayedWETH` contract. Failure to hold funds would be easily noticable, only results in any impact if the `hold` operation occurs very close to the finalization time. Does not apply to any future upgrades.

### Mitigations

- Clear version documentation
- Operator training on version differences
- Version-specific runbooks
- Retaining legacy `superchain-ops` tasks until further notice
- Directing operators to reach out to OP Labs for assistance with any incident

### Detection

Dedicated detection apparatus not recommended. Very low likelihood of occurring in practice and OP Labs would almost certainly be made aware of any instance where these incident response capabilities are being used.

### Recovery Path(s)

- Immediate corrective actions (where applicable)
- Operator retraining
- Documentation updates

### Action items

- [ ] Create version-aware runbooks
- [ ] Operator training sessions

## FM3: Anchor state fails to progress within a time bound

### Description

The anchor state held within the `AnchorStateRegistry` could fail to progress and become too old. In particular, this could be possible if:

- Guardian overuses the blacklisting/retirement mechanism to continuously block games from being used as the anchor state until the game is too old
- Bug in `FaultDisputeGame` fails to update anchor state on game finalization
- Bug in `AnchorStateRegistry` prevents games from updating the anchor state

### Risk Assessment

- Impact: HIGH
  - Reasoning: Failure mode would block user withdrawals until resolved.
- Likelihood: LOW/MEDIUM
  - Reasoning: Unlikely to occur via bug or Guardian overuse in practice. Possible that this occurs if there is a bug with the `FaultDisputeGame`'s call to `setAnchorState` if no extra monitoring exists.

#### Mitigations

- Monitor for the age of the anchor state

#### Detection

- Monitor for the age of the anchor state

#### Recovery Path(s)

- Manual intervention to refresh the anchor state: a newer valid anchor state can be permissionlessly set via `AnchorStateRegistry.setAnchorState(game)`.

#### Action items:

- [x] Implement anchor state age monitoring (Monitoring in hexagate)

## FM4: Invalid anchor state

### Description

An anchor state could be invalid if some assumption in [specs/fault-proof/stage-one/anchor-state-registry.md](https://github.com/ethereum-optimism/specs/blob/7706b68c6f8f4172f9396c03175e6c8cb299bfbc/specs/fault-proof/stage-one/anchor-state-registry.md) is violated, or if there's some bug in the `AnchorStateRegistry`. This implies a possibility of FaultDisputeGames that erroneously resolve, with invalid withdrawal finalization as a result.

### Risk Assessment

- Impact: CRITICAL
  - Reasoning: An invalid anchor state could result in an invalid finalized withdrawal.
- Likelihood: LOW
  - Reasoning: A bug in the `AnchorStateRegistry` is unlikely, and its assumptions are well-documented.

### Mitigations

- Unit/integration tests -
  [link](https://github.com/ethereum-optimism/optimism/blob/1d5770cd93a19797c7af2c0e9981265c3a3df847/packages/contracts-bedrock/test/dispute/AnchorStateRegistry.t.sol#L516-L804)
- 3rd party audits - [Offbeat Security](https://hackmd.io/@offbeatsecurity/optimism-incident-response-updates),
  [Spearbit](https://github.com/spearbit-audits/OP-Upgrade-13-review-200125)
- Anchor state monitoring - Monitored in hexagate

### Detection

- Monitor for invalid anchor states

### Recovery Path(s)

- Manual intervention to refresh the anchor state

### Action items

- [x] Runbook for handling this situation

## FM5: Bug in `FaultDisputeGame.wasRespectedGameTypeWhenCreated()` or Portal logic allows game that wasn't respected to be used for withdrawals

### Description

The `FaultDisputeGame.wasRespectedGameTypeWhenCreated()` function is now a critical dependency to Portal withdrawals. If there's a bug in the `wasRespectedGameTypeWhenCreated()` function or the code that calls it, it could allow withdrawals to be proven and finalized using a game that wasn't respected.

### Risk Assessment

- Impact: CRITICAL
  - Reasoning: A game that was not respected when it was created could be used to finalize an invalid withdrawal.
- Likelihood: LOW
  - Reasoning: The `FaultDisputeGame` has been extensively audited, and the setter logic around `wasRespectedGameTypeWhenCreated()` has been audited and is not complex, and the Portal logic is also well-audited.

### Mitigations

- Unit/integration tests - [link](https://github.com/ethereum-optimism/optimism/blob/20c221e87dedcd1ac4e446d0264ea7f4c46d9003/packages/contracts-bedrock/test/L1/OptimismPortal2.t.sol#L606-L616), [link](https://github.com/ethereum-optimism/optimism/blob/20c221e87dedcd1ac4e446d0264ea7f4c46d9003/packages/contracts-bedrock/test/L1/OptimismPortal2.t.sol#L1334-L1416), [link](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/dispute/AnchorStateRegistry.t.sol#L195-L212)
- 3rd party audits - [Offbeat Security](https://hackmd.io/@offbeatsecurity/optimism-incident-response-updates),
  [Spearbit](https://github.com/spearbit-audits/OP-Upgrade-13-review-200125)

### Detection

### Recovery Path(s)

## FM6: Errors in bond distribution

### Description

The bond distribution system could fail in multiple ways:

1. Bonds not distributed (users can't withdraw)
   - Incorrect refund recipient identification
   - Failed distribution transactions
   - System state preventing withdrawals
2. Excessive bond distribution
   - Double-counting of bonds
   - Incorrect distribution calculations
   - Race conditions in withdrawal process

### Risk Assessment

- Impact: HIGH
  - Reasoning: Impacts bonds but does not impact tokens within the bridge.
- Likelihood: MEDIUM
  - Reasoning: Bond refunding logic is somewhat more complex than other changes within this upgrade.

### Mitigations

- Unit/integration tests - [link](https://github.com/ethereum-optimism/optimism/blob/0072e29c5ac3efd31f5cf89d284e47bf5e0a4076/packages/contracts-bedrock/test/dispute/FaultDisputeGame.t.sol#L1903-L2100), [link](https://github.com/ethereum-optimism/optimism/blob/0072e29c5ac3efd31f5cf89d284e47bf5e0a4076/packages/contracts-bedrock/test/dispute/FaultDisputeGame.t.sol#L2393-L2475)
- 3rd party audits - [Offbeat Security](https://hackmd.io/@offbeatsecurity/optimism-incident-response-updates),
  [Spearbit](https://github.com/spearbit-audits/OP-Upgrade-13-review-200125)
- DelayedWETH `hold` functionality - [link](https://github.com/ethereum-optimism/optimism/blob/20c221e87dedcd1ac4e446d0264ea7f4c46d9003/packages/contracts-bedrock/src/dispute/DelayedWETH.sol#L105-L119)
- Clear documentation of distribution rules - [link](https://github.com/ethereum-optimism/specs/blob/5e1b5c44d46ff9848eb916d7cc10babf832f4226/specs/fault-proof/stage-one/bond-incentives.md#game-finalization)
- 30-minute training calls with chain operators

### Detection

- Existing monitoring of bonds within `op-dispute-mon` - [link]

### Recovery Path(s)

- Manual bond redistribution via `DelayedWETH`
- Fallback to `PermissionedDisputeGame` if necessary

### Action items

- [ ] Runbook for handling this situation

### Generic items we need to take into account:

See [./fma-generic-contracts.md](./fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- Action items are listed in the relevant failure mode sections above.
- [ ] Develop and test emergency response procedures
- [ ] Create runbooks for each failure mode
- [x] Implement automated testing for all new functionality

## Audit Requirements

The changes to the following contracts require auditing:

1. DelayedWETH - Changes to `hold` functionality
2. OptimismPortal - Game type management changes
3. AnchorStateRegistry - Unified state model
4. FaultDisputeGame - Bond distribution changes

The OPCM deployment scripts should also be reviewed as part of the audit process.
