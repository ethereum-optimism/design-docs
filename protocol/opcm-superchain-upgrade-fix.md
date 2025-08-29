<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [OPCM SuperchainConfig Upgrade Fix: Design Doc](#opcm-superchainconfig-upgrade-fix-design-doc)
  - [Summary](#summary)
  - [Problem Statement + Context](#problem-statement--context)
  - [Customer Requirements and Expected Behavior](#customer-requirements-and-expected-behavior)
    - [OPContractsManager upgradeSuperchainConfig function:](#opcontractsmanager-upgradesuperchainconfig-function)
    - [OPContractsManager upgrade function:](#opcontractsmanager-upgrade-function)
  - [Proposed Solution](#proposed-solution)
    - [Proposal 1: Move the SuperchainConfig upgrade functionality to it's own function:](#proposal-1-move-the-superchainconfig-upgrade-functionality-to-its-own-function)
    - [Proposal 2: SuperchainConfig Upgrade Check Fix](#proposal-2-superchainconfig-upgrade-check-fix)
    - [Proposal 3: Support for different superchainConfigs i.e different Superchains](#proposal-3-support-for-different-superchainconfigs-ie-different-superchains)
  - [Failure Mode Analysis](#failure-mode-analysis)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# OPCM SuperchainConfig Upgrade Fix: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Michael Amadi                                      |
| Created at         | _2025-07-29_                                       |
| Initial Reviewers  | Matt Solomon, John Mardlin                         |
| Need Approval From | _To be assigned_                                   |
| Status             | _Draft_                                            |

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

If ChainB's `proxyAdminOwner` calls the OPCM's `upgrade()` function, the check above (if (`superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)`) will be false and so it will not enter the if block (if it did it will revert since ChainB's `proxyAdminOwner` is not the SuperchainConfig's `ProxyAdminOwner`). So it will skip this and go ahead to upgrade ChainB's L1 contracts.

Now the a problem arises here:
- The SuperchainConfig's implementation is Impl1
- A new OPCM is created with an expected implementation of Impl2
- The OPCM's `upgrade()` function is called and the SuperchainConfig's implementation is upgraded to Impl2
- A new chain, ChainC comes in or is behind and has to use the old OPCM first

When it calls the OPCM's `upgrade()` function, the check above (if (`superchainProxyAdmin.getProxyImplementation(address(superchainConfig)) != impls.superchainConfigImpl)`) will be true and it will attempt to upgrade the SuperchainConfig which:
- If the ProxyAdmin is not the SuperchainConfig's ProxyAdmin, it will revert. This essentially means that all upgrades from that OPCM is not possible any longer. Of course unless the SuperchainProxyAdminOwner calls upgrade which would upgrade the SuperchainConfig to Impl1 (effectively a downgrade) and make it possible for other chains to upgrade but for obvious reasons downgrading the SuperchainConfig is not the best course of action.
- If however it's ProxyAdmin is the SuperchainConfig's ProxyAdmin, it will upgrade the SuperchainConfig to impl1 (old implementation) and then upgrade ChainC's L1 contracts.

## Customer Requirements and Expected Behavior

### OPContractsManager upgradeSuperchainConfig function:
- The SuperchainConfig has not been upgraded by the OPCM being called.
- The caller must be the SuperchainConfig's ProxyAdminOwner.

### OPContractsManager upgrade function:
- ChainA's SuperchainConfig has been upgraded by the OPCM being called.
- The caller must be ChainA's ProxyAdminOwner.

## Proposed Solution

### Proposal 1: Move the SuperchainConfig upgrade functionality to it's own function:
This will help simplify the `upgrade()` function and make the checks easier to reason about and implement. As part of this the `opcm-upgrade-checks` script will also need to be updated to support this too.


### Proposal 2: SuperchainConfig Upgrade Check Fix
A proposed solution for this is to change the check to version comparisons. We can hardcode an expected previous and target versions for the SuperchainConfig and compare it to the actual version when performing upgrades. E.g
- For `The SuperchainConfig has not been upgraded by the OPCM being called` in `OPCM.upgradeSuperchainConfig()`, we check if the SuperchainConfig's actual version is less than the target version. If it is, we can upgrade the SuperchainConfig. Otherwise we revert. We do this check to ensure that SuperchainConfigs are not accidentally downgraded.
- For `ChainA's SuperchainConfig has been upgraded by the OPCM being called` in `OPCM.upgrade()`, we check if the SuperchainConfig's actual version is greater than or equal to the expected target version. If it is, we can proceed to upgrade the OP Chains L1 contracts. Otherwise we revert. We do not make a strict check here so that OP Chains that are more than 2 SuperchainConfig versions behind can still be upgraded.


### Proposal 3: Support for different superchainConfigs i.e different Superchains
While at it, it is proposed to also add support for different superchainConfigs i.e different Superchains. We can easily achieve this by getting the superchainConfig of each OPChain from its optimismPortal (gotten from the systemConfig in the OPChainConfig struct) rather than using the hardcoded SuperchainConfig address in the OPCM. Note this means that it is possible for OPChains of different superchains to be upgraded in one call to `OPCM.upgrade()` as long as they share the same proxyAdminOwner. This means that we must check that each OPChain's superchainConfig is at the correct version before proceeding to upgrade it, otherwise we revert the transaction.


## Failure Mode Analysis

- **A chain might be upgraded when it's superchainConfig is not upgraded:**
    - Mitigation: We can implement a simple loop to assert that all chains in the array are of the same superchain and have tests for this.
