# Flashblock Access Lists (FBALs)

| Author             | @protolambda |
|--------------------|--------------|
| Created at         | *2025-09-12* |
| Initial Reviewers  | *TBD*        |
| Need Approval From | *TBD*        |
| Status             | *In Review*  |

# Summary

Some OP-Stack chains are growing fast and require Ethereum execution scaling.
The OP-Stack can implement Block Access Lists ahead of L1, without diverging,
by leaning in to flashblocks and improving the block distribution.

# Problem

Fees can be adapted to improve incentives for the used rollup resources,
but actual execution (incl. storage interaction) is not addressed yet.

Flashblocks focus on latency more than performance:
they can still be faster, but block-building and block-processing cannot yet keep up.

This design-doc focuses on the block-processing part:
block-building needs optimization too, but that problem is isolated to well-resourced sequencing nodes, and not as constrained by standardization.

### Pre-reading

Related, recommended pre-reading:
- Block Access Lists [EIP-7928].
- [Flashblock specs](https://github.com/flashbots/rollup-boost/blob/main/specs/flashblocks.md).
- [Flashblocks implementation docs](https://github.com/flashbots/op-rbuilder/blob/main/docs/flashblocks.md).
- [Block Access Lists Eth-Magicians thread](https://ethereum-magicians.org/t/eip-7928-block-level-access-lists/23337).

[EIP-7928]: https://eips.ethereum.org/EIPS/eip-7928


### Known issues

- State-root computation takes a long time, and creates a synchronous hiccup every 2 seconds (block interval)
- Flashblocks are 200ms and difficult to reduce further, because compute can take longer
- Compute time is highly variable primarily due to storage-access/mutation and poorly priced precompiles
    - Precompile prices are fairly accurate, but can be adjusted more.
    - State growth cannot be adjusted without hurting normal usage.
- Flashblock distribution has room for improvement.
- The State DB is getting overweight, compaction is unpredictable and slow,
  the geth DB are not keeping up with reth or the new rust DB work by Base.
- Flashblocks don’t make the actual raw verification processing faster, op-reth still operates largely the same.
  Flashblocks as-is are about reducing tx-confirmation latency.
- Block-access-lists from L1 are not yet compatible with flashblocks.

# Towards a Solution

Several ideas come together here:

- [EIP-7928]: Block-Level Access Lists (a.k.a. BALs).
    - Pre-declared keys of read-operations allow for parallel reads
    - Transaction-indexed storage mutations allow for parallel transaction processing
- Flashblocks: Distribute incremental additions to the block
    - We should pre-process flashblocks, for blocks to not take as long
- Out of protocol updates: Flashblocks and BALs both don’t need to modify the core state transition. 
    - If we don’t modify the block-header like in EIP-7928, then we can roll out this change without breaking the chain rules.
      - This is *Great for compatibility*: the Block-Access-List design can continue to improve, while the improvements can be adopted by the L2 without major hardfork.
      - This is *Great for rollout*: No need to wait for a protocol hardfork.

#### Related: Solana

If it still needs to go faster, we could also mix in more foreign block-propagation improvements, and adapt them to Ethereum:

- Solana Turbine: Fan-out distribution system for shreds. The “turbine tree” distributes data quickly to all validators in a hierarchical way. This is stake-weighted: with higher stake, you get shreds sooner.
  - This feature seems to be changing in Solana, and there is no clear path for stake-weighted propagation in L2, but this can be done.
- Solana Shreds: Reed-Solomon erasure code the diffs, prevent compounding of network-packet-loss. This enables one-way publishing, instead of the slower gossip (heartbeat of 500ms to correct missing data) and TCP (two-way comms).
  When publishing over UDP, or nodes in the tree malfunction, the erasure coded shreds help recover missing data.
  - At its current stage this is deep scope-creep. There is interesting overlap here with the UDP-based Data-availability-sampling work of ethereum in the past, but optimizing distribution more may be premature before improving the execution itself.

#### Related: Payload Chunking

While iterating on this design doc, the L1 Block Access List work continues also,
and "Payload Chunking" was proposed on [Eth Research](https://ethresear.ch/t/payload-chunking/23008).

This works *a lot* like Flashblocks work: it splits blocks in streaming slices, and distributes the blocks offchain.

It highlights processing changes:
- streaming: like flashblocks, apply the block data as it comes in.
- after a (regular) block exists fully: process chunks in parallel, including the downloading of chunks.

Ultimately the parallel processing is a feature of Block-Access-Lists already:
with the intermediate state-diffs and access information, the transactions can be processed in parallel.

Flashblocks differ in the sense that they avoid computing the state-root for every single flashblock.
In practice state-root computation is still done in some implementations of flashblocks, because of easier pre-conf failover handling.
The Payload-Chunking proposal suggests a `ChunkHeader` with a `post_state_root`, which might only add unnecessary overhead for the builder.
The flashblock design includes a "flash hash", committing to the pre-state and flashblock, rather than the post-state, to uniquely identify the state changes.


# Proposed Solution: FBALs

Block-level Access Lists (BALs) in L1 are coming, but not quite flashblock-compatible,
and not until the Glamsterdam upgrade (possibly a year from now).

This proposal is to introduce Flashblock-level Access Lists (FBALs):
we extend the BAL design to flashblocks.

The L1 client already includes common functionality,
such as the features to “warm up” the necessary storage, collect the access-data, etc.
This can be shared between the L1 BAL work and the L2 FBAL work.

See:

- geth:
    - [PR 31948](https://github.com/ethereum/go-ethereum/pull/31948): BAL types (merged PR)
    - [PR 31959](https://github.com/ethereum/go-ethereum/pull/31959): BAL construction (closed PR)
    - [PR 32134](https://github.com/ethereum/go-ethereum/pull/32134): storage pre-loads (merged PR)
    - [PR 32263](https://github.com/ethereum/go-ethereum/pull/32263): main BAL implementation (open PR)
- reth:
    - [Tracked in 18253](https://github.com/paradigmxyz/reth/issues/18253): work was started in non-PR branches

Flashblocks are already received in verifiers,
[reth 1.7.0](https://github.com/paradigmxyz/reth/releases/tag/v1.7.0) includes native OP-Stack flashblock support! 

Flashblocks can then:
- Be processed from the incoming stream (through rollup-boost, or the new flashblock EL P2P distribution).
- Be merged into the pending block
- Be processed each with transactions in parallel

And since flashblocks are offchain (not reproduced from batch-data),
we heuristically ideally have a verifier node hold off on regular block processing
until the above has been attempted,
since flashblocks will be faster (parallel txs) than regular blocks.

### Flash-BAL

A “flash BAL” is a diff increment of a block-level access-list.

We can distribute it as sidecar to the flashblock,
so tooling can always handle flashblocks, and opt-in to the latest version of the FBAL.

This alone can accelerate the normal block-processing time a lot:
parallel storage reads, parallel tx processing, eager flash block processing.

```python
# Same as in EIP-7928:
from eip7928 import 
  TxIndex,
  StorageChange,
  BalanceChange,
  NonceChange,
  CodeChange,
  SlotChanges,
  AccountChanges

# New: like BlockAccessList, but for a slice of transactions
class FlashBlockAccessList(Container):
  flash_hash: Bytes32  # link to the flash-block
  min_tx_index: TxIndex # incl.
  max_tx_index: TxIndex # excl.
  # Note: unique accounts per flash-BAL,
  # but flash-blocks may add changes to accounts shown in previous diffs,
  # and thus not necessarily unique full block scope.
  # This includes the "Address" type of pre-load hint, which isn't strictly a "Change".
  account_changes: List[AccountChanges, MAX_ACCOUNTS]
```

This does not get linked into the onchain header: the flashblocks are out-of-protocol, merely a happy-path optimization.
Once there is stability in the design, common with L1, it can make sense to enshrine the optimization, so that BALs can be synced for historical data easily, and verified against a block-header commitment.
This might look like enshrining the FBAL, or merging FBALs into regular BALs, or following something like the Payload Chunking proposal.
