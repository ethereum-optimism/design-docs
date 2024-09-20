# Purpose

To identify the challenges with starting a new chain using fault proofs and select an option to make the process simple
and reliable.

# Summary

New chains are recommended to use the `contracts/v1.6.0` release, which includes fault proofs. While this works better
with deployment tools, cyclic dependencies between the L1 contract deployments and the L2 genesis generation make it
impossible to create a fully valid deployment context with the `faultGameAbsolutePrestate` and
`faultGameGenesisOutputRoot` values being unknowable.

# Problem Statement + Context

The current chain deployment process involves deploying all the L1 contracts, then generating the L2 genesis state.
This is required because the L2 genesis state references a number of L1 contracts, including the `SystemConfig` and
`OptimismPortal`. The fault proof system creates two cyclic dependencies:

1. The `faultGameGenesisOutputRoot` is used by the L1 `AnchorStateRegistry` contract as the initial anchor state for
   permissioned and permissionless games. For new chains it must be set to the L2 genesis output root, which is
   unknowable prior to deploing the L1 contracts.
2. The `faultGameAbsolutePrestate` is used by the L1 `FaultDisputeGame` contract to define the version of `op-program`
   to be used as part of permissioned and permissionless dispute games. `op-program` requires the L2 chain
   configuration (rollup config and L2 genesis state) to be available in the version of the superchain-registry that it
   includes to operate correctly. Similar to above, the L2 chain configuration is unknowable prior to deploying the L1
   contracts. Additionally, chains cannot be added to the superchain-registry until _after_ they have launched.

# Proposed Solution

TBD

# Alternatives Considered

## Defer Deployment of Dispute Game Types

Remove the `FaultDisputeGame` and its dependent contracts (including `AnchorStateRegistry`) from the initial L1
deployment. The `DisputeGameFactory` would still be deployment, but with no supported game types. It would be impossible
to create any dispute games and thus withdrawals would be disabled. The `faultGameGenesisOutputRoot` and
`faultGameAbsolutePrestate` values would not be required ahead of this initial L1 deployment as they are only used by
the dispute game implementations.

The L2 genesis state could then be created based on the deployed L1 contracts. Optionally the chain could be started at
this point without support for withdrawals.

The config would need to be added to a version of `op-program` either by adding the chain to the superchain-registry,
updating the superchain-registry dependency in the optimism monorepo and tagging a new release of `op-program` or by
creating some kind of custom build of `op-program`.

Finally, the `ProxyAdminOwner` would deploy the `FaultDisputeGame` and dependent contracts, providing the correct
`faultGameGenesisOutputRoot` and `faultGameAbsolutePrestate` values and call `setImplementation` on the
`DisputeGameFactory`. Withdrawals would then be possible.

### Pros

* Neatly avoids the cyclic dependency.
* Chains that are not supporting permissionless games yet could deploy only the permissioned game, preventing creating
  permissionless games entirely.

### Cons

* The `ProxyAdminOwner` is required to perform the `setImplementation` calls because ownership of the L1 contracts has
  already been transferred. In the current system, the L1 contract deployment can be performed by anyone and the system
  is fully configured (albeit incorrectly for proofs) before ownership is transferred.
* It remains unclear at what point a chain can be added to the superchain-registry or how a version of `op-program` that
  supports the new chain would be created.

## Precalculate L1 Addresses

Since the L1 deployment addresses are deterministic for a given seed, it may be possible to pre-generate them, then
generate the L2 genesis block and chain configs prior to deploying the L1 contracts. This would allow the contracts
to be deployed with the correct fault proof genesis block. If the config could be added to the superchain registry
prior to deploying the L1 contracts, it would also allow a suitable fault proof absolute prestate hash to be specified.

One challenge here is that the `SystemConfig` contract stores the start block automatically based on the block it is
deployed in. This value affects the L2 genesis. While it does have some support for custom values, that's intended for
legacy chains. It is as yet unknown if setting a start block number from before the `SystemConfig` contract is actually
deployed causes any problems. Notably, the L1 start block timestamp is used as the genesis time for the L2 chain.
Delays in deploying the L1 contracts and actually starting the chain could then mean a large number of empty blocks need
to be generated before proper sequencing can begin for the chain. The L1 starting block hash is included in the rollup
config so a future block number can't be used.

### Pros

* L1 contract deployment remains atomic

### Cons

* If there is any issue with the L1 contract deployment, the L2 genesis and config in the superchain registry may need
  to be adjusted. Given the need for review, this could be a slow feedback loop.
* After the chain is added to the superchain-registry, the registry needs to be updated in the monorepo and likely a new
  release of `op-program` tagged that includes the new chain config.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->