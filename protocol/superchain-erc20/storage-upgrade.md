# `OptimismMintableERC20Factory` storage update
## Purpose

*This design doc is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

This document discusses two different approaches to perform the one time storage update of `OptimismMintableERC20Factory` necessary to include tokens deployed before holocene into the `deployments` mapping. It also provides implementation details.

## Summary

To fully implement liquidity migration of `OptimismMintableERC20` tokens to `OptimismSuperchainERC20`, the `OptimismMintableERC20Factory` needs to incorporate a deployment history. The contract changes will cover new deployments, but not the legacy representations. 

There are two paths to populate the `deployments` mapping with a given token list:
- Do a one-time atomic storage manipulation
- Initialize the new factory implementation with a single hash onion value, allowing anyone to uncover the complete token list progressively.

We think the hash-onion approach is cleaner and safer to execute.


## Problem Statement

To become interop compatible, `OptimismMintableERC20` tokens (also known as legacy tokens) will need to be converted into `OptimismSuperchainERC20` tokens. The [Convert design doc](convert.md) covered this liquidity migration process.  

The `L2StandardBridge` will have a `convert()` function that allows anyone to convert back and forth between each `OptimismMintableERC20` and their corresponding `OptimismSuperchainERC20`. The `convert()` function will include safety checks to ensure only trusted representations get burnt and minted. These checks include verifying the input addresses against a `deployments` history mapping living on the official factories, i.e. `OptimismMintableERC20Factory` and `OptimismSuperchainERC20Factory`. 

Each new factory deployment will store the token address in the `deployments` mapping. However, the `OptimismMintableERC20Factory` already has an extensive token history that is not captured. As the `convert()` method will need to catch these legacy tokens, the plan is to populate the `deployments` mapping with a pre-approved token list.

## Alternatives considered

This section will introduce two methods to populate the `deployments` mapping. It will NOT discuss how to create the token list.

### 1. Full storage surgery
Full storage surgery consists of atomically replacing the factory implementation for bytecode that allows the writing of any value to any storage slot, updating the complete list, and then replacing the implementation again for the final one.

Detailed flow:
- `ProxyAdmin` owner atomically upgrades the [`OptimismMintableERC20Factory`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/universal/OptimismMintableERC20Factory.sol) predeploy to the [`StorageSetter` contract](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/universal/StorageSetter.sol) (replace implementation in the Proxy).
	- On OP Mainnet, the `ProxyAdmin` owner is the aliased L1 governance multisig.
- Sets the storage slots corresponding to each local token within the `deposits` mapping to the remote token address.  
- Upgrade the `OptimismMintableERC20Factory` to the new final implementation.

This path requires to pre-compute the storage slots for the mappings.

### 2. Hash onion
The hash onion approach involves performing a single storage update on the new implementation initialization and then letting users populate the full `deployments` mapping with a simple cryptographic proof. This implies adding a public function to the factory.

Detailed flow:
Upgrade the `OptimismMintableERC20Factory` to the new final implementation.
	- Implementation should include an `initialize()` function to set a single storage slot to a selected value.
	- It is also possible to perform the single storage update with the atomic migration to the `StorageSetter` implementation, but it seems like an overkill.
- On the same transaction, call `initialize()` to set a single storage slot `hashOnion` to a hash onion structure representing all the tokens.
    - Each chain will have its token list and, therefore, `hashOnion` value.
- (Optional) After the whole data set has been reconstructed, the next upgrade can remove the now-useless function.

The final `OptimismMintableERC20Factory` should include a public function that allows anyone to provide subsequent openings for the `hashOnion` value and populate the deployments mapping permissionlessly. 

To execute this path, it is necessary to modify the `OptimismMintableERC20Factory` contract to handle the `hashOnion` logic.

Note: a similar implementation could be achieved with a Merkle root instead of a hash onion. Verifying and storing is easier in a hash onion, as it provides a clear ordering (automatically erases duplicates). On the other hand, a user who wants to convert with a legacy token will need to first unlock all previous layers in a hash onion, whereas in a Merkle tree, proofs are independent.

## Tradeoffs and conclusion
Pros of hash onion compared to full storage update:
- requires a single implementation change in the proxy contract instead of the two necessary for the full storage upgrade.
- doesn't require to precompute the storage slot for each element of the `deployments` mapping. This step should be straightforward, but any mistake requires a new upgrade.
- easier to check a single value per chain compared to a token list per chain. 
- fewer variables and, therefore, risk in the upgrade process.

Cons of hash onion compared to full storage update:
- after the `deployments` mapping is fully populated, the factory will have useless code. This can later be removed in a subsequent upgrade.
- it requires user interaction. In particular, users might be unable to convert a given token unless its address has already been proven in the factory. This action might be reserved for power users.
- less contract code, reducing on-chain risk.

All things considered, the hash onion approach optimizes the upgrade process's security and scalability. Unused code should be a minor downside, which can later be addressed in future upgrades. Solid documentation and UI tools can minimize the risk of users getting stuck. 

## Hash onion implementation details
The idea of the hash onion is to commit to a data set with a single `bytes32` value. There are multiple ways to do so. In what follows, a simple reference implementation will be presented.

Given a dataset `[d1, d2, ..., dn]`, the `hashOnion` will be built with the subsequent hashing of each data and the previous *layer* as follows:
```
layer0 = keccak256(d1)
layer1 = keccak256(abi.encodePacked(layer0, d2))
...
hashOnion = layerN = keccak256(abi.encodePacked(layer(N-1), dN))
```

This operation compromises $N$ hashes. 
The contract update will store the final `hashOnion`, and users will call a public function to reconstruct the whole dataset.

Example implementation:
```solidity
/**
 * @dev Verifies the previous layer and data, stores addresses in the mapping,
 * and updates the last layer.
 * @param localToken The local token address to be stored
 * @param remoteToken The remote token address to be stored
 * @param previousLayer The previous layer of the hash onion
 */
function verifyAndStore(
	address localToken,
	address remoteToken,
	bytes32 previousLayer
) public {
	// Concatenate localToken and remoteToken using abi.encodePacked
	bytes memory data = abi.encodePacked(localToken, remoteToken);

	// Recompute the hash of the previous layer + data
	bytes32 computedHash = keccak256(abi.encodePacked(previousLayer, data));

	// Ensure the computed hash matches the current stored layer
	require(computedHash == hashOnion, "Invalid layer provided.");

	// Store the localToken and remoteToken in the mapping
	deployments[localToken] = remoteToken;

	// Emit event for logging
	emit AddressesStored(localToken, remoteToken);

	// Update the current layer to the previous one for the next submission
	hashOnion = previousLayer;
}
```

It is possible to take `d1 = 0` to make it easier to determine when the onion has been fully uncovered.

## What needs to be done?

Every upgrade should go through the [superchain-ops](https://github.com/ethereum-optimism/superchain-ops) process. 

Actionables are the following:
- Modify the `OptimismMintableERC20Factory` contract to handle the `hashOnion` logic.
	- Add the single storage update to `initialize()`.
		- Select a safe storage slot.
	- Add the `verifyAndStore()` public function to the specs and implementation.
- Create a repository containing scripts to build each chain's hash onion given a token list. 
- Have a task written in `superchain-ops` that performs the upgrade. 