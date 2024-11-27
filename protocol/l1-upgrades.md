# Purpose

Our current upgrade process is onerous and error prone, requiring a significant amount of effort on
the part of both the developer and signers. This process is not scalable enough for the needs of
regular superchain wide upgrades.

This document outlines an improved and standardized approach to upgrading L1 contracts.

## Goals

We take the following as our design goals:

- **Atomicity:** It should be possible to complete an upgrade of the entire superchain's L1
  contracts within one block.
- **Declarative design:** As much as possible, a developer should be able to simply indicate what
  bytecode should be deployed, and what storage values (if any) should be initialized or modified.
- **Deterministic upgrade calldata:** The exact calldata to be submitted during the upgrade should
  predictable at the time of the governance proposal.
- **Safety:** Above all the upgrade must be safe, particularly with respect to the things which can
  typically go wrong with an upgrade, ie.
  - All contracts should be upgraded to the correct implementation.
  - Storage layouts should not be incorrectly modified.
  - Contracts should not be 're-initializable'
- **Standardization:** We avoid trying to account for all special cases, and instead require contracts to
  be implemented in a manner compatible with the solution outlined here.
- **Simplicity:** We avoid adding conventions which will require frequent conversations starting with "ok here's how it works..."
- **Auditability:** An upgrade path should be easy to understand, either by reviewing the source
  code as in the case of a forthcoming upgrade, or by reviewing the on-chain history along with
  verified source code.

## Non-goals

This document is not concerned with:

1. upgrading predeploys or other L2 contracts.
1. modifications to the ownership system, ie, it does not attempt to modify Safes, modules or
   guards.

With the exception of dispute game implementations this work will either include, or else depend on,
all L1 contracts becoming either:

1. Proxied and MCP ready
2. Superchain-shared contracts (proxied or not)

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

Key points of the proposed L1 contract upgrade system:

- Uses a two step upgrade flow, using a standard unstructured slot for `initialized`
- Adds an `upgrade() initializer` function to each L1 contract
- Consolidates `ProxyAdmin` contracts into a single superchain wide ProxyAdmin for all L1 contracts
- Performs an upgrade based entirely on calldata inputs, without reading or modify state
- Records the `op-contracts` release version in the OPCM
- Outlines special considerations for dispute game contracts

# Problem Statement + Context

Our upgrades have the following issues:

1. They are divorced from the implementation, meaning the work of writing the upgrade script occurs
   in `superchain-ops` apart from the work of writing the updated code in the `optimism` monorepo.
2. The act of writing the upgrade script is onerous, requiring:
   1. A bespoke script with custom assertions.
   2. A bespoke Validations.md document
3. The work required to perform the upgrade is difficult for signers, as they must work through the
   Validations.md document. This work has taken up to an hour in the past.
4. The effort described above will scale linearly with the number of OP Chains.

# Proposed Solution

Each new release will have its own OP Contracts Manager which can perform the following distinct
actions:

1. Deploy new proxies for a new OP Chain (this work is already covered by a recent [OPCM redesign](./op-contracts-manager-single-release-redesign.md)).
1. Upgrade all existing OP Chains.
1. Deploy any newly added superchain-shared contracts.
1. Upgrade existing superchain-shared contracts.

Note that the above functionality also makes for a good set of milestones, each of which can and
should be implemented incrementally.

## Modfications to L1 contracts

Taking inspiration from Chugsplash, we wish to separate the concern of what bytecode to run, and
what to write to storage. However we also wish to avoid using the current `StorageSetter` pattern,
which simply provides logic to "write anything anywhere" in storage.

In order to achieve this, each contract will:

1. Use the OpenZeppelin v5 Initializable contract, which stores the `initialized` value in an
   [unstructured storage location](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.2/contracts/proxy/utils/Initializable.sol#L77).
2. Have a new `upgrade()` method added alongside the existing `initialize()` method, both of which
   will have the `initializer` modifier

By way of example, we take an initializable version of `SimpleStorage`:

- the current version of the contract has only a `uint256 foo` variable.
- the new version of the contract introduces a `bytes32 bar` variable.

The status quo implementation of the new Simple Storage would look like this:

```solidity
contract SimpleStorage is Initializable {
  uint public foo;
  bytes32 public bar; // new variable added for the current release

  constructor() {
    _disableInitializers();
  }

  /// @notice initialize is called whether for a fresh system or when upgrading an existing system.
  function initialize(uint _foo, bytes32 _bar) public initializer {
    // chains being upgraded need to read the current value of foo, then write it again.
    foo = _foo;
    bar = _bar;
  }
}
```

The only change to this contract will be to:

1. store the `initialized` value in an unstructured storage location.
2. add a new `upgrade()` function which also has the `initializer` modifier.

```solidity
/// @notice Called to upgrade a pre-existing system.
///         If new values are not being added to the storage layout, this function can be empty
///         and take no arguments.
function upgrade(bytes32 _bar) public initializer {
  bar = _bar;
}
```

The `StorageSetter` contract can then be replaced with a special purpose contract which
directly resets the `initialized` slot.

The new contract **deployment** flow used by the OPCM thus becomes:

1. Deploy a new proxy.
2. Set its implementation to `SimpleStorage`.
3. Call `SimpleStorage.initialize()`.

And the contract **upgrade** flow used by the OPCM would be:

1. set its implementation to an `InitializerResetter` contract (does what the name says).
2. call `InitializerResetter.reset()`
3. set its implementation to `SimpleStorage`.
4. Call `SimpleStorage.upgrade()`.

**Alternatives considered:** Other approaches were considered including:

1. Remove the two step upgrade approach. This would have required using a `reinitializer` like pattern
   which was deemed more complex and error prone.
2. Separating the `initializer()` and `upgrade()` methods into a separate contract used in the
   first part of a two step upgrade. This was deemed too complex as it created more contracts to manage.
3. Breaking out the storage layout into a separate contract, this would have created a complex refactor
   and inheritance tree.
4. Using `StorageSetter` directly during the first step, in place of an `upgrade()` method. This approach
   would not allow us to easily perform sanity checks on inputs without duplicating logic into the OPCM.

## A single Superchain wide `ProxyAdmin`

All contracts will have their upgrade auth consolidated into a single Superchain-wide `ProxyAdmin`.

This will require a transfer of ownership of every single L1 contract. However these risks can be
mitigated, and the benefits of reduced complexity are believed to be worthwhile, and better done
sooner than later.

The ownership transfer operation should be handled by the OPCM itself. The success of the
ownership transfer should be verified after execution, and the transfer should revert unless
successful.

A separate design doc will be created to document this process.

**Alternatives considered:** Other approaches were considered including:

1. Keeping many `ProxyAdmin`s: this is complicated and expensive.
2. Making the `SuperchainProxyAdmin` an even thinner wrapper which effectively just forwards
   calldata. The approach chosen will reduce the amount of boilerplate in the OPCM.

## Identifying OP Chains by the SystemConfig

We will use the `SystemConfig` as the unique identifier for each OPChain, and get the contract
addresses from it. The custom logic required for the two non-standard proxy types will be handled in
the OPCM contract, and does not require tracking the proxy type (as currently done in the
`ProxyAdmin`), since it is known already for each contract, ie. the only non-standard Proxies are
the `L1StandardBridge` (`L1ChugsplashProxy`) and `L1CrossDomainMessenger` (`ResolvedDelegateProxy`).

## Contract versioning

Existing `version()` getters will continue to work as normal.

The OPCM will have a `version()` getter with a version matching `X.Y.Z` in the `op-contracts/vX.Y.Z` release
tag.

The `DeployImplementations` script will write the version into the OPCM, along with an `rc` tag. The
`rc` tag will be programmatically removed when `upgrade()` is called by the Upgrade Controller
multisig, which signifies governance approval.

## Proof system considerations

### Adding the PermissionlessDisputeGame to a chain

Given that not all chains will support the `PermissionlessDisputeGame` upon deployment, an
`OPCM.addGameType()` method will be added which will orchestrate the actions required to add a
new game type.

This method will:

1. deploy the `FaultDisputeGame` contract
2. setup the `DelayedWethProxy` for the new game
3. Reinitialize the `AnchorStateRegistry` to add the new game type.

### Making `op-program` shared

`op-program` will be updated by [other work](https://github.com/ethereum-optimism/design-docs/pull/161) to remove the chain config, so that it can be shared across chains.

### Short term allowances for non-compliant contracts

In order to avoid blocking this work, if the `FaultDisputeGame` and `PermissionedDisputeGame`
have not become MCP-L1 compatible in time, the OPCM upgrade path will deploy chain specific instances.

## Upgrade process

As part of the release process, the associated implementation contracts and a new OPCM, will be deployed.

The Upgrade Controller (which MUST be a Safe), will `DELEGATECALL` the `OPCM.upgrade()` function,
providing the address of the `SuperchainProxyAdmin` and a list of the `SystemConfig` contracts for the
OP Chains to be upgraded.

Thus the high level logic for upgrading a contract should be roughly as follows:

```solidity
// Some upgrades will not require any new values
struct NewChainConfig {
  address newValues;
  address disputeGameBlueprint;
}

Addresses immutable implementations;

constructor(Addresses _implementations) {
  implementations = _implementations;
}

/// @notice This function is intended to be DELEGATECALLed by the chain's Upgrade Controller, therefore it
///         must not read or write from storage, and should receive all required data as calldata.
function upgrade(SuperchainProxyAdmin _admin, ISystemConfig[] _systemConfigs, NewChainConfig[] _newConfigs) public {
  for(uint i=0; i< systemConfigs.length; i++) {
    // Read the `Addresses` struct from each `SystemConfig` in the `systemConfigs` mapping.
    // For each entry in the `Addresses` struct:
      // 1. Call the appropriate `SuperchainProxyAdmin.upgradeAndCall()` function to reset the initialized slot.
      // 2. Call the appropriate `SuperchainProxyAdmin.upgradeAndCall()` function to update the
      //    implementation and call `upgrade()` on the contract.
      // 3. For non-MCP compliant dispute game contracts, call `setImplementation()` and update the `AnchorStateRegistry`.
  }
  // run safety assertions to validate the upgrade
}
```

To enumerate the full flow:

1. Deploy new OPCM contract, which contains:
   1. `deploy()` and `upgrade()` methods to deploy new chains and ugprade existing chains
   1. immutables pointing to all new implementation contracts and shared config (e.g. a new contract).
1. For each new implementation an upgrade call will be executed via the `SuperchainProxyAdmin`
1. A Safe will `DELEGATECALL` to the `OPCM.upgrade()` method, which MUST be `pure` for safety purposes.
1. A two step upgrade is used where the first upgrade is to an `InitializerResetter` which resets
   the `initialized` value, then the implementation is updated to the final address and `upgrade()`
   is called.

   **Notes:**

- A new getter should be added to the `SystemConfig` to retrieve the `Addresses` in a single call.
- The `AddressManager` for each chain must be used to upgrade the `L1CrossDomainMessenger`, therefore it should be added to the `Addresses` struct.
- We should identify measures for ensuring the OPCM is nearly stateless, and the `upgrade()` function
  should not read or write from its own storage.

Importantly, this design does not provide special affordances to enable systems with different
upgrade controllers to upgrade together atomically, although this could be achieved with additional
coordination.

## Upgrading multiple versions

All systems MUST undergo the same upgrade path, using each OPCM in order.

For example, if the latest version is `2.2.0`, and a system is on `2.0.0`, then it will first need to
be upgraded to `2.1.0` using the OPCM for that version.

The address of the OPCM corresponding to each upgrade will be stored in the `superchain-registry`,
and `op-deployer` will ensure that upgrades conform to the specified ordering.

## Development and Testing considerations

This solution should be implemented such that it both enables and _requires_ developers to implement
the upgrade path from the most recent release to the one curently being developed. This means it
will be necessary to have a deployment of the previous release available on the current commit, in
order to test the upgrade path. This is achievable using `op-deployer`.

All tests should be executed against both a freshly deployed and upgraded system.

**Alternatives considered:**

- Rather than duplicated tests, new tests could ensure that given the same configuration as inputs,
  a fresh chain has the same resulting configuration as an upgraded chain. Duplicating tests was
  deemed acceptable as they are parallelizable.

# Resource Usage

A rough per-chain estimate (accounting for lookups but not memory) based on the implementation described in [the upgrade process](#the-upgrade-process):

- Cold `SLOAD` the `SystemConfig` address: 2100
- Cold `CALL` the `SystemConfig` address: 2600
  - Cold `SLOAD` the `Addresses` struct (excluding `GasPayingToken`): 6 x 2100
- `CALL` the `SuperchainProxyAdmin`: 2600
  - Do the following 6 times:
    - `CALL` the Proxy: 2600
    - Cold `SSTORE` the temporary `InitializerResetter` implementation address over a non-zero slot: 5,000
    - Cold `SSTORE` a zero value: 2900
    - Warm `CALL` the Proxy again: 100
    - Warm `SSTORE` the final implementation address over a non-zero slot: 100
    - Warm `CALL` the Proxy again: 100
    - Cold `SSTORE` new values: 20,000 each

Aggregating the above slightly:

1. Per chain overhead: 2100 + 2600 + 6 x 2100 = 17,300
2. Per contract per chain: 2600 + 5000 + 100 + 100 = 7,800
3. Cost per new value in storage: 20,000

Since new values in storage are fairly rare, with the possible exception of the System Config, we'll assume just one new value per-chain, giving a cost of: 17,300 + 6\*7,800 + 20,000 = 84,100

This is a lower bound, but if we give a healthy margin of 3x,for
about 250,000 per chain upgraded, we can fit 120 OP Chain upgrades
inside an L1 block.

We will likely need to handle that limitation at some point in the future, but can defer that for the time being.

# Alternatives Considered

Where relevant, alternatives considered are documented above in each section.

# Risks & Uncertainties

1. **Storage Layout Risks**: There are risks around incorrect storage layout modifications during upgrades, which could corrupt contract state if not handled properly.

2. **Re-initialization Risk**: Contracts could potentially be re-initialized if the initialization state is not properly managed. The proposed solution mitigates this by using OpenZeppelin v5's unstructured storage for initialization state.

3. **Gas Cost Uncertainty**: While estimates suggest ~250,000 gas per chain upgrade would allow 120 OP Chain upgrades per L1 block, actual gas costs could vary significantly:

   - Cold storage access patterns may differ from estimates
   - Number of new storage values needed could exceed assumptions
   - Future L1 gas cost changes could impact upgrade capacity

4. **Testing Coverage**: The dual testing approach (fresh deploy vs upgrade) adds complexity:

   - Need to maintain previous release state representations
   - Must ensure test coverage is equivalent for both paths
   - Parallel test execution could mask upgrade-specific issues

5. **Implementation Complexity**: The new upgrade pattern requires:

   - Careful coordination between implementation and upgrade code
   - Strict adherence to standardized contract modifications
   - Proper handling of the initialization state across all contracts

6. **Scale Limitations**: While current estimates support 120 chain upgrades per block, this may become insufficient as the number of OP Chains grows, potentially requiring:
   - Breaking upgrades across multiple blocks
   - More sophisticated batching mechanisms
   - Additional gas optimizations
