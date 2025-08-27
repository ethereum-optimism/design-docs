<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [OPCM Improvements: Design Doc](#opcm-improvements-design-doc)
  - [Purpose](#purpose)
  - [Summary](#summary)
  - [Problem Statement + Context](#problem-statement--context)
    - [Validation Logic Fragmentation](#validation-logic-fragmentation)
    - [StandardValidator Integration Gap](#standardvalidator-integration-gap)
    - [Dispute Game Validation Issues](#dispute-game-validation-issues)
  - [Proposed Solution](#proposed-solution)
    - [1. StandardValidator Integration into OPCM](#1-standardvalidator-integration-into-opcm)
      - [Implementation Approach](#implementation-approach)
      - [Code Cleanup](#code-cleanup)
      - [Alternatives Considered](#alternatives-considered)
    - [2. Validation Integration in OPCM Methods](#2-validation-integration-in-opcm-methods)
      - [Alternatives Considered](#alternatives-considered-1)
    - [3. Redundant Assertion Removal](#3-redundant-assertion-removal)
      - [Implementation Approach](#implementation-approach-1)
      - [Alternatives Considered](#alternatives-considered-2)
    - [4. Enhanced Dispute Game Validation](#4-enhanced-dispute-game-validation)
      - [State 1: Super Games Only](#state-1-super-games-only)
      - [State 2: Standard Games Only](#state-2-standard-games-only)
    - [5. Remove proxyAdmin from OpChainConfig](#5-remove-proxyadmin-from-opchainconfig)
      - [Implementation Approach](#implementation-approach-2)
      - [Rationale](#rationale)
      - [Alternatives Considered](#alternatives-considered-3)
    - [Resource Usage](#resource-usage)
    - [Single Point of Failure and Multi Client Considerations](#single-point-of-failure-and-multi-client-considerations)
  - [Impact on Developer Experience](#impact-on-developer-experience)
    - [Positive Impacts](#positive-impacts)
    - [Breaking Changes](#breaking-changes)
  - [Risks & Uncertainties](#risks--uncertainties)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# OPCM Improvements: Design Doc

|                    |                            |
| ------------------ | -------------------------- |
| Author             | Amadi Michael              |
| Created at         | _2025-07-16_               |
| Initial Reviewers  | Matt Solomon, John Mardlin |
| Need Approval From | _To be assigned_           |
| Status             | _Draft_                    |

## Purpose

This design document outlines improvements to the OP Contracts Manager (OPCM) to enhance its validation scope while also reducing code duplication and improving maintainability of the codebase. The improvements focus on integrating the StandardValidator as a core component of OPCM, enhancing validation scope, and moving majority of the validation logic from the deploy scripts to the StandardValidator (which by the time would be a component of OPCM).

## Summary

The proposed improvements to OPCM include:

1. **StandardValidator Integration**: Integrate the StandardValidator as a core OPCM component following existing patterns
2. **Validation Consolidation**: After the above, add `validate()` calls to OPCM's `deploy()` and `upgrade()` methods
3. **Redundant Assertion Removal**: Remove duplicate validation logic from deploy scripts and other contracts, if they do not exist in the StandardValidator, they would be added to it.
4. **Dispute Game Validation Enhancement**: Improve StandardValidator's dispute game validation logic
5. **Remove proxyAdmin from OpChainConfig**: Remove the `proxyAdmin` field from the `OpChainConfig` struct used as input for `OPCM.upgrade()`

## Problem Statement + Context

The current OPCM implementation has several areas for improvement:

### Validation Logic Fragmentation

Currently, validation logic is scattered across multiple locations:

- `ChainAssertions.sol` contains validation checks
- Deploy scripts contain redundant validation logic
- `StandardValidator` exists as a separate component
- OPCM methods lack comprehensive validation

This fragmentation leads to:

- Code duplication and maintenance overhead
- In a few cases, inconsistent validation across different deployment paths
- Difficulty in ensuring all validation rules are applied consistently

### StandardValidator Integration Gap

The `StandardValidator` currently exists as a separate entity outside of OPCM, requiring:

- Separate deployment scripts (`DeployStandardValidator.s.sol`)
- Separate op-deployer components (`op-deployer/.../bootstrap/validator.go`)
- Manual integration in deployment workflows
- Need to keep track of the StandardValidator's address

### Dispute Game Validation Issues

The current dispute game validation in StandardValidator has limitations:

- Does not support validation for Super game types
- Missing validation of `respectedGameTypes` against registered implementations
- Potential for inconsistent dispute game configurations (e.g. when both super and standard games are registered)

## Proposed Solution

### 1. StandardValidator Integration into OPCM

The StandardValidator will be integrated into OPCM following the same patterns as existing components like `OPCMDeployer` and `OPCMUpgrader`.

#### Implementation Approach

- Rename the `StandardValidator` contract to `OPContractsManagerStandardValidator` to follow the existing naming pattern for OPCM components
- Add `challenger` as input to the OPCM deploy function since this is a requirement for the StandardValidator. Add the `challenger` in other places where it is needed.
- Now that the `StandardValidator` is a component of OPCM, add helper `validate()` functions to the OPCM which just call the `validate()` function on the StandardValidator.

#### Code Cleanup

After integration, the following code can be removed:

- `DeployStandardValidator.s.sol`
- `op-deployer/.../bootstrap/validator.go`

#### Alternatives Considered

- **Leave StandardValidator as a separate contract and component.**
  - This would require continued maintenance of separate deployment scripts, contract address etc.

### 2. Validation Integration in OPCM Methods

Add `validate()` calls to OPCM's core methods:

```solidity
function deploy(DeployConfig memory _config) external {
    // ... existing deployment logic

    // Validate configuration after deployment via the OPCM's validate() helper function
    validate(_config);
}

function upgrade(UpgradeConfig memory _config) external {
    // ... existing upgrade logic

    // Validate configuration after upgrade via the OPCM's validate() helper function
    validate(_config);
}
```

#### Alternatives Considered

- **Do not add validate() calls to OPCM methods.**
  - This would leave validation as an external concern, risking deployments and upgrades that bypass critical validation, and can lead to inconsistent validation logic across the codebase.

### 3. Redundant Assertion Removal

#### Implementation Approach

- Remove checks that already exist in `ChainAssertions.sol` from the other `Deploy*.s.sol` scripts and their invocations will simply just use `chainAssertions.methodName(...)`
- Move the checks in `ChainAssertions.sol` to the StandardValidator.
- Since codesize of the `OPContractsManagerStandardValidator` might be an issue, checks on the implementation contracts can exist in a separate contract which can be called just once at deploy time to validate implementation contracts constraints. This way at runtime, we only check the proxy contract constraints.

#### Alternatives Considered

- **Move the checks to the StandardValidator but process it at runtime and via an external call from the StandardValidator**
  - This would work but adds a lot of runtime gas cost to the StandardValidator's `validate()` function.

### 4. Enhanced Dispute Game Validation

Improve the `OPContractsManagerStandardValidator`'s dispute game validation to accept exactly two valid game-registry states:

#### State 1: Super Games Only

- Both super games registered
- NO other games registered
- Validates that `respectedGameTypes` matches registered super game implementations

#### State 2: Standard Games Only

- No super games registered
- ONLY `permissionedDisputeGame` OR `permissionedDisputeGame + faultDisputeGame`
- Validates that `respectedGameTypes` matches registered standard game implementations

### 5. Remove proxyAdmin from OpChainConfig

Remove the `proxyAdmin` field from the `OpChainConfig` struct used as input for `OPCM.upgrade()` function, since it can be retrieved via `systemConfigProxy.proxyAdmin()`.

#### Implementation Approach

- Remove the `proxyAdmin` field from the `OpChainConfig` struct
- Update the `OPCM.upgrade()` function to retrieve the `proxyAdmin` via `systemConfigProxy.proxyAdmin()` instead of using it as an input parameter
- Update all callers of `OPCM.upgrade()` to remove the `proxyAdmin` parameter from their calls

#### Rationale

The `proxyAdmin` field is not required as an input anymore since it can be retrieved via `systemConfigProxy.proxyAdmin()`. The `systemConfigProxy` is already a field in the `OpChainConfig` struct, making the `proxyAdmin` field redundant.

#### Alternatives Considered

- **Keep proxyAdmin as input parameter**
  - This would maintain backward compatibility but adds unnecessary confusion and redundancy since the value can be derived from existing data.

### Resource Usage

- Running the StandardValidator's `validate()` function after upgrade or deploy functions might reduce the amount of these calls can be done within a block, though not by a significant amount.

### Single Point of Failure and Multi Client Considerations

These improvements maintain OPCM's existing architecture and don't introduce new single points of failure.

## Impact on Developer Experience

### Positive Impacts

- **Simplified Deployment**: Developers no longer need to manage separate StandardValidator deployment and reduces the size of the deployment scripts.

### Breaking Changes

- Existing deployment or go scripts that rely on separate StandardValidator will need updates.
- Custom validation logic in deploy scripts will need to be removed or migrated.

## Risks & Uncertainties

1. **Integration Complexity**: Integrating StandardValidator into OPCM may introduce subtle bugs
2. **Validation Logic Changes**: Modifying dispute game validation could break existing deployments
3. **Migration Challenges**: Updating existing deployment scripts may be complex
