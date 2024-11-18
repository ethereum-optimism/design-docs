# Purpose

Design a solution that makes it easier to add or update chain configurations for use in fault proof programs and
separate the release of op-program from updates to chain configuration.

# Problem Statement + Context

Currently, op-program and kona embed the configurations for chains they support via the superchain-registry and these
are committed to on-chain as part of the `FaultDisputeGame` absolute pre-state. For existing chains that activate hard
forks as part of the superchain configuration, this works well and the new chain configuration is included in the same
update to the absolute prestate that implements the new hard fork rules in the fault proof program.

However, chains that activate hard forks after the initial superchain set or new chains deployed between hard forks,
they are unable to use the same absolute prestate because it doesn't contain the latest chain configuration. A new
op-program release is required but it then isn't governance approved and may have other dependencies (e.g a recent
update to the version of go used means an update to MIPS.sol is required).

The conflation of implementation and chain configuration also means that we need a different absolute prestate for
devnet, testnet and mainnet - each typically only differing by the addition of the hard fork activation time.

# Proposed Solution

Updating chain configuration could be simplified if the implementation and chain configuration were separated rather
than both being committed to as part of the same absolute pre-state hash. `op-program` and `kona` would no longer
include the superchain-registry but instead always load the chain configuration by requesting it via a local key from
the `PreimageOracle`.

A new `ChainConfigRegistry` contract would be created which provides a mapping from L2 chain ID to a hash of the chain
configuration. A permissioned method would be provided to update the chain configuration, restricted to the
`ProxyAdminOwner`. The `ProxyAdminOwner` role is used here because changing the chain configuration affects the
definition of the valid chain, similar to upgrading contracts.

The chain configuration files are JSON and may be large, so are committed to via a hash rather than uploaded directly.

When a `FaultDisputeGame` is created, it would read and store the current chain configuration hash from the
`ChainConfigRegistry` to ensure that the configuration is consistent for the entire life of the game.

# Alternatives Considered

## Store the Entire Chain Configuration

The entire chain config could be stored in the registry rather than a commitment hash. However, the gas cost of storing
the entire config for all chains would be quite significant.

## Store Partial Chain Configuration

It would be possible to store only part of the chain configuration on chain, such as timings of hard forks along with
the hash of the genesis block. This would be more inline with the information stored in the superchain registry.

This could be a better option if there was an efficient serialization that allowed us to add new fields to the 
configuration (e.g a new hard fork activation time) without needing to update the `ChainConfigRegistry` contract itself.
Ideally, such a format would still make it easy to access the information on chain.

The main benefit of this option is that it provides the required information in a standard format, avoiding the config
file format differences between op-program and kona and also avoids problems formatting differences affecting the hash.

It's unclear how the dispute games would "lock in" the chain configuration used at the start of the game with this 
approach as copying the configuration would be too expensive.

# Risks & Uncertainties

## Configuration File Format

Currently, `kona` and `op-program` use slightly different configuration formats and the genesis file format used by the
execution engines is controlled by upstream and may diverge further in future. We will need to define a common format
that can be converted into the appropriate configuration for the libraries as needed. This is already done in the
superchain-registry and just needs to be adapted to be usable in this context.

## Large Configuration Files

While the `ChainConfigRegistry` commits to the chain configuration via a hash, during a dispute game the honest actor
may be forced to provide the full preimage into the `PreimageOracle` if a step that reads from it is disputed. The
chain configuration would then need to be small enough to fit within a single block.

If we use `keccak` to create the hash commitment, we could use the large preimage solution to handle chains with larger
configuration files, but in general we want to avoid adding more dependencies on that solution.

We should analyse the existing chain configurations and determine how close they are to being too big to submit in a
single block.

## Differing Formats

The chain configuration is a text based format - typically either JSON or TOML, and so there may be many equivalent
files that have different formatting. To populate the `PreimageOracle` the formatting must precisely match that used to
create the original commitment. We will need a way to normalize the formatting and ensure that only the normalized form
is used when creating the commitment hash.

Note that this is a security critical concern as dispute games could be lost if the preimage for the chain configuration
commitment cannot be provided.

## Increased Proposal Gas Costs

As the chain configuration commitment would need to be copied to the `FaultDisputeGame` upon creation, this will incur
additional gas costs for each proposal (roughly the cost of `CALL`, `SLOAD` + `SSTORE` plus a bit of overhead - ~27000).
