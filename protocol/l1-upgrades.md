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
- **Simplicity:** We avoid trying to account for all special cases, and instead require contracts to
  be implemented in a manner compatible with the solution outlined here.
- **Auditability:** An upgrade path should be easy to understand, either by reviewing the source
  code as in the case of a forthcoming upgrade, or by reviewing the on-chain history along with
  verified source code.

## Non-goals

This document is not concerned with:

1. upgrading predeploys or other L2 contracts.
1. modifications to the ownership system, ie, it does not attempt to modify Safes, modules or
   guards.

More over, it does not present a plan for managing contracts which are not proxied, and not
Multichain Prepared (MCP-L1) contracts (ie. FaultDisputeGame, AnchorStateRegistry). This
work will either include, or else depend on, all L1 contracts becoming either:

1. Proxied and MCP ready
2. Superchain-shared contracts (proxied or not)

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

The proposed solution will perform upgrades by transferring upgrade authorization of all L1
contracts to the release specific OPCM. Upon completion of the upgrade, this authorization will
be returned to the Security Council.

Rather than the curent `StorageSetter` based approach which simply deletes the `_initialized` slot,
allowing for `initialize()` to be called, we take this two-step upgrade further, with a short-lived
`Populator` contract being temporarily set as the implementation in order to set or modify any
values as required. This has two significant benefits. First it removes the need to keep a one-time
`initialize()` function in the run-time logic. Secondly it enables a clean separation of concerns
between contracts which are newly deployed (requiring all values to be set), and contracts being
upgraded (requiring only new values to be set).

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

1. Deploy new proxies for a new OP Chain (this work is already covered by [link to Blaine's doc]())
1. Upgrade all existing OP Chains.
1. Deploy any newly added superchain-shared contracts.
1. Upgrade existing superchain-shared contracts.

Note that the above functionality also makes for a good set of milestones, each of which can and
should be implemented incrementally.

## Declarative storage management

Taking inspiration from Chugsplash, we wish to separate the concern of what bytecode to run, and
what to write to storage. However we also wish to avoid using the current `StorageSetter` pattern,
which simply provides logic to "write anything anywhere" in storage.

In order to achieve this, each contract will be refactored to inherit from a `__Layout` contract. By
way of example, we take an initializable version of `SimpleStorage`, and refactor it into the set of
contracts which would be required to specify a hypothetical Magma release, which adds a new variable
to the contract.

```solidity
contract SimpleStorage is Initializable {
  uint public foo;
  bytes32 public bar; // new variable added for the Magma release.


  /// @notice initialize is called whether for a fresh system or when upgrading an existing system.
  function initialize(uint _foo, bytes32 bar) public initializer {
    // chains being upgraded need to read the current value of foo, then write it again.
    foo = _foo;
    bar = _bar;
  }
}
```

This `SimpleStorage` will become:

```solidity
abstract contract SimpleStorageLayout {
  uint public foo;
  bytes32 public bar;
}

contract SimpleStoragePopulator is SimpleStorageLayout {
  function initialize() public {
    foo = 1;
    bar = 2_000
  }

  function upgrade() public {
    bar = 2_000;
  }
}

contract SimpleStorage is SimpleStorageLayout {
  // Whatever runtime logic is needed
}
```

With this architecture, the OPCM contract (which will be a non-proxied singleton per release) would
then either:

1. Deploy a new contract by:
   1. Deploying a new proxy
   1. setting its implementation to the `SimpleStoragePopulator`
   1. calling `initialize()`
   1. setting its implementation to `SimpleStorage`
1. Upgrade a contract by:
   1. setting its implementation to the `SimpleStoragePopulator`
   1. calling `upgrade()`
   1. setting its implementation to `SimpleStorage`

This workflow has the benefit of removing the `initializer()` functions, which:

- create a risk of reinit attacks
- bloat the runtime bytecode size
- are extremely verbose
- require finesse to avoid modifying or resetting values which are not meant to be changed.

## A single `SuperchainProxyAdmin`

The existing L1 `ProxyAdmin` (of which there is currently one per OP Chain), will be replaced with a
single L1 `SuperchainProxyAdmin`. The `SuperchainProxyAdmin`, in contrast with the L1 `ProxyAdmin`,
will have minimal storage, and will not internally track the type of Proxy which each contract is.

This contract will be owned by the [Upgrade
Controller](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/stage-1.md#L30), and
will be a simple pass through contract with a minimal interface.

```solidity
/// @notice forwards calldata to the specified address.
///   Reverts unless called by owner.
function forward(address, bytes) external;

/// @notice forwards calldata to the specified addresses.
///   Reverts unless called by owner.
function forward(address[], bytes[]) external;
```

This architecture makes it possible to manage upgrade authorization for the whole superchain through
a single storage value in a single contract (`SuperchainProxyAdmin.owner`), rather than having a separate contract per OP Chain.

It also avoids the need for an `sload` to read the `proxyType` mapping for each contract.

## Identifying OP Chains by the SystemConfig

We will use the `SystemConfig` as theunique identifier for each OPChain, and get the contract
addresses from it. The custom logic required for the two non-standard proxy types will be handled in
the OPCM contract, and does not require tracking the proxy type (as currently done in the
`ProxyAdmin`), since it is known already for each contract, ie. the only non-standard Proxies are
the `L1StandardBridge` (`L1ChugsplashProxy`) and `L1CrossDomainMessenger` (`ResolvedDelegateProxy`).

## Contract versioning

As part of this work, we will cease to version each L1 contract, and instead move to a global L1
contracts version. This version will be stored in the `SystemConfig` and OPCM. For backwards
compatibility, the existing `version()` getters will return the value from `SystemConfig.version()`.

The `DeployImplementations` script will write the version into the OPCM, which writes it into the
`SystemConfig`.

The `rc` tag will be included on the OPCM's initial version. It will be programmatically removed
when `upgrade()` is called by the Upgrade Controller multisig, which signifies governance approval.

## Proof system considerations

### Adding the PermissionlessDisputeGame to a chain

Given that not all chains will support the `PermissionlessDisputeGame` upon deployment, an
`OPCM.addGameType()` method will be added which will handle this.

WIP

1. ideally we update op-program to remove the chain config, so that it is identical across chains
   then it reads the config at run time.
   But we won't block isthmus on it, and will allow the `FaultDisputeGame` and `PermissionedDisputeGame` to be non-compliant.
2. inputs for each hardfork: one prestatehash per chain.

## Release process

As part of the release process, the associated implementation contracts and a new OPCM, will be deployed.

The Upgrade Controller (which MUST be a Safe), will `DELEGATECALL` the `OPCM.upgrade()` function,
providing the address of the `SuperchainProxyAdmin` and a list of the `SystemConfig` contracts for the
OP Chains to be upgraded.

Importantly,

Thus the high level logic for upgrading a contract should be roughly as follows:

```js

// Some upgrades will
struct NewChainConfig {
  address newValues;
}

Addresses immutable implementations;

constructor(Addresses _implementations) {
  implementations = _implementations;
}

function upgrade(SuperchainProxyAdmin _admin, ISystemConfig[] _systemConfigs, NewChainConfig[] _newConfigs) public {
  for(uint i=0; i< systemConfigs.length; i++) {
    // Read the `Addresses` struct from each `SystemConfig` in the `systemConfigs` mapping.
    // For each entry in the `Addresses` struct:
      // 1. Call the `SuperchainProxyAdmin` to update the implementation to the `Populator` contract
      //    TODO: OR update the implementation to
      // 2. call
      // 2. Update the implementation to the final implementation contract.
  }
  // Return ownership of the `SuperchainProxyAdmin` to the Upgrade Controller (`msg.sender`)
      //    `SuperchainProxyAdmin`. The unique upgrade call encoding required for the non-standard proxies
      //    should be handled here.
}
```

**Notes:**

- A new getter should be added to the `SystemConfig` to retrieve the `Addresses` in a single call.
- The `AddressManager` for each chain must be used to upgrade the `L1CrossDomainMessenger`, therefore it should be added to the `Addresses` struct.

## Development and Testing considerations

This solution should be implemented such that it both enables and _requires_ developers to implement
the upgrade path from the most recent release to the one curently being developed. This means it
will be necessary to have a deployment of the previous release available on the current commit, in
order to test the upgrade path.

This can be achieved by storing an L1 Genesis representation of the latest release in the monorepo.
This genesis file can be generated by from a state dump of the currently deployed test system. The
test suite should then be modified so that each test runs twice, one against a system freshly
deployed by the OPCM, and another which was upgraded from the previous release. In CI these two
major test groups can be run in parallel.

# Resource Usage

A rough per-chain estimate (accounting for lookups but not memory) based on the implementation described in [the upgrade process](#the-upgrade-process):

- Cold `SLOAD` the `SystemConfig` address: 2100
- Cold `CALL` the `SystemConfig` address: 2600
  - Cold `SLOAD` the `Addresses` struct (excluding `GasPayingToken`): 6 x 2100
- `CALL` the `SuperchainProxyAdmin`: 2600
  - Do the following 6 times:
    - `CALL` the Proxy: 2600
    - Cold `SSTORE` the temporary `Populator` implementation address over a non-zero slot: 5,000
    - Cold `SSTORE` new values: 20,000 each assuming empty slots.
    - Warm `CALL` the Proxy again: 100
    - Warm `SSTORE` the final implementation address over a non-zero slot: 100

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

1. Keeping the existing L1 Proxy Admin was rejected as being overly redundant and inefficient.
2.
3. Keeping initializers, and the StorageSetter was rejected as more over-prone, though possible simpler for testing purposes.
4.

# Risks & Uncertainties

TODO

---

6. We will track the correct addresses of the OPCMs in the registry. If you are upgrading by more than one version, you'll
   get the list from there.

function upgrade() {
require(systemConfig.releaseVersion() < )
}
