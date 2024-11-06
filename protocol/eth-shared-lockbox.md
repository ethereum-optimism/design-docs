# Purpose

*The document presented below is part of the [Interoperability project](https://github.com/orgs/ethereum-optimism/projects/71/views/1?sliceBy%5Bvalue%5D=skeletor-spaceman).*

This document discusses possible solutions to address the constraints on ETH withdrawals resulting from the introduction of interop and `SuperchainWETH` that share ETH liquidity across a set of interoperable OP Chains.

# Summary

With interoperable ETH, withdrawals may fail if the referenced `OptimismPortal` lacks sufficient ETH—especially for large amounts or on OP Chains with low liquidity—since ETH liquidity isn't shared at the L1 level. To prevent users from being stuck, several solutions have been explored. Currently, the Superchain design favors the `SharedLockbox` and the `DependencySetManager` as the most effective solution.

# Problem Statement + Context

With the introduction of interop, `SuperchainWETH` is created as a medium to move ETH across a set of interoperable chains, instead of using native ETH. To enable users to convert `SuperchainWETH` into native ETH, `ETHLiquidity` is also introduced, containing a sufficiently large virtual pool of native ETH, which can only be used for this purpose. As a result, ETH can be shared through all interconnected chains.

However, one remaining problem to solve relates to [ETH withdrawals](https://github.com/ethereum-optimism/specs/issues/362). Currently, with Interop, the amount of ETH on an L2 can differ from what is deposited in the respective `OptimismPortal`. This mismatch can interfere with the finalization of withdrawals, especially if a request requires more ETH than is actually deposited at a given time.

This means a solution is needed to achieve a truly shared `SuperchainWETH` via shared ETH liquidity given an interoperable set of chains.

# Prior considerations

Before explaining the proposed solution, it is important to note that there are essential assumptions that underpin the whole design: the Shared Security Model, Unified Chain Governance and Superchain WETH activation.

### Shared Security Model

The security model for the set of interoperable chains (commonly called a "cluster") is shared across all involved chains, ensuring that all state transitions are secured. For example, the shared security model allows for defending against any maliciously claimed state transition for any chain within the cluster. This is accomplished through a set of security features at the proof level, such as permissionless proposing, interop-provable proofs (also called shared proofs), and the presence of a single Guardian role across the entire system. This model is independent of the proof system used (e.g., ZK, fault proofs, etc.).

### Unified Chain Governance

OP Chains will be governed within a common Chain Cluster governance entity (the Collective). Currently, this entity is responsible for:

- **Ensuring chains are consistent at the implementation side**, achieved either through trusted deployment methods (e.g., OP Contracts Manager) or by approval after security checks are performed.
- **Approving protocol upgrades** for all chains involved.
- **Adding new chains** to the interoperable set, and managing the `dependencyManager` role. The final decision is up to governance and chains cannot be removed afterwards.
- **Replacing chain servicers** for existing chains if they fail to satisfy technical requirements.

### Shared Bridging and SuperchainWETH usage

In any OP Chain that joins the cluster, it is assumed that `SuperchainWETH` has been deployed in advance before being added to the interoperable graph. As a result, the equivalence between ETH deposits and withdrawal history and the actual ETH supply will vary from the outset. In a world with freedom of movement, all real ETH liquidity is theoretically shared across the entire cluster eventually, regardless of how deposits are handled.

# Solution

The existing problem and considerations motivate us to propose an L1 shared liquidity design through the introduction of a new `SharedLockbox` on L1, which serves as a singleton contract for ETH, given a defined set of interoperable chains. To ensure consistency, the `DependencySetManager` is also introduced to manage the dependency set that the lockbox uses as a source of truth. New ETH deposits will be directed to the lockbox, with the same process applying to ETH withdrawals.

## Spec changes

The core changes proposed are as follows:

- Introduce the `DependencySetManager` contract: This contract manages and tracks the dependency graph.
- Introduce the `SharedLockbox` contract: It acts as an escrow for ETH, receiving deposits and allows withdrawals from approved `OptimismPortal2` contracts. The `DependencySetManager` contract serves as the source of truth of the lockbox.
- Modify the `SystemConfigInterop`: Add `addDependencies` and `removeDependencies` functions to accept an array of `_chainId` values.
- Modify the `L1BlockInterop`: To maintain consistency with `SystemConfigInterop`.
- Modify the `OptimismPortal2`/`OptimismPortalInterop`: To forward ETH into the `SharedLockbox` when `depositTransaction` is called, with the same process applying when `finalizeWithdrawal` is called.

### Managing `DependencySetManager`

This contract serves as the single point for managing the dependency set of a cluster and is expected to be managed by an admin in the same manner as other L1 OP contracts (proxiable). This contract assumes the role of `dependencyManager` for every `SystemConfigInterop` contract involved. In the case of a simple dependency, the `DependencySetManager` only stores a mapping (or array) of chains added, e.g. given a `chainId`.

Adding a new chain can be done as follows:

1. `registerChain` is called, which adds the chain to the registry (`chainId` value, `SystemConfig` address, `OptimismPortal2` address) but does not add it to the dependency set. The [Superchain Registry](https://github.com/ethereum-optimism/superchain-registry) or the [OP Contracts Manager](https://specs.optimism.io/experimental/op-contracts-manager.html?highlight=chain#chain-id-source-of-truth) can be used as the source of truth to add chains that have been verified as compatible with this integration.
2. An OP-governed address, who has the role to call `addChain` with the new `chainId`.
    1. `addChain` then calls each `SystemConfigInterop` in the existing `dependencySet` list.
        1. For the chain being added, it triggers `addDependency` with `chainId1`, `chainId2`, … `chainIdn`.
        2. For all existing chains, it triggers `addDependency` with `chainId` as the newly added chain.
    2. Updates the dependency graph (mapping or array).
    3. Emits an event for step (1).

A code example for step (2) would look like this:

```solidity
// Mapping from chainId to SystemConfigInterop address
mapping(uint256 => address) public systemConfigInterops;
// Mapping to check if a chainId is in the dependency set
mapping(uint256 => bool) public alreadyAdded;
// Current dependency set list
uint256[] public dependencySet;

// Function to add a chain to the dependency set
function addChain(uint256 chainId) external onlyOwner {
		require(systemConfigInterops[chainId] != address(0), "DependencySetManager: Chain not registered");
		require(!isRegistered[chainId], "DependencySetManager: Chain already in dependency set");

		// For the chain being added, call addDependencies with existing chainIds
		if (dependencySet.length > 0) {
				for (uint256 i = 0; i < dependencySet.length; i++) {
            ISystemConfigInterop(systemConfigInterops[chainId]).addDependency(dependencySet[i]);
            }
		}

		// For each existing chain, call addDependencies with the new chainId
		for (uint256 i = 0; i < dependencySet.length; i++) {
				ISystemConfigInterop(systemConfigInterops[dependencySet[i]]).addDependency(chainId);
    }
    // Update the dependency set
    dependencySet.push(chainId);
    isRegistered[chainId] = true;

    emit ChainAdded(chainId);
}
    
```

For `removeChain`, the process would follows the same logic.

Note that, under the specified flow, the dependency set consistently maintains the form of a [complete graph](https://en.wikipedia.org/wiki/Complete_graph) at all times.

### `addDependencies` and `removeDependencies` in `SystemConfigInterop` and `L1BlockInterop`

The purpose of these functions is to extend the current `SystemConfigInterop` implementation by allowing multiple `chainId` values to be added in a single call by passing them as an array. There are a few ways to pass these values through `setConfig`: one is making a single call and triggering the `TransactionDeposited` event, or by looping through the current `addDependency`/`removeDependency` functions.

With this optimization, the `DependencySetManager` can simplify the number of calls, as follows:

```solidity
function addChain(uint256 chainId) external onlyOwner {
  // ...
	if (dependencySet.length > 0) {
				// If we can input an array chainId values thorugh addDependencies function
				ISystemConfigInterop(newChainSystemConfig).addDependencies(dependencySet);
	}		
	// ...
```

### `SharedLockbox` implementation

A minimal set of functions should include:

- `lockETH`: Accepts ETH from the `depositTransaction` originating from a valid `OptimismPortal2`.
- `unlockETH`: Releases ETH from the `finalizeWithdrawalTransaction` originating from a valid `OptimismPortal2`.

Access control for `lockETH` and `unlockETH` is validated against the list reated by `registerChain`.

### `OptimismPortal2` upgrade process

A one-time L1 liquidity migration is required for each approved chain. By using an intermediate contract during the upgrade, all ETH held by the `OptimismPortal` can be transferred to the `SharedLockbox` within the `initialize` function. After this migration, the `OptimismPortal` is upgraded to the new version with the desired functionality.

Importantly, the entire upgrade process—including migrating the ETH to the `SharedLockbox` and updating the `OptimismPortal` to the latest version—can be executed in a single transaction. This approach ensures a migration without the necessity of maintaining a persistent migration function in the final contract.

Only chains registered in the `DependencySetManager` should be approved to join the `SharedLockbox`. Once approved, the `SharedLockbox` address is added in the `constructor` and `initialize` functions for each `OptimismPortal2`.

## Impact

The following components require an audit of the new and modified contracts: `OptimismPortal2`, `SharedLockbox`, `SystemConfigInterop`, `L1BlockInterop`, and `DependencySetManager`. This also includes the scripts necessary to perform the migration.

# Alternatives Considered

### Reverts on L2

One alternative is not to share liquidity at the L1 level and instead prevent withdrawals from being stuck by reverting them on L2. This approach requires tracking the exact ETH balance in L2 via deposits and withdrawals. An `ETHBalance` counter would increment with new mints from deposits and decrease with successful `initiateWithdrawal` calls.

This method would require minimal changes to `L1Block` and adjustments to how `TransactionDeposited` is processed to validate new ETH counts. Additionally, it necessitates porting the initial balance as part of an upgrade transaction.

The problem with L2 reverts is that it breaks the ETH withdrawal guarantee invariant and still exposes the system to solvency issues. Based on previous feedback, this would affect existing applications and pre-deployed contracts such as the current `FeeVault` withdrawal contract logic.

### **Permission to withdraw ETH from a different Portal**

[Another solution](https://github.com/ethereum-optimism/specs/issues/362#issuecomment-2332481041) involves allowing ETH withdrawals to be finalized by taking ETH from one or more `OptimismPortal` contracts. In a cluster, this is done by authorizing withdrawals across the set of chains. For example, if  we have a dependency set composed by Chain A, B and C, a withdrawal initiated from A could be finalized by using funds from B and C if needed.

The implementation would require iterating through the dependency set, determined on L1 to find the next `OptimismPortal` with available funds. This approach incurs more modifications to the `OptimismPortal`, increasing complexity.

# Risks & Uncertainties

- **Scalable security**: With interop, withdrawals are a critical flow to protect, especially for ETH, since it tentatively becomes the most bridged asset across the Superchain. This means proof systems, dedicated monitoring services, the Security Council, and the Guardian need to be proven to tolerate the growing number of chains.
- **Necessity of gas optimizations**: the `DependencySetManager`’s `addChain` and `removeChain` functions call every `SystemConfigInterop`, which will increase in number over time. This would lead to significant gas expenditure as the number of chains continues to grow.
- **Chain list consistency around OP contracts**: OP Chains can have different statuses over time. This is reflected by the potential presence of several lists, such as those in the `OPCM` and the `DependencySetManager`. It would make sense to coordinate on implementing the most ideal chain registry for all expected use cases, including those described in this doc.
