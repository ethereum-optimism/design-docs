# Purpose

The process of deploying the fault proof system is suboptimal for a number of reasons. It requires the `OPContractsManager` (OPCM) to redeploy core contracts on a per-chain basis when ideally these contracts would be shared across all chains in the Superchain. This is expensive in terms of gas, and it is overly complex, requiring [core parameters](https://github.com/ethereum-optimism/optimism/blob/c9f845e7839075674f1698e79ec5293bfddb1f8a/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1669-L1672) on these contracts to be configured for every L2 deployed. 

This document recommends reworking this deployment process to decrease complexity and improve efficiency.

# Summary

This proposal recommends modifying the dispute game implementation contracts (`FaultDisputeGame.sol`, `PermissionedDisputeGame`, etc) to remove chain-specific immutable arguments. These parameters would instead be provided during [game creation](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L133-L142) via configuration stored in the `DisputeGameFactory`.

This new approach allows the OPCM to embed a fixed reference to a single canonical game contract implementation per `GameType`. These implementations can then be reused across all L2 chains. This eleminates the need for full game implementation contract deployments for every L2 chain. It also allows core game configuration parameters (such as [splitDepth, maxClockDuration, etc](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L1098-L1116)) to be defined immutably as [part of the OPCM deployment](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1716-L1731).

# Problem Statement + Context

The fault proof system uses a `DisputeGameFactory` contract to deploy new dispute games. This factory stores [a mapping (`gameImpls`)](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L57-L59) from `GameType` to game implementation. When `DisputeGameFactory.create` is called, the factory deploys a game, which is a proxy contract that points to the appropriate `gameImpls` implementation. 

The game implementation contracts (`FaultDisputeGame`, `PermissionedDisputeGame`, etc) currently store chain-specific parameters (like `l2ChainId`, `absolutePrestate`, etc) as immutable variables. This in turn means that in order to [deploy](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1668-L1672), [upgrade](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L794-L802), or [configure](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L476-L484) any given L2 chain, the OPCM must redeploy these game implementation contracts.

When [deploying a new chain](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1668-L1672) or [configuring a new game type](https://github.com/ethereum-optimism/optimism/blob/2850953d58334025b2159a0b1cd946c0c34a72af/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1747-L1750), game implementation constructor arguments must be passed to `OPContractsManager` (OPCM), increasing the risk of misconfiguration. Additionally, redeploying the full contract is expensive in terms of gas. 


# Proposed Solution

This document proposes changes to simplify the deployment of game contracts, enabling reuse of a single implementation per `GameType` across all chains. This is accomplished by removing chain-specific configuration from immutable variables on the game implementation contracts and instead using clone with immutable arguments (CWIA) to pass this data into the game proxy contract at game creation time. 

The `DisputeGameFactory.sol` should be updated to facilitate this. It would include a `gameArgs` mapping (`GameType => bytes`) to store the necessary chain-specific configuration data for each game type. When `create()` is called, the factory would retrieve the appropriate `gameArgs` for the requested `GameType`, concatenate it with the game-specific data (`rootClaim`, `extraData`, etc.), and pass this combined `bytes` payload to the `clone` function.

This approach would allow a single canonical game implementation to be deployed and reused across multiple chains. The factory would handle injecting the chain-specific parameters needed for each instance via the CWIA payload. The implementation contract would still retain immutable arguments related to the core bisection game logic (e.g., `MAX_GAME_DEPTH`, `SPLIT_DEPTH`), but all chain-specific configuration would be externalized to the factory and the cloning process. This would simplify deployment for new chains, requiring only a factory configuration update and proxy deployment rather than a full implementation redeployment.

**Illustrative Code Changes:**

*Example: Removing immutable and adding a getter for an CWIA variable in `FaultDisputeGame.sol`*
```solidity
// Before (Immutable State Variable):
Claim internal immutable ABSOLUTE_PRESTATE_CLAIM;

// After (Removed immutable, added getter to parse the absolute prestate from the CWIA payload):
function absolutePrestateClaim() public pure returns (Claim claim_) {
    claim_ = _getArgBytes32(0xF1); // Example offset/ID for absolutePrestate
}
```

*Example: Adding `gameArgs` and modifying `create` in `DisputeGameFactory.sol`*

```solidity
/// @notice Maps each Game Type to an associated configuration to use with it...
mapping(GameType => bytes) public gameArgs;

function create(GameType _gameType, Claim _rootClaim, bytes calldata _extraData) ... {
    // ... other checks ...

    // Load base constructor args for the implementation into memory
    bytes memory baseConstructorArgs = gameArgs[_gameType];
    // Pack arguments specific to this given instatiation of the game
    bytes memory gameSpecificArgs = abi.encodePacked(msg.sender, _rootClaim, parentHash, _extraData);

    // Clone using concatenated args
    proxy_ = IDisputeGame(address(impl).clone(bytes.concat(gameSpecificArgs, baseConstructorArgs)));
    proxy_.initialize{ value: msg.value }();

    // ... rest of function ...
}

function setImplementation(GameType _gameType, IDisputeGame _impl, bytes calldata _args) external onlyOwner {
    gameImpls[_gameType] = _impl;
    gameArgs[_gameType] = _args; // Store the base arguments
    emit ImplementationSet(address(_impl), _args, _gameType);
}
```

*Example `_args` payload structure (illustrative):*

The `_args` bytes payload passed to `setImplementation` and stored in `gameArgs` would contain the ABI-encoded values for the base configuration parameters read by the `FaultDisputeGame` getters (starting from offset 0x74 in the full CWIA payload). This typically includes:

```solidity
// Example _args construction (conceptual, order based on implementation tests)
bytes memory exampleArgs = abi.encodePacked(
    Claim absolutePrestate,         // bytes32
    IBigStepper vm,                 // address
    IAnchorStateRegistry registry,  // address
);
```

# Alternatives Considered

One alternative considered was to pass all chain-specific configuration purely within the `_extraData` field during game creation. This is similar to the proposed approach but would rely solely on `_extraData` instead of utilizing the structured `gameArgs` stored in the factory alongside the CWIA mechanism. While feasible, encoding all configuration into `_extraData` was deemed less explicit and potentially harder to manage compared to storing base configuration per `GameType` in the factory's `gameArgs` and combining it with instance-specific `_extraData` during cloning.

Another option considered was moving the immutable arguements into a DisputeConfig.sol contract that would store the variables immutably but instead of being an immutable on the implementation it would be immutables stored on a separate contract.  This would be passed as a reference by the config contract address to the initialize function for the FaultDisputeGame.sol proxy contract.  This option was deemed not as optimal as the CWIA solution because it required a larger diff to implement the change.

An earlier proposal involving distinct "Creator" contracts for each `GameType` was also considered but ultimately deemed more complex than necessary. The proposed approach of using `gameArgs` in the factory should provide sufficient flexibility without the added complexity of deploying and managing separate Creator contracts.

# Risks & Uncertainties

The primary risk with this proposed approach lies in the configuration of the `DisputeGameFactory.sol`. Since the factory would be responsible for providing the correct chain-specific parameters (`gameArgs`) for each `GameType`, any misconfiguration (e.g., incorrect addresses, IDs, or improperly encoded `bytes`) would result in incorrectly configured `FaultDisputeGame` proxies being created. Careful deployment and verification procedures for setting `gameArgs` would be crucial.

An additional risk worth noting is the increased cost for deployment of the CWIA proxies due to appending additional bytes data to the end of the proxy. We are increasing the bytecode size of these proxies by roughly 250 bytes.  This implies an increase in deployment cost of about 200 gas / byte * 250 bytes or about 50,000 gas.