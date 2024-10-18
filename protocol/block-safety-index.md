# Block safety index

# Summary

Unify the mapping between L2 blocks and the L1 blocks they were derived from,
to address cross-protocol safety-validation challenges.

This mapping conceptually exists in different forms in the OP-Stack,
but should be described as a feature of the rollup-node, for other components to be built against.

This does not directly affect the op-program (or Kona), but does affect how we source the data to enter a dispute game.

# Problem

The following things are all *the same*:
- The op-node warmup UX problem
- The span-batch recovery problem
- The fault-proof pre-state problem
- The interop reorg problem
- The L2 finalization problem
- The op-proposer consistency problem
- The safety indexing problem

At the core, all these issues stem from a single problem:
determining which L2 block corresponds to which L1 block.

Yet we currently have *different solutions* for each case.

The reason why this is hard: the L2 chain, by design, does not pre-commit to how/where data gets confirmed on L1.
This reduces reorg risk and DA cost by deferring optionality to the batcher, but introduces "offchain" state.

Can we fix this, and simplify the OP-Stack?

## Sub-problems

### The op-node warmup UX problem

The majority of op-node warmup is derivation-pipeline reset time.
It traverses to an old L2 block guaranteed to still be canonical,
then traverses to an old L1 block that is guaranteed to produce
the inputs needed to continue derivation of next L2 block inputs.

This traversal can be lengthy:
- The L2 chain is traversed back, because we cannot with certainty say whether it was derived from a recent L1 block.
  Often this means it "bottoms out" on the finalized L2 block, at least ~12 minutes old.
- The L1 chain is traversed back a full channel-timeout; 300 blocks, i.e. an hour of L1 data!
  (reduced in Granite to 50 blocks = 10 minutes).

This delay slows down the process of making nodes fully operational after a restart,
particularly with regard to derivation.

But it can be so much faster, if we introduce state:
- L2 traversal can go back to the last L2 block derived from a still-canonical L1 block.
  Seconds, not minutes, of traversed time-span.
- L1 traversal can go quickly find the relevant L1 data with a much shorter search space,
  if we had stricter batch-ordering guarantees (being introduced with Holocene!) and the derived-from mapping.
    - The last derived-from L1 block is an upper bound.
    - The previous-to-last derived-from L1 block is a lower bound.

### The span-batch recovery problem

Currently, span-batch recovery (pre-Holocene) relies on 'pending-safe',
where a span-batch may generate pending blocks that can be reorged out
if later content of the span-batch is found invalid.

This is changing with Holocene (steady batch derivation, aka strict ordering): we do not want the complexity of having to revert data that was tentatively accepted.
For interop fault-proving, the L2 assumptions should not be larger than 1 block,
since we want to consolidate cross-L2 after confirming each L2 optimistically.

If we find a pre-existing L2 chain (simply on node restart, or on reorg),
we currently cannot be sure it was derived from a certain L1; we have to recreate the derivation state to verify.
And if the tip was part of a span-batch, we need to find the start of said span-batch.
So while we can do away with the multi-block pending-safe reorg, we still have to "find" the start of a span-batch.
If we had an index of L2-block to L1-derived-from, with span-batch bounds info,
then finding that start would be much faster.

### The fault-proof pre-state problem

The current fault-proof prestate-agreement depends on having an optional "safedb" feature enabled in the op-node.
This essentially serves as an index, mapping which L2 block has been derived from which L1 block.

That index is good, close to what this doc proposes, but the implementation is fragile:
- It's optional, no clear way of backfilling the data, when enabled on an existing node.
- It wraps a key-value DB with arbitrary key / update scheme: more easy to corrupt, noisy (level/table changes) to update.
- It depends on the temporary synchronous derivation state changes for consistency,
  it is not yet attached to clear derived-from events.

Deprecating/removing the "safedb" and its updating, in favor of a new unified stable derived-from indexing solution,
would thus be a good improvement.

### The interop reorg problem

Currently, interop cross-L2-reorgs are not supported yet, but we will need to for following devnets / production.
To do so, as verifier node we need to answer: "Is my L2 A view invalidating your L2 B view, or vice-versa?"

The key to answer that is ordering of L2 safety increments based on L1:
we can then say if A or B was first, and which chain has to reorg to fix its dependencies in favor of the other chain.

Unless there is some other novel way around the above, the op-supervisor needs to tell apart that order.
To support that, we again need some form of a DB index of when a L2 block became safely derived from a L1 block.

If the op-node maintains this index already, then it can upstream its single-chain view of this to the op-supervisor,
which can maintain the multi-L2 view of L1 safety.

The index should contain two things to support interop:
- per L1 block, tell up to which L2 block was considered "local-safe"
- per L1 block, tell up to which L2 block was considered "cross-safe"

The "local-safe" entries are not monotonic; if external dependencies change, we can see the label rewind.
But the key is that the label rewinding per L2 has a global ordering,
helping us resolve what do about a L2 chain that may or may not be local-safe.

The "cross-safe" entries are monotonic; these can only change on L1 reorg,
at which point the whole derived-from index would reorg.

### The L2 finalization problem

Current L2 finalization caches recent L2-to-L1 derivation relations.
The L1 beacon chain updates, if it finalizes at all, with a delay of ~2 beacon epochs from the tip of the L1 chain.

By keeping the last 2 epochs of derived-from mappings in memory,
the op-node can tell up to which L2 block was derived from a L1 block. And as soon as the L1 finalizes,
trigger the finality of the L2 blocks that are now irreversibly derived from finalized L1 inputs.

All of this caching logic would become unnecessary,
if we had a reliable lookup method of this L2-derived-from-L1 mapping data, to act on for any L1 finalization signal.

On top of that: the finality deriver sub-system of the op-node can potentially be completely removed,
if finality is checked through the op-supervisor API.
The op-supervisor already knows which L2 block is safe,
and what L1 block it was derived from, and can thus answer when a L2 block is finalized, given a finalized L1 view.

### The op-proposer consistency problem

The op-proposer has two modes:
- Wait for finalized L2 block, which we know we can propose, regardless of L1, as it's already final.
- Wait for a safe L2 block, and make op-node stop-the-world to get consistent derived-from and safe-L2 data.

For safety/stability the finalized-L2 mode is generally used in production.
But the latter would be faster, and is preferred in testing.
It can work better if we wouldn't rely on the stop-the-world state, but instead had an index to query,
to determine which block proposal has to be bound to which L1 blockhash, for reorg safety of L2 proposals.

### The safety indexing problem

Safety indexing is a feature of:
- explorers: show that a L2 tx became safe at a specific point in L1, possibly even linking the batch data.
- bridges: determine what L2 blocks to accept for bridging, given some coupling to L1 batch confirmation.

An index, of which L2 block was derived from which L1 block, would help standardize this data,
and enable every bridge/explorer to interact safely with cross-L2 block data.

# Proposed Solution

Interop, or technically "Isthmus" really, is an upgrade meant to apply to all OP-Stack chains.
Chains with a dependency-set of 1 (themselves) should not have to run the op-supervisor,
but should maintain a reliable view of the events and safety of their local chain.

Like the op-supervisor events DB, the history is append-only, except for reorgs.
This means we could optimize, and store things in an append-only log style system.

The "safedb" in op-node can be replaced with a DB like the above, shared code with the op-supervisor.
Once the above is done, we can remove the finality mechanism: the op-supervisor signals are used for finalization.

The supervisor is Go, and modular enough that we can embed the single-DB solution into the op-node,
to serve those less-standard chains with an interop-set of 1.
This prevents these chains from entering a divergent hardfork path:
very important to keep the code-base unified and tech-debt low.

The main remaining question is how we bootstrap the data:
- Chain-sync is on-chain information only, since it needs to verify retrieved data against authenticated commitments.
  We cannot start from snap-sync if we still have to backfill the history data.
- Nodes don't currently have this state (if they have not been running the safe-db),
  we need to roll it out gradually.

This feature could either be a "soft-rollout" type of thing,
or a Holocene feature (if it turns out to be tightly integrated into the steady batch dervation work).

## Resource Usage

The op-node "safeDB" is closest to what we have today,
to an index that maps between L2 blocks and L1 blocks that were derived from.

This index is missing space for interop-compatibility (storing both local-safe and cross-safe increments),
but gives an idea of the resource usage.

In terms of compute it's very minimal to maintain:
the op-node verifies L1 anyway, and writing the incremental updates is not expensive.

In terms of disk-space it is a little more significant:
at 32 bytes per L1 and L2 block hash, some L1 metadata, some L2 metadata, and both local-safe cross-safe,
each entry may about 200 bytes. It does not compress well, as it largely contains blockhash data.

Storing a week of this data, at 2 second blocks, would thus be `7 * 24 * 60 * 60 / 2 * 200 = 60,480,000`, or about 60 MB.
Storing a year of this data would be around ~3 GB.

The main resource-risk might be the reconstruction work: this requires resyncing the L2 chain.
Easy snapshots, robust handling to avoid data-corruption, and simple distribution would avoid the need to re-sync.
Potentially this can be merged with the work around op-supervisor events DB snapshot handling.

# Alternatives Considered

## Reusing the safedb

The safeDB in the op-node could be re-used, but there are several things to fix, with breaking changes:
- the way it is updated needs to be less fragile.
- the way it is searched for a L2/L1 mapping, likely needs to improve.
- the format needs to change to support interop cross-safe data.

Since the scope of the index itself is relatively small,
I believe redoing it, while designing for all the known problems, not just fault-proofs, produces a better product.

# Risks & Uncertainties

## Scope

While the number of problems is large, we do also already have (suboptimal) solutions in place for each of them.
We could make a draft implementation,
and merge one problem at a time, in order of priority, to remove the legacy incrementally.

In particular, this work might be a key part of Interop devnet 2, to handle reorgs. This should be prioritized.
The potential benefit to Holocene steady-batch-derivation should also be reviewed early,
so we can reduce Holocene work, by de-duplicating it with Interop.

