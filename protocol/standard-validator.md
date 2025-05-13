# Standard Validator Improvements

|                    |                             |
| ------------------ | --------------------------- |
| Author             | Maurelian                   |
| Created at         | 2025-05-07                  |
| Initial Reviewers  | Matt Solomon, Blaine Malone |
| Need Approval From | Kelvin Fichter              |
| Status             | Draft                       |

## Purpose

To clarify the purpose of the standard validator and layout a plan for incorporating it more
effectively into the OPCM based deployment and upgrade process.

## Summary

The Standard Validator should be renamed to the OP Contracts Validator (the remainder of this document
uses "Standard Validator" in case the proposal is controversial).

It will become the new source of truth for verifying that a given L1 system has:
1. the correct entities assigned to each role
2. configuration parameters within the allowed ranges
3. the correct relationships between various contracts

It should be deployed alongside the OPCM, and effectively incorporated into the release process.

## Problem Statement + Context

### Lack of integration

The existing
[StandardValidator](https://github.com/ethereum-optimism/optimism/blob/e62c14e64f08ae3cd82973b41315d5797810569b/packages/contracts-bedrock/src/L1/StandardValidator.sol#L1)
contract is purely a 'side car' to the both the L1 contracts and the OPCM (which are independent).

It is currently used during the upgrade process in the [superchain-ops repo following upgrades tasks](https://github.com/ethereum-optimism/superchain-ops/blob/main/src/improvements/template/OPCMUpgradeV200.sol#L113),
however this is not enforced within the OPCM, nor is it included as part of the OPCM's deployment
flow.

### Issues with validations

As can be seen from the many [long `expectedError` strings in the superchain-ops repo](https://github.com/ethereum-optimism/superchain-ops/blob/44f9a09186073456ac1e03c485206b74aa742c30/src/improvements/template/OPCMUpgradeV200.sol#L115-L155),
there are a variety of errors which have been deemed acceptable for one reason or another.

Before the StandardValidator can be included as a required check in all OPCM operations, we
need to have confidence that any failures merit reverting the operation.

In order to clarify the cause of these `expectedErrors` it is necessary to clarify the jobs the
StandardValidator is currently doing:

1. Enforcing the standard configuration.
2. Checking that values are 'as expected', even for chains which do not satisfy the standard configuration.

To clarify the latter, chains may have a different L1ProxyAdminOwner (ie.
[Unichain](https://github.com/ethereum-optimism/superchain-registry/blob/0831c2509152b457d865634616925ca6240b219e/superchain/configs/mainnet/unichain.toml#L45)),
or a different Challenger (ie.
[Base](https://github.com/ethereum-optimism/superchain-registry/blob/0831c2509152b457d865634616925ca6240b219e/superchain/configs/mainnet/base.toml#L47)).
In such cases the StandardValidator currently returns an error string (ie. `PROXYA-10` or `PDDG-120`),
  the current practice is to recognize that this error is expected, but does not allow the StandardValidator
  to make assertions on the expected value of these roles.

## Proposed Solution

The proposed solution is broken up in to work that can be done prior to upgrade 16, and work
that will come afterwards.

### Pre-upgrade 16 work

#### 1. Validation Overrides:

The `StandardValidator` should correctly handle certain allowable/expected deviations.

A non-exhaustive list includes:

- `absolutePrestate`: this value varies for each chain. The [current design](https://github.com/ethereum-optimism/optimism/blob/e62c14e64f08ae3cd82973b41315d5797810569b/packages/contracts-bedrock/src/L1/StandardValidator.sol#L61) accepts this value as an input, so no change is needed.
- Various roles: the default roles should be hardcoded into the StandardValidator, but calls to
  `validate()` should optionally override them.
  - note: The
    [guardian](https://github.com/ethereum-optimism/optimism/blob/e62c14e64f08ae3cd82973b41315d5797810569b/packages/contracts-bedrock/src/L1/StandardValidator.sol#L138)
    is not currently checked, and the paused state should not be checked.

Note: In order to definitively differentiate between chains which satisfy the default standard configuration,
from those which require overrides, `allowFailure` MUST be `true` if any overrides are provided,
and in that case the error string returned will include at least a default error such as `HAS-OVERRIDES`.

Note: The initial list of overrides should be minimal, and only include values which are known to
vary between chains based on usage in previous tasks.

Note: the charter specifically calls out the Base and Uni L1PAO as allowed on their chains only, this
 should be reflected in the StandardValidator.

A small mock up of the implementation approach to allow overridng specific values, focusing on the
`guardian` value is below:

```solidity
contract StandardGuardianValidator {
  struct ValidationInput {
    IProxyAdmin proxyAdmin;
    ISystemConfig sysCfg;
    uint256 l2ChainID;
  }

  struct ValidationOverrides {
    address guardian;
  }

  address immutable GUARDIAN;
  ISuperchainConfig immutable SUPERCHAIN_CONFIG;

  constructor(address _guardian, ISuperchainConfig _superchainConfig) public {
    GUARDIAN = _guardian;
    SUPERCHAIN_CONFIG = _superchainConfig;
  }

  function expectedGuardian(ValidationOverrides memory _overrides) public view returns (address) {
    if (_overrides.guardian != address(0)) {
      return _overrides.guardian;
    }
    return GUARDIAN;
  }

  /// @notice Validates the standard configuration of the L1 system.
  /// @param _allowFailure Whether to allow the validation to fail.
  /// @param _input The input parameters for the validation.
  /// @param _overrides The overrides for the validation. This is an array of structs, with a maximum of one element.
  ///                   An array is used as a means of making the overrides optional.
  /// @return _errors The errors from the validation.
  function validate(
    bool _allowFailure,
    ValidationInput memory _input,
    ValidationOverrides[] memory _overrides
  ) public view returns (string memory _errors) {
    // Initialize the overrides to an empty struct.
    ValidationOverrides memory overrides = ValidationOverrides({});
    string memory errors = "";

    if(_overrides.length > 1) {
      revert TooManyOverrides();
    }

    if (_overrides.length == 1) {
      if (!_allowFailure) {
        revert OverridesProvidedWithoutFailureAllowance();
      }
      _errors = "HAS-OVERRIDES";
    }

    // If overrides are provided, overwrite the empty struct with the first element of the array.
    if (_overrides.length == 1) {
      overrides = _overrides[0];
    }

    _errors = internalRequire(superchainConfig.guardian() == expectedGuardian(overrides), "SPRCFG-10", _errors);

    if (bytes(_errors).length > 0 && !_allowFailure) {
      revert(string.concat("StandardValidator: ", _errors));
    }

    return _errors;
  }
}
```

#### 2. Dispute Game handling:

Various aspects of the Dispute Game are not properly handled by the StandardValidator, as it needs
to handle different combinations of game implementations and respected game types:

There are two acceptable states for `gameImpls`:

1. both super games set AND no other games
2. no super games AND permissionedDisputeGame only OR both the permissionedDisputeGame and faultDisputeGame

In both cases, the `respectedGameTypes` should correspond to one of the registered game implementations.

### Post-upgrade 16 work (Target: Upgrade 18)

#### 1. OPCM integration:

The `StandardValidator` should become a [component of the OPCM](https://github.com/ethereum-optimism/optimism/blob/0c986c9b40b8aedfce663421d7074fb856229cde/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1762-L1768),
so that the `validate()` function can be called directly on the OPCM without needing to track
another address.

#### 2. Enhance the StandardValidator's status as source of truth.

There are currently 4 places that define parts of the standard config:

- The [Standard Charter](https://github.com/ethereum-optimism/OPerating-manual/blob/e1b305a96a0b60515cc1111a26e73e1973d9c34e/Standard%20Rollup%20Charter.md#role-configuration-exceptions)
- The [specs configurability section](https://specs.optimism.io/protocol/configurability.html)
- The StandardValidator contract
- The TOML files in the [validation/standard directory](https://github.com/ethereum-optimism/superchain-registry/tree/9095778d45a5066649890ee838f87b27062a0d4d/validation/standard) of the superchain registry

We should work to remove as many as these redundant references as possible.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
