# [Interop Output Root to Super Root Migration]: Design Doc

|                    |                                    |
| ------------------ | ---------------------------------- |
| Author             | Adrian Sutton                      |
| Created at         | 2025-04-16                         |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_ |
| Need Approval From | _Reviewer Name_                    |
| Status             | In Review                          |

## Summary

Pre-interop the root claim of a dispute game is an `OutputRoot` which attests to the state of a single L2 chain and each chain has a separate `DisputeGameFactory`. Post-interop, there is a single `DisputeGameFactory` shared by all chains in the interop set and the root claim of games is a SuperRoot which commits to the state of all chains in the dependency set at a specific timestamp.

The process for migrating is: for each chain in the set we will create a dispute game, so that all chains in the set have a game at the same timestamp, and then wait for them to finalize. (For interop sets with only one chain we can just use any existing finalized game). These will then be converted into a super root.

## Problem Statement + Context

Pre-interop the root claim of a dispute game is an OutputRoot which attests to the state of a single L2 chain and each chain has a separate DisputeGameFactory. Post-interop, there is a single DisputeGameFactory shared by all chains in the interop set and the root claim of games is a SuperRoot which commits to the state of all chains in the dependency set at a specific timestamp.

Currently, the OPCM updates have been completed to migrate chains from pre-interop to post-interop by providing a trusted super root to use as the new anchor state. This can be used both for migrating individual chains to a 1-of-1 dependency set and for migrating the two chains that will be part of the initial interop cluster to a shared `DisputeGameFactory` with them both in the same dependency set.

The key input for that is [this `MigrateInput` struct](https://github.com/ethereum-optimism/optimism/blob/b8b1bc1e9b8c27982f7466bb8f0a7ff78c113f21/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L1235-L1241):

```solidity
struct MigrateInput {
    bool usePermissionlessGame;
    Proposal startingAnchorRoot;
    GameParameters gameParameters;
    OPContractsManager.OpChainConfig[] opChainConfigs;
}
```

The question is how to do this in a trustless way instead of manually specifying `startingAnchorRoot`.

## Proposed Solution

At a high level the work that is remaining is to modify that `MigrateInput` struct and remove the `startingAnchorRoot` field. Instead the starting anchor root should be created in a trustless way using existing on-chain data as much as possible. There will be other inputs required (e.g. the preimage of the current output root) but they should be verified against trusted on-chain data so it is not possible to migrate to an invalid anchor root.

### PoC

A proof of concept for converting `OutputRoot` to `SuperRoot` is available: [https://github.com/ajsutton/migratoor](https://github.com/ajsutton/migratoor). This shows how a `SuperRoot` can be trustlessly calculated from the `OutputRoot` from finalized dispute games from each chain in the target dependency set.

#### OutputRoot Structure

An `OutputRoot` is the keccak hash of an `BlockOutput` which consists of the following data all concatenated:

- `bytes32 version`
- `bytes32 stateRoot`
- `bytes32 messagePasserStorageRoot`
- `bytes32 latestBlockhash`

See [the spec](https://specs.optimism.io/fault-proof/stage-one/optimism-portal.html?highlight=outputroot#block-output) for full details. The only valid version is `0`.

#### Super Root Structure

A `SuperRoot` is the keccak hash of the following data all concatenated:

- `byte version` (always 1)
- `uint64 timestamp`
- `uint256 chainID` and `bytes32 outputRoot` for each chain sorted in ascending chain ID order.

See [the spec](https://specs.optimism.io/fault-proof/stage-one/optimism-portal.html?highlight=outputroot#super-output) for full details.

Importantly the output roots included must be from the last possible block at or before the super root timestamp.

#### Conversion Process

For each chain:

- Three inputs are needed:
    - Index in the chain’s current `DisputeGameFactory` of a finalized dispute game that resolved as valid
        - For chains going to a 1-of-1 dependency set, this can just be the current anchor state.
        - For op & uni we need to find dispute games that have the same timestamp - the current anchor states won’t fit that. We’ll need to propose our own games ahead of time to use and wait for them to finalize.
    - Preimage of the `OutputRoot` proposed in that game (ie the `BlockOutput` that hashes to the root claim).
    - The block header that’s the preimage of the `latestBlockhash` from that `BlockOutput` (this includes the timestamp)
- Verify the provided dispute game index exists and is a finalized game that resolved as valid and hasn’t been blacklisted etc.
- Get the chain ID from the dispute game (`FaultDisputeGame.l2ChainID()` - this isn’t part of the `IDisputeGame` interface but it’s present on all game type implementations we currently support)
- Get the output root from those dispute games (`IDisputeGame.rootClaim()`)
- Verify the provided `BlockOutput` hashes to that output root
- Verify the provided block header hashes to the `latestBlockhash` from the provided `BlockOutput`
- Get the timestamp from the provided block header

That gives us the `OutputRoot`, timestamp and chain ID for every chain. We can then build the super output:

- Verify the timestamp for all chains are the same
    - With different block times there are some valid cases where the timestamp doesn’t have to be the same, but verification is much simpler by requiring it to be the same and it has no real impact on the migration process.
- Verify the chains were provided in ascending chain ID order (or sort them but that’s more complex)
- Concatenate those values in the right order to build a super output
- Hash it to get the super root

The `Proposal startingAnchorRoot` input we’re removing can now be substituted by that super root and it’s timestamp as the L2 sequence number. The rest of the migration should follow the existing process.

### Selecting The Source Games

The governance proposal needs to be published, including the specific calldata the SC will sign some 4-6 weeks ahead of the upgrade actually being executed and during that time the current anchor state will keep advancing. Prior to MT cannon, we were concerned about the anchor state being too old, but with MT cannon we are confident that it is still safe even with up to a 6 month old anchor state.

Thus, to simplify the migration process we should select the dispute games that will be used to create the new super root anchor state as part of creating the calldata for the governance post.

The main downside of this is that when the migration is performed, the current anchor state will drop back in time by 4-6 weeks. It will then skip forward again in about 7 days when the first of the new games resolves and continue normally from there.
### Why Is This Better?

For SC controlled chains, since the dispute game to be used is included in the calldata in the gov post we’re actually running it through governance so could just use the new anchor root and skip this. However, it is more difficult for the security council to validate an arbitrary super root and more likely that we’d stuff up and propose a bad super root (wrong chain or whatever). Reducing the risk of things going wrong is particularly useful for chains not in the security council as they tend to have less stringent review. Plus if they ever do migrate to our security council a validated migration process makes it much easier to be confident that the chain history is valid and safe to take over.

### Preimages in Calldata

Including the preimage of the output root and block header in the parameters to `migrate` will make it a fairly large call. We have often tried to migrate multiple chains in the same transaction and the block gas limit can be a limiting factor for that. With the number of contracts deployed as part of the migration and the preimages in calldata we may not be able to batch as many chains in one call as usual. We should check how many chains can be upgraded in a single transaction.

### Off-Chain Coordination Required

We will need to run this migration on quite a few chains so need to make it simple to gather the required inputs.

For chains going to a 1-of-1 dependency set this would require:

- Select a recent, valid dispute game proposal to use as the dispute game index input
- Fetch the preimage of that game’s output root (`optimism_outputAtBlock` rpc from op-node)
- Fetch the block header for that block (`eth_blockByRoot` rpc from op-geth)

These can then be encoded into the calldata included in the governance proposal.

For op and unichain we would need to fetch a canonical, safe, output root from op and unichain at the same timestamp and create new dispute games using those output roots. We’d then use the index of those dispute games in the calldata and fetch the preimages for them as above. The games will have finalized by the time the proposal has completed governance.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

### Single Point of Failure and Multi Client Considerations

This doesn't require changes in multiple clients. It's a one-time action for each interop set.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

## Impact on Developer Experience

<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->