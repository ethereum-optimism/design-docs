<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [[Project Name]: Design Doc](#project-name-design-doc)
  - [Summary](#summary)
  - [Problem Statement + Context](#problem-statement--context)
  - [Proposed Solution](#proposed-solution)
  - [Requirements and Expected behaviour](#requirements-and-expected-behaviour)
  - [Failure Mode Analysis](#failure-mode-analysis)
  - [Alternatives Considered](#alternatives-considered)
  - [Risks & Uncertainties](#risks--uncertainties)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# [Project Name]: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Michael Amadi                                      |
| Created at         | _2025-07-29_                                       |
| Initial Reviewers  | Matt Solomon, John Mardlin                 |
| Need Approval From | _To be assigned_                                    |
| Status             | _Draft_ |

## Summary

Currently, the OPCM's `upgrade()` function optionally upgrades the superchainConfigProxy based on the following checks:

For [op-contracts/v2.0.0](https://github.com/ethereum-optimism/optimism/blob/8d0dd96e494b2ba154587877351e87788336a4ec/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L477)
```solidity
        // If the SuperchainConfig is not already upgraded, upgrade it.
        if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl) {
            // Attempt to upgrade. If the ProxyAdmin is not the SuperchainConfig's admin, this will revert.
            upgradeTo(superchainProxyAdmin, address(superchainConfig), impls.superchainConfigImpl);
        }
```

and [op-contracts/v4.0.0](https://github.com/ethereum-optimism/optimism/blob/54c19f6acb7a6d3505f884bae601733d3d54a3a6/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L615)
```solidity
        // If the SuperchainConfig is not already upgraded, upgrade it. NOTE that this type of
        // upgrade means that chains can ONLY be upgraded via this OPCM contract if they use the
        // same SuperchainConfig contract. We will assert this later.
        if (_superchainProxyAdmin.getProxyImplementation(address(_superchainConfig)) != impls.superchainConfigImpl) {
            // Attempt to upgrade. If the ProxyAdmin is not the SuperchainConfig's admin, this will revert.
            upgradeToAndCall(
                _superchainProxyAdmin,
                address(_superchainConfig),
                impls.superchainConfigImpl,
                abi.encodeCall(ISuperchainConfig.upgrade, ())
            );
```

The check basically says that, if the implementation of the superchainConfig is not the same as the implementation the OPCM has stored, then upgrade it.

To upgrade the superchainConfig, the superchainProxyAdminOwner will need to call the OPCM's `upgrade()` function and optionally pass in values to upgrade any other chain's contracts as long as it controls that chain's proxyAdmin.

## Problem Statement + Context

This works for the most part but has a flaw. Assume the following:
- There are 2 chains in the Superchain, ChainA and ChainB
- ChainA's proxyAdmin is also the SuperchainConfig's ProxyAdmin
- The SuperchainConfig's implementation is Impl0
- The OPCM has stored Impl1 as the implementation to upgrade to

If ChainA's proxyAdmin (also the SuperchainConfig's ProxyAdmin) calls the OPCM's `upgrade()` function, the check above (if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)) will be true and it will upgrade the SuperchainConfig to Impl1 and also upgrade ChainA's L1 contracts.

If ChainB's proxyAdmin calls the OPCM's `upgrade()` function, the check above (if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)) will be false and so it will not enter the if block (if it did it will revert since ChainB's proxyAdmin is not the SuperchainConfig's ProxyAdmin). So it will skip this and go ahead to upgrade ChainB's L1 contracts.

Now the a problem arises here:
- The SuperchainConfig's implementation is Impl1
- A new OPCM is created with an expected implementation of Impl2
- The OPCM's `upgrade()` function is called and the SuperchainConfig's implementation is upgraded to Impl2
- A new chain, ChainC comes in or is behind and has to use the old OPCM first

When it calls the OPCM's `upgrade()` function, the check above (if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)) will be true and it will attempt to upgrade the SuperchainConfig which will fail since it's ProxyAdmin will not be the SuperchainConfig's ProxyAdmin.

This means all upgrades from that OPCM is not possible any longer.


## Proposed Solution

A proposed solution for this is to change the check:
- Create a variable `bool isSuperchainUpgraded` in the OPCMUpgrader contract.
- When `OPCM.upgrade()` is called, it delegates call to `OPCMUpgrader.upgrade()` as usual.
- `OPCMUpgrader.upgrade()` calls back into itself (to be able to access it's storage) and checks if the superchainConfig is already upgraded by checking the `isSuperchainUpgraded` variable.
- If it is not upgraded, it calls back into itself once more to set the `isSuperchainUpgraded` variable to true and then upgrade the superchainConfig.
- If it is already upgraded, it will skip the upgrade and continue execution.

While at it, it is proposed to also add support for different superchainConfigs i.e different Superchains. We can easily do this by replacing the `isSuperchainUpgraded` variable with a mapping `mapping(ISuperchainConfig superchainConfig => bool isSuperchainUpgraded)`. This way we can check which superchainConfig is being upgraded and set the `isSuperchainUpgraded` variable to true for that superchainConfig while also preventing the same superchainConfig from being upgraded again.

We should note also that:
- Multiple superchainConfigs can not be upgraded in one call to upgrade(...)
- It is assumed that all the chains upgraded in one call to upgrade(...) will be of the same superchain. We assert this through a check similar to this
```solidity
ISuperchainConfig _superchainConfig = _opChainInputs[0].systemConfig.optimismPortal().superchainConfig();
for (uint256 i = 0; i < _opChainInputs.length; i++) {
    if (_opChainInputs[i].systemConfig.optimismPortal().superchainConfig() != _superchainConfig) {
        revert SuperchainConfigMismatch();
    }
}
```

## Requirements and Expected behaviour

| Scenario | Description | Expected Behavior |
|----------|-------------|------------------|
| Dedicated SuperchainConfig ProxyAdmin | A SuperchainConfig has its own ProxyAdmin that is not shared with any of it's OP chains | When OPCM.upgrade() is called by this SuperchainProxyAdminOwner, it should check if the SuperchainConfig is already upgraded via a isSuperchainUpgraded mapping. If not upgraded, it should upgrade the SuperchainConfig and set isSuperchainUpgraded for that SuperchainConfig to true. No other contract upgrades would occur since this ProxyAdmin only controls the SuperchainConfig. |
| Shared ProxyAdmin | A SuperchainConfig shares a ProxyAdmin with an OP Chain | When OPCM.upgrade() is called by the shared ProxyAdminOwner, it should first check if the SuperchainConfig is already upgraded. If not upgraded, it should upgrade both the SuperchainConfig and the OP Chain's contracts. If already upgraded, it should only upgrade the OP Chain's contracts. |
| Non shared ProxyAdmin (SuperchainConfig Already Upgraded) | An OP Chain whose ProxyAdmin is not the SuperchainConfig's ProxyAdmin is being upgraded and its SuperchainConfig implementation has already been upgraded to the target version | When OPCM.upgrade() is called by the OP Chain's ProxyAdminOwner, it should check isSuperchainUpgraded for the SuperchainConfig and see it's true. It should then proceed with upgrading only the OP Chain's contracts. |
| Non shared ProxyAdmin (SuperchainConfig Not Upgraded) | An OP Chain whose ProxyAdmin is not the SuperchainConfig's ProxyAdmin is being upgraded but its SuperchainConfig has not been upgraded to the target version yet | When OPCM.upgrade() is called by the OP Chain's ProxyAdminOwner, it should check isSuperchainUpgraded for the SuperchainConfig. Since it's false, it will attempt to upgrade both. This must revert. |
| Multiple Chains in Single Upgrade | Multiple chains are being upgraded in a single call to upgrade() | All chains being upgraded must belong to the same superchain. The upgrade function should verify this by checking that all chains have the same SuperchainConfig, otherwise it should revert. |

## Failure Mode Analysis

- **Protecting the `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` function from being called directly by a malicious actor:**
    - Explainer: The way the OPCM sets the `isSuperchainUpgraded` variable is by calling  `OPCMUpgrader.setSuperchainUpgraded(ISuperchainConfig superchainConfig)` function. This is safe and does not need checks because if a malicious actor tries doing this, it will fail when trying to actually upgrade the superchainConfig. However, A malicious actor could call the `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` function directly (not via an OPCM upgrade call) to set the `isSuperchainUpgraded` variable to true for any superchainConfig while not being that Superchain's ProxyAdmin. 
    - Mitigation: The obvious way to do this is to check that the msg.sender is the Superchain's ProxyAdmin. This is possible for future OPCM versions because the proxyAdmin variable is accessible via the SuperchainConfig contract. For v2.0.0 and v4.0.0, this is not possible.
- **A chain might be upgraded when it's superchainConfig is not upgraded:**
    - Explainer: If the array of chains to upgrade include chains of more than one superchain, since we only check the superchainConfig of the first chain in the array and assume that to be the superchainConfig of all inputs, if any other chain in the array is of a different superchain, the function would not check if it's superchainConfig has been upgraded yet.
    - Mitigation: We can implement a simple loop to assert that all chains in the array are of the same superchain and have tests for this.

## Alternatives Considered

- Define the mapping `mapping(ISuperchainConfig superchainConfig => bool isSuperchainUpgraded) public isSuperchainUpgraded` and `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` in the OPCM contract:
    This was considered but was not implementable because OPCM components (since they are created before the OPCM) do not have any knowledge of what the OPCM address is in order to call the `isSuperchainUpgraded(ISuperchainConfig superchainConfig)` and `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` functions.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
