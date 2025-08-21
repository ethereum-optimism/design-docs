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

The proposed Custom Gas Token (CGT) upgrade path enables any OP Stack chain running the legacy CGT implementation to migrate to the new design. The migration process assumes that the set of smart contracts either corresponds to an official release (e.g., v1.8.0) or complies with the legacy CGT specifications. Only a single upgrade procedure is expected to transition a valid old implementation into the new one. Finally, we propose a method to address the issue of in-flight withdrawals when necessary.

## Problem Statement + Context

A new CGT implementation is being proposed to minimize the number of changes required to enable the custom gas token mode. In contrast, allow greater customizability for chain governors to define the native asset. However, chains already in production use a legacy CGT implementation, for example, based in the op-contracts/v1.8.0 release. There should be a way to migrate the legacy implementation into the new one to make it possible for them to stay in a standard version for present and future compatibility in the OP Stack development.

The goals for the migration must be:

- Removes obsolete code from existing contracts and introduces the new changes.
- Avoid users from performing migration actions.
- Reduce the risk of breaking existing integrations.
- Requires as little client and infrastructure work as possible.
- One upgrade procedure in L2 should be enough to make it work.

## Proposed Solution

The solution consists of an upgrade procedure that performs two main changes:

- Upgrade the chain
- Enable a new L1-L2 bridge for native asset minting, defined by the chain governor.

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
    - Migrate the ERC20 balance into a new bridge.

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

**Consensus / Execution clients**

Other than setting the new state containing the upgrade, no major changes are required. General configs might require being custom gas token aware.

**Proofs**

Other than setting the new state containing the upgrade, no major changes are required.

### Native Asset Bridging

Native asset bridging is outsourced from OP Stack core components, which means chain governors must implement their bridge set of contracts to migrate from the existing flow (through `OptimismPortal`) into a new one, which gets coupled into the `LiquidityController`.

For example, a chain governor can implement a set of contracts that acts as minter of the `LiquidityController` along with any set of L1 contracts to support bridging from L1.

### Liquidity Migration on `OptimismPortal`

The `OptimismPortal` would be required to be upgraded into an intermediate contract containing a function such as `migrateLiquidity` to move the ERC20 balance into the new bridge.

If the token is upgradable, the chain governor can also upgrade the token itself to swap the balances between the `OptimismPortal` and the new bridge.

### CGTBridge Deployment

To automate the deployment process of the `L1CGTBridge` and `L2CGTBridge`, a Factory contract will be deployed on L1. This Factory is responsible for deploying the L1 and L2 bridge contracts (proxy + implementation) and ensuring that their addresses are deterministically derived. Since the addresses are precomputed, each bridge will already know the address of its counterpart at the time of deployment, eliminating the need for an additional step to manually set the bridge addresses.

The `ProxyAdmin` owner must authorize the `L2CGTBridge` proxy address as a minter in the `LiquidityController`.

### Upgrade Overview Flow

A chain using the CGT legacy implementation would need to pass through an upgrade process, which requires the following steps:

1. Prepare and deploy the CGT Bridge.

A chain governor would require deploying the desired bridge for native asset bridging in L1. This bridge will remain unusable (or paused) from deposits and withdrawals until the chain’s upgrade is completed. Deploying this contract in advance allows the chain governor to shorten the time the native asset bridging is disabled from the UX perspective.
    
2. Upgrade all the contracts to the new CGT version via hard fork.
    
    Perform the upgrade for L1 and L2 contracts into the new implementations.
    
    Hardfork the L2 chain to add `NativeAssetLiquidity` (fund liquidity) and add `LiquidityController` and upgrade each existing predeploy contract.
    
    - For `NativeAssetLiquidity`, burn the amount of native asset that isn’t planned to be used.
        - Relevant for chains that already has a considerable amount of native asset supply scattered in the L2 state.
    - For `OptimismPortal`, use an intermediate contract/implementation to move the funds into the new L1 CGT Bridge.
4. Update the chain state in Fault Proofs.
5. Grants the `authorizedMinter` role to the `L2CGTBridge`.
6. Activates `L1CGTBridge`.
7. Restore the rest of the OP Stack Chains related services.

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
- **Return funds on L2 via a reimburse contract**: Deploy an reimburse contract as an authorized minter of the `LiquidityController`. The user initiates a request in L1, which verifies the unfinalized withdrawal status, sends a cross-chain message, and reimburses users directly on L2, effectively reversing unfinalized withdrawals. The contract can be retired once all reimbursements are completed.

## Risks & Uncertainties

- OP Stack chains might use contract versions that differ from v1.8.0 or even from the old CGT specs. A complete review of the core contracts is needed to ensure there are no inconsistencies with the proposed upgrade path that could introduce safety or liveness risks.
- Chains should ensure no critical dependencies on `depositERC20Transaction`, `initiateWithdrawal`, and related legacy CGT code being removed, and communicate with enough anticipation for proper migration.
- Chains with an enormous amount of native assets might cause an overflow in `NativeAssetLiquidity`. Currently, we set the liquidity to `type(uint248).max` wei, which should make such cases extremely rare.
