# Purpose

The current fault-proof system requires a redeployment of the `DisputeGame.sol`, `DisputeGameFactory.sol`, and `AnchorStateRegistry.sol` for new L2 chains due to chain-specific immutable variables (like `anchorStateRegistry`, `absolutePrestate`) in their constructor. This document proposes changes to simplify the Fault Dispute Game contract setup, enabling reuse of a single implementation across chains by moving the chain-specific configuration from immutable variables stored as immutable variables to variables stored as part of the payload for clones with immutable args (CWIA).

# Summary

This proposal recommends modifying the `FaultDisputeGame.sol` contract to remove chain-specific immutable arguments. These parameters would instead be provided during proxy creation via a `bytes` payload constructed by the `DisputeGameFactory.sol`. The factory should be updated to store base configuration per `GameType` in a `gameArgs` mapping and combine it with game-specific data (`rootClaim`, `extraData`) at creation time of the proxy contract. This would allow new chain deployments to use proxies pointing to a canonical implementation, configured by the factory, eliminating the need for full contract redeployments.

# Problem Statement + Context

The Fault Dispute Game system relies on several core contracts, including `DisputeGameFactory.sol` and `FaultDisputeGame.sol`. Currently, the `FaultDisputeGame.sol` implementation contains immutable arguments specific to a particular chain deployment (e.g., WETH address, L2 Chain ID). This requires deploying a new `FaultDisputeGame.sol` implementation and, consequently, a new `DisputeGameFactory.sol` for each chain.

The goal is to allow a single, canonical `FaultDisputeGame.sol` implementation to be reused, with chain-specific parameters injected during the proxy creation process, simplifying deployments and upgrades.

An important nuance here is that each `FaultDisputeGame.sol` deployment has both a minimal proxy with immutable args deployed that points at an implementation specific to the `FaultDisputeGameFactory.sol`. The two sets of immutable args are first specific to the fault dispute game, then the second set on the implementation are specific to the rollup.

# Proposed Solution

The `FaultDisputeGame.sol` contract be modified to remove chain-specific immutable variables (like `WETH`, `ANCHOR_STATE_REGISTRY`, `L2_CHAIN_ID`, etc.). Instead, these configuration parameters should be read dynamically from the `clones-with-immutable-args` (CWIA) data payload.

The `DisputeGameFactory.sol` should be updated to facilitate this. It would include a `gameArgs` mapping (`GameType => bytes`) to store the necessary chain-specific configuration data for each game type. When `create()` is called, the factory would retrieve the appropriate `gameArgs` for the requested `GameType`, concatenate it with the game-specific data (`rootClaim`, `extraData`, etc.), and pass this combined `bytes` payload to the `clone` function.

This approach would allow a single canonical `FaultDisputeGame` implementation to be deployed and reused across multiple chains or configurations. The factory would handle injecting the specific parameters needed for each instance via the CWIA payload. The implementation contract would still retain immutable arguments related to the core bisection game logic (e.g., `MAX_GAME_DEPTH`, `SPLIT_DEPTH`), but all chain-specific configuration would be externalized to the factory and the cloning process. This would simplify deployment for new chains, requiring only a factory configuration update and proxy deployment rather than a full implementation redeployment.

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
/// @notice Maps each Game Type to an assosiated configuration to use with it...
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