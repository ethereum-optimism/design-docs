# [Stage 1 changes]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[FM1: System Cannot Be Paused]](#fm1-system-cannot-be-paused)
  - [[FM2: System Cannot Be Unpaused]](#fm2-system-cannot-be-unpaused)
  - [[FM3: Invalid identifier on pause]](#fm3-invalid-identifier-on-pause)
  - [[FM4: Outdated Runbooks for Pause Mechanism]](#fm4-outdated-runbooks-for-pause-mechanism)
  - [[FM5: Unintended Unpause During Lockbox Changes / Upgrades]](#fm5-unintended-unpause-during-lockbox-changes)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | JosepBove                                          |
| Created at         | 2025-04-06                                         |
| Initial Reviewers  |                                                    |
| Need Approval From |                                                    |
| Status             |  Implementing actions                              |

## Introduction

This document covers an upgrade to the OP Stack that includes a number of changes to the pausing system to prepare the stack to match the stage framework changes that L2Beat will do in the middle of 2025.

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/202/)
- [Specs](https://github.com/ethereum-optimism/specs/pull/625)
- [Implementation](https://github.com/ethereum-optimism/optimism/pull/15174)

## Failure Modes and Recovery Paths

### FM1: System Cannot Be Paused

- **Description:** If the system's pause mechanism fails or is not properly implemented, the system cannot be paused in case of emergencies, potentially leading to continued operation during critical issues.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:** 
  1. Testing of pause functionality
  2. Clear documentation of pause procedures
- **Detection:** 
  - Monitoring of pause-related events
- **Recovery Path(s)**: 
  - If pause mechanism fails, immediate governance action may be required
  - May require emergency upgrade to fix pause mechanism
  - Consider implementing backup pause mechanisms
- **Action Item(s)**:
  - [x] FM1: Provide tests.
  - [ ] FM1: Provide monitoring solutions.
  - [ ] FM1: Demonstrate pause functionality in a production environment.

### FM2: System Cannot Be Unpaused

- **Description:** If the system's unpause mechanism fails, the system remains paused indefinitely, causing service disruption and potential loss of user access.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
  1. Testing of unpause functionality
  2. Clear documentation of unpause procedures
- **Detection:**
  - Monitoring of unpause-related events
- **Recovery Path(s)**:
  - If unpause mechanism fails, immediate governance action may be required
  - May require emergency upgrade to fix unpause mechanism
  - Consider implementing backup unpause mechanisms
- **Action Item(s)**:
- [x] FM2: Provide tests.
- [ ] FM2: Provide monitoring solutions.
- [ ] FM2: Demonstrate pause functionality in a production environment.

### FM3: Invalid identifier on pause

- **Description:** If somehow the identifier used in the pause mechanism of the superchain config receives an invalid value, the pause mechanism will not be triggered for the right cluster of the superchain.
- **Risk Assessment:** Medium impact, low likelihood
- **Mitigations:**
  1. Always call the pause function through our tooling, that gets the identifier (EthLockbox) from the OptimismPortal2 contract.
- **Detection:**
  - Post-upgrade monitoring and verification
- **Recovery Path(s)**:
  1. Unpausing the invalid identifier
  2. Pausing the correct identifier
- **Action Item(s)**:
- [x] FM3: Provide tests.

### FM4: Outdated Runbooks for Pause Mechanism

- **Description:** If runbooks and documentation for the pause mechanism are not updated to reflect the new system architecture and pause procedures, operators may not know how or when to use the pause functionality correctly, leading to delayed or incorrect responses during emergencies.
- **Risk Assessment:** High impact, medium likelihood
- **Mitigations:**
  1. Maintain up-to-date runbooks with clear procedures
  2. Regular training sessions for operators
- **Detection:**
  - Review of runbook accuracy when doing changes
  - Feedback from operator training sessions
- **Recovery Path(s)**:
  1. Immediate update of runbooks
  2. Emergency communication to operators
  3. Conduct emergency training if needed
- **Action Item(s)**:
  - [ ] FM4: Update the runbook for pause procedures
  - [ ] FM4: Conduct operator training on pause procedures

### FM5: Unintended Unpause During Lockbox Changes / Upgrades

- **Description:** If users change their lockbox or perform upgrades while the system is paused, it could trigger an unpause of the system. This could lead to premature resumption of operations before the original issue that caused the pause has been resolved.
- **Risk Assessment:** High impact, medium likelihood
- **Mitigations:**
  1. Be careful when doing lockbox changes or upgrades.
  2. Add information to the runbooks about this.
- **Detection:**
  - Monitor lockbox change events during pause periods
- **Recovery Path(s)**:
  1. Immediate re-pause if unintended unpause occurs
  2. Emergency communication to affected users
  3. Implement additional safeguards to prevent future occurrences
- **Action Item(s)**:
  - [ ] FM5: Update the runbooks to include pause-aware upgrade procedures

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).


- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements

The following contracts require an audit before production:

- `ETHLockbox.sol`
- `L1CrossDomainMessenger.sol`
- `L1ERC721Bridge.sol`
- `L1StandardBridge.sol`
- `OPContractsManager.sol`
- `OptimismPortal2.sol`
- `StandardValidator.sol`
- `SuperchainConfig.sol`
- `SystemConfig.sol`
- `AnchorStateRegistry.sol`
- `DelayedWETH.sol`
- `DeputyPauseModule.sol`

## Action Items

- [x] Schedule audit and prepare docs
