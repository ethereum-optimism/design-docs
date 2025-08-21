# Custom Gas Token Migration: Design Doc

| Author | *AgusDuha, Joxes, Skeletor* |
| --- | --- |
| Created at | *2025-07-31* |
| Initial Reviewers | *Mark Tyneway, TBD* |
| Need Approval From | *TBD* |
| Status | *In Review* |

## Purpose

Provide a repeatable, low-risk procedure for OP Stack chains that still run the legacy CGT implementation to migrate to the new proposed CGT design. The plan aims to keep full backward compatibility.

## Summary

The proposed Custom Gas Token upgrade lets any OP Stack chain introduce its native asset as the gas currency with almost no core-code intrusion: a single `isCustomGasToken()` flag turns off ETH transfer flows in all bridging methods, while two new pre-deploys, `NativeAssetLiquidity`, a contract with pre-minted assets, and `LiquidityController`, an owner-governed mint/burn router, hand supply control to chain governors or authorized “minter” contracts that can plug in anything from ERC-20 converters to third-party bridges or emission schedules. Wrapped-asset compatibility is preserved, and the entire system is initialized by seeding the native asset and allowing the governor to fully manage liquidity, such as integrating with the `L2CGTBridge` and burning any surplus. Lastly, we propose a method to address the issue of in-flight withdrawals when necessary.

## Problem Statement + Context

A new CGT implementation is being proposed to minimize the number of changes required to enable the custom gas token mode. In contrast, allow greater customizability for chain governors to define the native asset. However, chains already in production use the legacy CGT implementation. There should be a way to migrate the legacy implementation into the new one to make it possible for them to stay in a standard version for present and future compatibility in the OP Stack development.

The goals for the migration must be:

- Removes obsolete code from existing contracts and introduces the new changes.
- Avoid users from performing migration actions.
- Reduce the risk of breaking existing integrations.
- Requires as little client and infrastructure work as possible.
- One upgrade procedure in L2 should be enough to make it work.

## Proposed Solution

The solution consists of an upgrade procedure that performs two main changes:

- Upgrade the chain
- Enable a new bridge for native asset minting, defined by the chain governor.

### Chain’s Contracts Changes

**L1 Contracts Upgrade**

The following contracts are upgraded to the version that contains the new CGT implementation. The following main changes are expected:

- In `SystemConfig`:
    - Removal of the `GasPayingToken` library.
    - This removes the `gasPayingToken`, `gasPayingTokenName` and `gasPayingTokenSymbol`.
    - Removal of the `GAS_PAYING_TOKEN_DECIMALS` constant.
    - `isCustomGasToken()` boolean is read from `OptimismPortal`.
- In `OptimismPortal`:
    - Removal of the `depositERC20Transaction` and `gasPayingToken` dependencies.
    - Addition of the `isCustomGasToken` boolean to block `msg.value` > 0 during deposits, to block ETH sends.
    - Migrate ERC20 balance into a new bridge.

**L2 Contracts Upgrade**

The following contracts are upgraded to the version as well:

- In `L1Block`:
    - Removal of the `gasPayingToken`.
    - Modifications of `gasPayingTokenName`, `gasPayingTokenSymbol` to query from `LiquidityController`.
        - Retro-compatibility is guaranteed in the `WETH` predeploy contract, which contains old CGT code.
    - Set `isCustomGasToken` at initialization.
- In `L2ToL1MessagePasser`:
    - Import `isCustomGasToken` to block `msg.value` > 0 during withdrawals, to block native asset sends.
- `FeeVaults`:
    - Upgraded to allow withdrawals into an L2 address and turn off L1 withdrawals.

**New predeploys**

Since the legacy L1-L2 native asset bridging is down, liquidity creation is replaced by introducing the following contracts during the upgrade:

- `NativeAssetLiquidity`: This contains a large amount of native assets minted during the upgrade.
- `LiquidityController`: In charge of managing such liquidity.
- `L2CGTBridge`: predeploy addresses reserved for managing users’ requests to mint native assets, which the chain governor decides their implementation.

**Consensus / Execution clients**

Other than setting the new state containing the upgrade, no major changes are required. General configs might require being custom gas token aware.

**Proofs**

Other than setting the new state containing the upgrade, no major changes are required.

### Native Asset Bridging

Native asset bridging is outsourced from OP Stack core components, which means chain governors must implement their bridge set of contracts to migrate from the existing flow (through `OptimismPortal`) into a new one, which gets coupled into the `LiquidityController` and `L2CGTBridge` proxy.

For example, a chain governor can implement a set of contracts that uses the reserved `L2CGTBridge` pre-deploy along with any set of L1 contracts to support bridging from L1.

### Liquidity Migration on `OptimismPortal`

The `OptimismPortal` would be required to be upgraded into an intermediate contract containing a function such as migrateLiquidity to move the ERC20 tokens into the new bridge.

If the token is upgradable, the chain governor can also upgrade the token itself to swap the balances between the `OptimismPortal` and the new bridge.

### Upgrade Overview Flow

A chain using the CGT legacy implementation would need to pass through an upgrade process, which requires the following steps:

1. Prepare and deploy the CGT Bridge.

A chain governor would require deploying the desired bridge for native asset bridging in L1. This bridge will remain unusable (or paused) from deposits and withdrawals until the chain’s upgrade is completed. Deploying this contract in advance allows the chain governor to shorten the time the native asset bridging is disabled from the UX perspective.
    
2. Upgrade all the contracts to the new CGT version via hard fork.
    
    Perform the upgrade for L1 and L2 contracts into the new implementations.
    
    Hardfork the L2 chain to add `NativeAssetLiquidity` (fund liquidity), add `LiquidityController`, reserve the `L2CGTBridge` address, and upgrade each predeploy contract.
    
    - For `NativeAssetLiquidity`, burn the amount of native asset that isn’t planned to be used.
        - Relevant for chains that already run over pre-defined tokenomics.
    - For `L2CGTBridge`, point to a desired implementation if the chain governor desires to give it to a user.
    - For `OptimismPortal`, use an intermediate contract/implementation to move the funds into the new L1 CGT Bridge.
4. Update the chain state in Fault Proofs.
5. Activates `L1CGTBridge`.
6. Restore the rest of the OP Stack Chains related services.

### In-Flight Withdrawals

The upgrade may affect some initiated but not finalized withdrawals. One option that fits the migration flow is using the `L1CGTBridge` to support legacy withdrawals. This is done by porting `OptimismPortal`’s withdrawal logic and the latest valid state into the CGT Bridge and consulting the withdrawal mapping of `OptimismPortal` to ensure no completed withdrawals are replayed.

The goal is for applications and users to consume the same APIs to prove and finalize into the new `L1CGTBridge`, which will contain the liquidity.

### Resource Usage

No usage changes.

### Single Point of Failure and Multi Client Considerations

Upgrades are a sensitive process and should be performed according to best practices. 

## Failure Mode Analysis

TBD.

## Impact on Developer Experience

The following aspects should be taken into account:

- Contracts that rely on `depositERC20Transaction` and `initiateWithdrawal` methods will revert when tried to call, so they need to upgrade their app to migrate to a new contract that uses the new bridging methods.
    - It is recommended to track the `initiateWithdrawal` to understand the impact on them.
- Bridge UI interfaces must update the native asset bridging methods as well.
- Apps relying on `SystemConfig.gasPayingToken` methods must be aware and instead rely on the `L1Block` / `LiquidityController`.

Please review the new CGT design doc for the impact of the latest proposed implementation.

## Alternatives Considered

**On Unfinalized Withdrawals**

One of the main user-facing risks during the upgrade processes is that some unfinalized withdrawals that try to withdraw native assets will fail after the upgrade. Other alternatives were considered to satisfy each user to be able to get their assets during or after the upgrade:

- **Pause or block msg.value > 0 in L2ToL1MessagePasser**: Upgrade the contract to block native asset withdrawals and let the users (or the chain governor) prove and finalize the withdrawals before fully upgrading the chain, cleaning the withdrawal queue.
- **OptimismPortal maintains the finalizeWithdrawal legacy code**: retain the withdrawal legacy code to maintain backward compatibility, at the cost of keeping old code just for this purpose and storing the exact portion of tokens there, which should be upgraded on a future date, since new CGT chains will not have this logic.
- **Inherit the CGT Bridge interface into OptimismPortal**: replicate the latest `OptimismPortal` logic (`ETHLockbox`), in this case, where the native assets are pulled from the CGT Bridge, at the cost of adding new code for this sole purpose.
- **Upgrade the Gas Token**: Only for chains that use an upgradable token can it simply mint tokens and reimburse the affected users due to the migration.

## Risks & Uncertainties

- Chains should ensure no critical dependencies on `depositERC20Transaction`, `initiateWithdrawal`, and related legacy CGT code being removed, and communicate with enough anticipation for proper migration.
- Chains with an enormous amount of native assets might cause an overflow in `NativeAssetLiquidity`. Currently, we set the liquidity to `type(uint248).max` wei, which should make such cases extremely rare.
