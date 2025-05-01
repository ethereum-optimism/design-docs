# [Stage 1 changes]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[FM1: System Cannot Be Paused]](#fm1-system-cannot-be-paused)
  - [[FM2: System Cannot Be Unpaused]](#fm2-system-cannot-be-unpaused)
  - [[FM3: Upgrade Process Bug]](#fm3-upgrade-process-bug)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | JosepBove                                          |
| Created at         | 2025-04-06                                         |
| Initial Reviewers  |                                                    |
| Need Approval From |                                                    |
| Status             |  Draft                                             |

## Introduction

This document covers _[project name, high-level summary of the project, and scope of this analysis]._

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/202/)
- [Specs](https://github.com/ethereum-optimism/specs/pull/625)
- [Implementation](https://github.com/ethereum-optimism/optimism/pull/15174)

## Failure Modes and Recovery Paths

### FM1: System Cannot Be Paused

- **Description:** If the system's pause mechanism fails or is not properly implemented, the system cannot be paused in case of emergencies, potentially leading to continued operation during critical issues.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:** 
  1. Implement robust pause mechanism with multiple authorized addresses
  2. Testing of pause functionality
  3. Clear documentation of pause procedures
- **Detection:** 
  - Monitoring of pause-related events
- **Recovery Path(s)**: 
  - If pause mechanism fails, immediate governance action may be required
  - May require emergency upgrade to fix pause mechanism
  - Consider implementing backup pause mechanisms

### FM2: System Cannot Be Unpaused

- **Description:** If the system's unpause mechanism fails, the system remains paused indefinitely, causing service disruption and potential loss of user access.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations:**
  1. Implement robust unpause mechanism
  2. Testing of unpause functionality
  3. Clear documentation of unpause procedures
- **Detection:**
  - Monitoring of unpause-related events
- **Recovery Path(s)**:
  - If unpause mechanism fails, immediate governance action may be required
  - May require emergency upgrade to fix unpause mechanism
  - Consider implementing backup unpause mechanisms

### FM3: Upgrade Process Bug

- **Description:** If there is a bug in the upgrade process or implementation, it could lead to incorrect behavior, or complete system failure.
- **Risk Assessment:** High impact, medium likelihood
- **Mitigations:**
  1. Comprehensive testing of upgrade process
  2. Multiple deployments before mainnet
  3. Refer to [fma-generic-contracts](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md), items applies to this case. Any new implementation must go through the audit and testing. Upgrades should not be regular in this contract and it should maintain minimal code. Upgrade procedures and keys must follows the proper security practices.
- **Detection:**
  - Post-upgrade monitoring and verification
  - Verify proxy and implementation contracts through automated checks in the superchain-ops task
- **Recovery Path(s)**:
  - If upgrade fails, system may need to be rolled back
  - Emergency fix deployment may be required

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

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
