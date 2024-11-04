# Purpose

The current fault proof system requires a redeployment of the `DisputeGame.sol`, `DisputeGameFactory.sol`, and `AnchorStateRegistry.sol`. Due to the immutable arguments in the constructor, specifically `_weth`, `_anchorStateRegistry`, and `_l2ChainId` the implementations must be fully redeployed for each new chain. The aim of this proposal is to simplify the Fauly Dispute Game contract setup so that all new deployments simply just require new proxies rather than new deployments of the factory and implementation.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

# Problem Statement + Context

The Fault Dispute Game contracts are currently made up of a couple core contracts, `AnchorStateRegistry.sol`, `DelayedWETH.sol`, `DisputeGameFactory.sol`, and `FaultDisputeGame.sol/PermissionedFaultDisputeGame.sol`. When deploying "standard" rollups we currently need to deploy a new set of all of these contracts because specific immutable arguments in the implementation of `FaultDisputeGame.sol`, which in turn requires a specific `FaultDisputeGameFactory.sol` and in turn `AnchorStateRegistry.sol`. If `FaultDisputeGame.sol` implementations did not have chain specific immutables, then they could simply be added at the time of cloning on a per chain basis, requiring only one set of deployments and making the process simpler.

An important nuance here is that each `FaultDisputeGame.sol` deployment has both a minimal proxy with immutable args deployed that points at an implementation specific to the `FaultDisputeGameFactory.sol`. The two sets of immutable args are first specific to the fault dipuste game, then the second set on the implementation are specific to the rollup.
 
# Proposed Solution

In order to elegantly resolve this, the two sets of immutable args should be flattened into a single set of immutable args deployed alongside the proxy from the factory. The factory will instead store all of the different per-chain values, and pass in all arguments as a top level proxy with immutable args.

# Alternatives Considered

It would be possible to remove out the chain specific immutable arguments to

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
