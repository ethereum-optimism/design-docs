# Purpose

The current fault proof system requires a redeployment of the `DisputeGame.sol`, `DisputeGameFactory.sol`, and `AnchorStateRegistry.sol`. Due to the immutable arguments in the constructor, specifically `_weth`, `_anchorStateRegistry`, and `_l2ChainId` the implementations must be fully redeployed for each new chain. The aim of this proposal is to simplify the Fauly Dispute Game contract setup so that all new deployments simply just require new proxies rather than new deployments of the factory and implementation.

# Summary

The `DisputeGameFactory.sol` should be upgraded to contain the arguments needed for every new `DisputeGame.sol`, and all of the constructor argument/logic should be moved into proxy which is actually pointed toward the `DisputeGame.sol` implementation, so that a single implementation of `DisputeGame.sol` can be used across each rollup.

# Problem Statement + Context

The Fault Dispute Game contracts are currently made up of a couple core contracts, `AnchorStateRegistry.sol`, `DelayedWETH.sol`, `DisputeGameFactory.sol`, and `FaultDisputeGame.sol/PermissionedFaultDisputeGame.sol`. When deploying "standard" rollups we currently need to deploy a new set of all of these contracts because specific immutable arguments in the implementation of `FaultDisputeGame.sol`, which in turn requires a specific `FaultDisputeGameFactory.sol` and in turn `AnchorStateRegistry.sol`. If `FaultDisputeGame.sol` implementations did not have chain specific immutables, then they could simply be added at the time of cloning on a per chain basis, requiring only one set of deployments and making the process simpler.

An important nuance here is that each `FaultDisputeGame.sol` deployment has both a minimal proxy with immutable args deployed that points at an implementation specific to the `FaultDisputeGameFactory.sol`. The two sets of immutable args are first specific to the fault dipuste game, then the second set on the implementation are specific to the rollup.
 
# Proposed Solution

In order to elegantly resolve this, the two sets of immutable args should be flattened into a single set of immutable args deployed alongside the proxy from the factory. The factory will instead store all of the different per-chain values, and pass in all arguments as a top level proxy with immutable args.

# Alternatives Considered

It would be possible to remove out the chain specific immutable arguments to simply become "extraData" passed into the proxy constructor, however this complicates the system further because the ultimate goal is to consildate all of our contracts so that each rollup can point to a single set of implementation contracts, so keeping some configuration parameters on the implementation of `FaultDisputeGame.sol` would not be flexible enough down the line.

# Risks & Uncertainties

The main risk for this upgrade is now that the factory must be properly configured, otherwise the factory will produce every single `FaultDisputeGame.sol` incorrectly.
