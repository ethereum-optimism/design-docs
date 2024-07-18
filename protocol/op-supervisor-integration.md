# `op-supervisor` integration

# Purpose

The previously described [`op-supervisor` backend design](./superchain-backend.md) 
serves the resolution of cross-L2 dependencies, i.e. the "cross" part of safety,
but not the resolution of superchain-wide L1-inclusion and case of reorg, i.e. the "safe" part.

This document describes how to complete the supervisor-functionality,
such that it can be integrated with the op-node and op-geth functionality.

# Summary

We extend the op-supervisor with an append-only database, similar to that of the event-database,
to track the interdependent safety-level increments across the superchain.
This data can then be exchanged with the op-node to promote the safety of its respective L2 blocks.

# Problem Statement + Context

# op-supervisor super-safe-db

The main challenge with the op-supervisor, and op-node, are reorgs.

With interop, due to cross-chain dependencies, reorgs may be triggered on a previously locally derived series of L2 blocks (single chain).
This may happen due to a missing event-log on another chain, which may have been independently reorged (either due to L1 change, or due to another L2 interop reorg).

Flip-flops in reorgs, non-deterministic ordering of op-node interactions with the supervisor, and replay of events / L2 blocks from different batches on L1, all make single-chain-perspective solutions infeasible. The safety changes of L2 chains need to be tracked atomically across the superchain, to consistently apply resolution of invalid dependencies.

Taking a global perspective on safety-increments, each increment anchored to the view of the L1 chain, is identical to the way the interop Fault-Proof resolution is proposed to work.

Offchain however, we need to book-keep the different increments, to replicate the view of each one of them, in case of a reorg back to this. Likewise, an FP game is initiated with this same data, artificially constraining the view of the L1 chain and the L2 chain derived from it thus far.

The “safe db” functionality of the op-node maintains this exact data, except only for a single chain, in an unoptimized manner, as optional add-on.

With interop, we propose to introduce a “super safe db”, maintained by the op-supervisor, to book-keep this information as part of interop verification. With this, the supervisor will be able to handle reorgs and provide accurate cross-L2 safety views to op-nodes.

# Alternatives Considered

*List out a short summary of each possible solution that was considered.
Comparing the effort of each solution*

## Only using op-supervisor event-logs and in-memory heads data

We started without the `super-safe-db`, with an attempt to resolve safety and reorgs against the event-logs data.

However, the event-logs data does not capture:
- Where L2 blocks were derived from on L1: determining which L2 block needs to be reorged out upon invalid executing messages
    requires an understanding of L1-ordering.
- When we choose to go with a deposits-only block, it may invalidate other L2 data at the same timestamp.
- Relations between L2 cross-safe increments, to set up a FP pre-state view.

# Proposed Solution

The solution consists of 3 parts:
- super-safe-db: data in supervisor to resolve safety.
- op-supervisor RPC: serve queries to op-node / op-geth.
- integration: op-node / op-geth access op-supervisor RPC to maintain a view of safety.

## Super-safe-db

### Primary key

Unlike the previous local "safe-db", we use the superchain timestamp, not the L1, as primary key.

With L1 block number as primary key:

- Key-space may leave gaps in L2 blockhashes
- Not superchain timestamp aligned
- Binary search to find what L1 blocks we need to determine a specific L2 timestamp
- Binary search to determine L2 safety
- Easy to roll back on L1 reorg

With superchain timestamp as primary key:

- All L2 blockhashes can be found.
- We store more duplicate derived-from information; presumably 1 block-number per L2 block, and the L1 blockhash of at least the highest L1 block number.
- Binary-search to find what L2 data was derived from up to a specific L1 block.
- More difficult to roll back on L1 reorg.
- Easy to capture interop timestamp invariant, thus easy to maintain horizon properties.
- Weakest-link safety problem: cross-safe becomes global, increments all at once for everyone.
  (This forms a delayed safety risk when 1 chain is not batch-submitting).
- Consistent, roll-back on invalidated executing messages always straight-forward.
- While the same as the FP bisection step, the L1-view is now inconsistent,
  or completely delayed/unavailable, between different L2 chains.

With each L1-block or timestamp increment, we need to capture the addition
of any L2 block that is failing executing-message checks. 

Note that additions are dynamic-size (based on chain interop set), and so every lookup will always be some kind of search.

### DB updates by op-supervisor

This DB can then be used to query when each L2 block became cross-safe.

This same DB can be used by the challenger, to replace the existing safe-db. A must when starting interop-FP work.

And this same DB can be used to improve the `FindSyncStart` performance, since the super-safe-db will have the exact data to determine where to start syncing from, when given a L1 block and chainID.

#### finalized (always cross)

To query the latest finalized L2 block,
simply binary search for the last L2 block fully derived from the finalized L1 block.
This can simplify existing op-node functionality, and make finality more reliable.

#### cross-safe

To query whether an L2 block is `cross-safe`, binary search for the L2 block, to determine when (w.r.t. L1 block)
it became cross-safe. If this L1 block exists and is canonical, then the L2 block is cross-safe.

#### local-safe

The op-nodes determine this for themselves, and exchange it with the op-supervisor. The op-supervisor includes the "local safe" increment in the entry, but it may not be promoted to "cross safe" until all dependencies are met.

#### cross-unsafe

Optimistically the supervisor may fetch the event-logs of unsafe blocks. Blocks with transitively safe dependencies may be marked as cross-unsafe.

The cross-unsafe data is not persisted across restarts: this can be re-computed from the event-log DB quickly,
and recomputing makes it much less complicated to stay consistent with the database.

#### unsafe

The sync-status exchange triggers the op-supervisor to fetch the event-logs of new unsafe blocks.

If the new unsafe-block does not build on the previous unsafe block (parent-hash check), then the event-log DB needs to be rewound to the last blockhash checkpoint still consistent with the post-reorg L2 chain.
This may not revert deeper than cross-safe, as the L1 chain is the source of truth for reversals that deep.
The cross-safe will has to rewind first, and then when unsafe-blocks still “don’t fit” on the remaining unsafe data, the unsafe data reorg is triggered again, and can be rewound further.

### Super-safe-DB format

The DB is append-only, with truncation on rollback:
this makes it easy to recover on corruption, and fast and efficient.

The entries are fixed-size:

```go
<version> // 1 byte identifier of entry
<derived-up-and-including-to L1 block number> // 8 bytes, primary key
<derived-up-and-including-to L1 truncated blockhash> // 20 bytes

<chain idx> // 4 bytes, same indexing as executing event-log entry

<l2 local-safe number> // 8 bytes. local-safe will be promoted to cross-safe once all dependencies transitively verify
<l2 local-safe truncated blockhash> // 20 bytes. May refer to an invalid non-canonical L2 block, as dependencies are unverified.

<l2 cross-safe number> // 8 bytes. Each L2 has its own. Also implied to be finalized, if derived-from L1 is finalized.
<l2 cross-safe truncated blockhash> // 20 bytes.

<flags> // 1 byte, to indicate if cross-safe was a deposit-only block
// total: 1+8+20+4+8+20+8+20+1 = 90 bytes
```

For each chain in the dependency set, for each L1 block,
we append one of these entries to the super-safe-db.
If an L2 chain has not traversed gotten to the next L1 block yet, the DB writing work halts until it does.

In a future iteration we may add other `<version>` data,
and fit it into `90` bytes (using multiple entries if necessary) to register data.

TODO: We modify the `event` DB to checkpoint every L2 block,
rather than only forcefully placing such entry every 256 entries.
This will help verify any gaps between L2 blocks,
if the `cross-safe` or `local-safe` jumps by more than 1 block at a time.

## op-supervisor RPC

### `CheckMessage`

```go
type SafetyLevel string

const (
    Finalized   SafetyLevel = "finalized"
    Safe        SafetyLevel = "safe"
    CrossUnsafe SafetyLevel = "cross-unsafe"
    Unsafe      SafetyLevel = "unsafe"
	Unknown     SafetyLevel = "unknown"
    Conflicts   SafetyLevel = "conflicts"
)

type RPC interface {
	CheckMessage(identifier types.Identifier,
                 payloadHash common.Hash,
                 ) (types.SafetyLevel, error)
}
```

This method is used to verify initiating messages exist, e.g. used by the tx-pool and block-building, to verify user-input before inclusion into the chain.

API implementation:

1. Find the event log-db entry at the given identifier
2. Reconstruct a `TruncatedHash` from the `identifier` and `payloadHash`
3. Check if the log-db entry matches the reconstructed `TruncatedHash`
    - If it matches, return the safety-level of the entry
        - The safety is determined by the first (and thus strongest safety level) type of head of the chain that is past or equal the entry.
    - If it does not match, return the `conflicts` safety level.
    - If the entry is not yet known, return an `unknown` safety level.


### `TryCrossUnsafe`

```go
type RPC interface {
    GetCrossUnsafe(chainID types.ChainID, maxNumber uint64) (crossUnsafe eth.BlockID, err error)
}
```

API implementation:
1. Compare the `maxNumber` to the locally known `unsafe` tip of the event-DB:
    - If newer:
      1. Initiate fetching of new event-log data.
      2. Return the latest known `cross-unsafe`.
    - If equal or older:
      1. Return the latest known `cross-unsafe` block before `maxNumber`.
         This is the last block checkpoint `<= maxNumber`

By clipping using `maxNumber` the op-node is able to verify the returned `cross-unsafe`, and not accept it if it's non-canonical, and optionally query for older cross-unsafe blocks to try to bump to.

### `TryCrossSafe`

```go
type CrossSafeStatus string

const (
	CrossSafeAccept CrossSafeStatus = "accept"
	CrossSafeUnknown CrossSafeStatus = "unknown"
	CrossSafeInvalid CrossSafeStatus = "invalid"
)

type RPC interface {
    TryCrossSafe(chainID types.ChainID, localSafe eth.BlockID, derivedFrom eth.BlockID) (crossSafe eth.BlockID, err error)
}
```
`TryCrossSafe`:
    1. The tip of the `super-safe-db` is checked against L1, to be canonical.
        - Rewind if not canonical.
        - This may be skipped if the L1 chain was recently checked already.
    2. The `derivedFrom` is checked against the `super-safe-db`.
        - If it is `<=` the current tip:
            - If it is canonical: 
                1. Find the latest `crossSafe` for the `chainID` that is `<=` to the input `localSafe`.
                2. Return this to the op-node. The op-node will be able to verify whether the result is canonical, and accept it as new cross-safe, promoting the safety-level of part of its L2 chain.
                    - TODO: need to also return current state of local-safe, to catch invalid executing message in `localSafe` situation.
            - If it is non-canonical: wrong L1 view, return error.
        - If it is just on top of the current tip:
            - Buffer the data
                - The op-supervisor needs to append a new `super-safe-db` entry,
                    after having collected data from every L2 chain.
            - Return an error, the op-node should retry later.
        - If further ahead:
            - Return a reset error, since we need the op-node to produce older derived-from information.
                - the op-node has to change L1 view.
                - the op-supervisor has to change L1 view.

TODO: we need to make `TryCrossSafe` more robust. The main direction to explore is to make the op-node run derivation based on where the supervisor says it needs data, rather than trying to push things from the op-node to the supervisor at seemingly arbitrary times, to avoid difficult synchronization issues.

### `TryFinalize`

```go
type RPC interface {
    TryFinalize(chainID types.ChainID, finalizedL1 eth.BlockID) (finalizedL2 eth.BlockID, err error)
}
```

- `TryFinalize`: this simply returns the latest L2 block of the specified `chainID` to finalize, which must be fully derived from the specified `finalizedL1` block.

## Integration

### op-node safety

- Track `cross-unsafe` as new safety-level in the `EngineController`, similar to pseudo-levels like `backup-unsafe-reorg`, and `pending-safe`.
- Modify the `engine` payload-processing step to not immediately mark a derived-from-L1 block as `safe`, but rather emit an event that indicates the `unsafe` block was derived and ready to promote. Don’t emit the pending-safe update immediately either.
- Add new Interop event-deriver.
    - Monitor for what used to be the promote unsafe→safe event. If pre-interop fork, immediately promote it, no checks. If post-interop fork, check the op-supervisor, with the `TryCrossSafe` RPC.
        - If the supervisor RPC call itself fails, we can return an error, and with some backoff, retry (by requesting the latest promotion-ready block from the engine)
        - If the supervisor indicates a `crossSafe` block, then verify the block, and promote it.
        - TODO: more response handling, depending on API design.
    - Replace finalizer deriver, to simply take the L1 finality signal, forward it to the supervisor, and get back a signal what to finalize on the L2 accordingly. If the L1 finalization signal is too new, that’s ok, eventually we’ll have recent data, or we can pass an older L1 block as input to the supervisor, to get an older L2 block to finalize.

### op-geth safety

During block-building:

- After adding a tx to an in-progress block building job, inspect all the emitted new event-logs by said tx:
    - For each executing message, query the op-supervisor `CheckMessage`, to verify if the message is sufficiently safe to include.
        - Undo the tx if the supervisor returns `conflicts`, and apply back-pressure to the tx-origin.
        - Undo the tx if the supervisor returns `unknown`, and apply back-pressure to the inclusion of interop txs as a whole. The supervisor needs to catch up, but we can continue to build regular blocks.
        - Accept the tx if the supervisor indicates it is `unsafe`/ `safe` / `finalized`; depending on the sequencer setting. If the setting does not allow `unsafe`, then re-try the tx at a later point, when the safety may have been bumped up.
- For txs in the tx-pool: periodically check the safety, and prune the pool accordingly. This is out of scope for devnet 1.

# Risks & Uncertainties

*An overview of what could go wrong. Also any open questions that need more work to resolve.*
