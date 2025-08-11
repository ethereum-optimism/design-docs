<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [[Project Name]: Design Doc](#project-name-design-doc)
  - [Summary](#summary)
  - [Problem Statement + Context](#problem-statement--context)
  - [Customer Requirements and Expected Behavior](#customer-requirements-and-expected-behavior)
  - [Proposed Solution](#proposed-solution)
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

If ChainA's proxyAdminOwner (also the SuperchainConfig's ProxyAdminOwner) calls the OPCM's `upgrade()` function, the check above (if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)) will be true and it will upgrade the SuperchainConfig to Impl1 and also upgrade ChainA's L1 contracts.

If ChainB's proxyAdminOwner calls the OPCM's `upgrade()` function, the check above (if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)) will be false and so it will not enter the if block (if it did it will revert since ChainB's proxyAdminOwner is not the SuperchainConfig's ProxyAdminOwner). So it will skip this and go ahead to upgrade ChainB's L1 contracts.

Now the a problem arises here:
- The SuperchainConfig's implementation is Impl1
- A new OPCM is created with an expected implementation of Impl2
- The OPCM's `upgrade()` function is called and the SuperchainConfig's implementation is upgraded to Impl2
- A new chain, ChainC comes in or is behind and has to use the old OPCM first

When it calls the OPCM's `upgrade()` function, the check above (`if (superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)`) will be true and it will attempt to upgrade the SuperchainConfig which:
- If the ProxyAdmin is not the SuperchainConfig's ProxyAdmin, it will revert. This essentially means that all upgrades from that OPCM is not possible any longer. Of course unless the SuperchainProxyAdminOwner calls upgrade which would upgrade the SuperchainConfig to Impl1 (effectively a downgrade) and make it possible for other chains to upgrade but for obvious reasons downgrading the SuperchainConfig is not the best course of action.
- If however it's ProxyAdmin is the SuperchainConfig's ProxyAdmin, it will upgrade the SuperchainConfig to impl1 (old implementation) and then upgrade ChainC's L1 contracts.

## Customer Requirements and Expected Behavior

| Scenario Number | ChainAProxyAdmin == superchainProxyAdmin?  | Target version is latest? | SuperchainConfig already on latest? | Expected Behavior                                                                                                                                                                                                     |
| --------------- | ---------------------------- | ------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1               | Yes                          | Yes                       | Yes                                 | Upgrades ChainA's L1 contracts, Function returns successfully                                                                                                                                                         |
| 2               | Yes                          | Yes                       | No                                  | Upgrades SuperchainConfig to latest version, Upgrades ChainA's L1 contracts, Function returns successfully                                                                                                            |
| 3a              | No (Caller is ChainAPAO)     | Yes                       | Yes                                 | Upgrades ChainA's L1 contracts, Function returns successfully                                                                                                                                                         |
| 3b              | No (Caller is SuperchainPAO) | Yes                       | Yes                                 | Nothing happens, Function returns successfully                                                                                                                                                                        |
| 4a              | No (Caller is ChainAPAO)     | Yes                       | No                                  | Function reverts since it will try to upgrade the SuperchainConfig to the latest version with the wrong owner                                                                                                         |
| 4b              | No (Caller is SuperchainPAO) | Yes                       | No                                  | Upgrades SuperchainConfig to latest version, Function returns successfully                                                                                                                                            |
| 5               | Yes                          | No                        | Yes                                 | It should revert since it will firstly attempt to upgrade to a former implementation, currently it does not revert and it upgrades the SuperchainConfig to the target version and also upgrades ChainA's L1 contracts |
| 6               | Yes                          | No                        | No                                  | It should ideally only work if we are upgrading to a newer version, currently it does not revert and it upgrades the SuperchainConfig to the target version and also upgrades ChainA's L1 contracts                   |
| 7a               | No (Caller is ChainAPAO)     | No                        | Yes                                 | Upgrades ChainA's L1 contracts, Function returns successfully                                                                                                                                                         |
| 7b               | No (Caller is SuperchainPAO) | No                        | Yes                                 | Function reverts since it has already been upgraded                                                                                                                                                                   |
| 8a               | No (Caller is ChainAPAO)     | No                        | No                                  | Function reverts since it will try to upgrade the SuperchainConfig to the latest version with the wrong owner                                                                                                         |
| 8b               | No (Caller is SuperchainPAO) | No                        | No                                  | Upgrades SuperchainConfig to latest version, Function returns successfully

## Proposed Solution

A proposed solution for this is to change the check to version comparisons. We can hardcode an expected version for the SuperchainConfig and compare it to the actual version. If the actual version is equal to the expected version, we can upgrade the SuperchainConfig. Otherwise we continue the execution.

While at it, it is proposed to also add support for different superchainConfigs i.e different Superchains. The proposed solution also allows for this.

We should note also that:
- Multiple superchainConfigs can not be upgraded in one call to upgrade(...)
- It is assumed that all the chains upgraded in one call to upgrade(...) will be of the same superchain. We assert this through a check similar to this
```solidity
ISuperchainConfig _superchainConfig = _opChainInputs[0].systemConfig.optimismPortal().superchainConfig();
for (uint256 i = 0; i < _opChainInputs.length; i++) {
    if (_opChainInputs[i].systemConfig.optimismPortal().superchainConfig() != _superchainConfig) {
        revert SuperchainConfigInconsistent();
    }
}
```
- It is also important to note that for any superchain asides the one with its superchainConfig hardcoded in the OPCM, in order to upgrade the superchainConfig, one of it's L1 chains which has the same ProxyAdmin as the SuperchainConfig's ProxyAdmin will need to be upgraded in the same transaction i.e the `OpChainConfig` should not be empty. This is because if the `OpChainConfig` is empty, the function defaults to using a hardcoded superchainConfig and superchainProxyAdmin.

## Failure Mode Analysis

- **A chain might be upgraded when it's superchainConfig is not upgraded:**
    - Explainer: If the array of chains to upgrade include chains of more than one superchain, since we only check the superchainConfig of the first chain in the array and assume that to be the superchainConfig of all inputs, if any other chain in the array is of a different superchain, the function would not check if it's superchainConfig has been upgraded yet.
    - Mitigation: We can implement a simple loop to assert that all chains in the array are of the same superchain and have tests for this.

## Alternatives Considered

- Define the mapping `mapping(ISuperchainConfig superchainConfig => bool isSuperchainUpgraded) public isSuperchainUpgraded` and `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` in the OPCM contract:
    This was considered but was not implementable because OPCM components (since they are created before the OPCM) do not have any knowledge of what the OPCM address is in order to call the `isSuperchainUpgraded(ISuperchainConfig superchainConfig)` and `setSuperchainUpgraded(ISuperchainConfig superchainConfig)` functions.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
