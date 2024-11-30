# Purpose

The current fault-proof system requires a redeployment of the `DisputeGame.sol`, `DisputeGameFactory.sol`, and `AnchorStateRegistry.sol`. Due to the immutable arguments in the constructor, specifically `_weth`, `_anchorStateRegistry`, and `_l2ChainId` the implementations must be fully redeployed for each new chain. This proposal aims to simplify the Fault Dispute Game contract setup so that all new deployments simply just require new proxies rather than new deployments of the factory and implementation.

# Summary

The `DisputeGameFactory.sol` should be upgraded to allow for multiple "Creator" contracts that control the creation of each GameType, which when proxied will allow for all new standard rollup deployments to be able to use proxies rather than needing to redeploy the Dispute Game system each time, instead pointing to a canonical implementation. Additionally, "Creator" contracts will make upgrades easier allowing the reconfiguration of new `DisputeGame.sol` clones without needing to redeploy the entire contract.

# Problem Statement + Context

The Fault Dispute Game contracts are currently made up of a couple of core contracts, `AnchorStateRegistry.sol`, `DelayedWETH.sol`, `DisputeGameFactory.sol`, and `FaultDisputeGame.sol/PermissionedFaultDisputeGame.sol`. When deploying "standard" rollups we currently need to deploy a new set of all of these contracts because of specific immutable arguments in the implementation of `FaultDisputeGame.sol`, which in turn requires a specific `FaultDisputeGameFactory.sol` and in turn `AnchorStateRegistry.sol`. If `FaultDisputeGame.sol` implementations did not have chain-specific immutables, then they could simply be added at the time of cloning on a per-chain basis, requiring only one set of deployments and making the process simpler.
An important nuance here is that each `FaultDisputeGame.sol` deployment has both a minimal proxy with immutable args deployed that points at an implementation specific to the `FaultDisputeGameFactory.sol`. The two sets of immutable args are first specific to the fault dipuste game, then the second set on the implementation are specific to the rollup.

# Proposed Solution

In order to fix this we need to create a new class of `Creator` contracts for each `FaultDisputeGame`. Each Creator contract is responsible for deploying its own type of `FaultDisputeGame`, and instead `FaultDisputeGameFactory.sol` will no longer have a set implementation it clones from or a `gameType` to implementation mapping. Instead, the `FaultDisputeGameFactory.sol` will be responsible for keeping track of "creators" disputeGame per `gameType`. The main advantage of this is that the information and configuration for each fault dispute game can be granually controlled and adjusted, without forcing it into an unreadable "blob of bytes." Additionally, we can allow each creator to be owned, and so for upgrades, redeployment may not even be necessary, variables like `absolutePrestate` could instead just be simply updated. 
As for whether or not arguments provided by the `Creator` contracts should be immutable, CWIA, or any combination, gas benchmarking will determine whichever approach is best. Additionally, the implementation contract will still retain its current immutable arguments related to configuring the bisection game, only arguments that pertain to a specific chain will be removed from the implementation contract. Additionally, each `Creator` contract will be deployed behind a proxy, which should streamline the upgrade process, as things like `absolutePrestate` can be updated without redeploying the entire contract.

# Alternatives Considered

It would be possible to remove the chain-specific immutable arguments to simply become "extraData" passed into the proxy constructor, however, this complicates the system further because the ultimate goal is to consolidate all of our contracts so that each rollup can point to a single set of implementation contracts, so keeping some configuration parameters on the implementation of `FaultDisputeGame.sol` would not be flexible enough down the line.
Flattening all immutables into a single layer was also considered heavily, but ultimately rejected because it concentrates too much configuration and information in the `DisputeGameFactory.sol`, and places the critical configuration variables in a more unreadable format.
# Risks & Uncertainties

The main risk for this upgrade is now that the factory must be properly configured, otherwise, the factory will produce every single `FaultDisputeGame.sol` incorrectly. Additionally, all risk is now concentrated on a single implementation, which if incorrect would be worse, but overall reduces the risk surface area for fraud proofs.

