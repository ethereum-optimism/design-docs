# Superchain Backend

# Purpose

<!-- *This section is also sometimes called “Motivations” or “Goals”.* -->

The purpose of this design-doc is to align on this abstraction,
and initial implementation approaches, for integration of a superchain-backend into the OP-Stack.

[messaging specs]: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/messaging.md
[verifier specs]: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/verifier.md

# Summary

To verify cross-L2 interactions as part of interop,
we need to verify the cross-L2 messages as described
in the [messaging specs] and the [verifier specs].

Messages are based on EVM log-events, of which there can be many.
This, multiplied by the number of participating chains,
forms a significant verification workload in the rollup-node.

The Superchain Backend aims to encapsulate this,
and provide optionality in the different ways this workload can be processed.

The Superchain Backend interface is how this is integrated with the op-node,
as well as newer components such as an interop-enabled tx-pool.
There are different possible implementations of this interface.


# Problem Statement + Context

## Context

Components of the Go interop stack:
- (Horizontal) Tx-pool: verify candidate transactions, pre-filter cross-L2 dependencies
- Sequencer / Block-builder: extend chain, including cross-L2 executing messages.
- Driver: asynchronous orchestration of verifier work.
- Derivation: replicate L2 chain inputs from L1 data.
- Superchain Backend: cross-verify L2 chains.

Note that the Superchain Backend does not run the EVM.
It merely verifies cross-L2 receipt inclusion.

## Tradeoffs

Not every node deployment is the same: there are trade-offs between running specialized services and
relying on higher-latency but easily available execution RPCs to service verification queries.
A backend abstracts this away, such that these different options can be chosen freely.

This design-doc describes different options in which this backend can be implemented:
- A standalone service, utilizing an optimized view of the aggregate of all L2 messages.
- An RPC-backed, utilizing L2 execution engines to surface the data just in time.
- A minimal implementation for the purpose of fault-proving the message dependencies.

### Workload approaches

In the case of low-resource users, and increasingly so with a growing number of interoperating chains,
distributing the verification should be supported.
This can be distributed while preserving a high level of security with the help of light-client protocols
that serve the same message-verification interface.

A common data-availability layer between interoperating chains allows for the light-clients to be computation-only.

As an individual verifier, i.e. a subjective view of the L2 chain,
it is possible to verify any form of offchain computation proof of choice.

As an L1 smart-contract verifier, i.e. an objective view  of the chain (insofar the bridge is considered canonical),
it is possible to use any combination of onchain computation proofs,
without relying on secondary security assumptions for the L2 inputs.

For users that do run multiple nodes and/or nodes of multiple L2 chains,
the interface can be used to de-duplicate verification work.
Since the L2 verification is inter-dependent, the verification work largely overlaps,
and is thus better served by a shared service.

The Superchain Backend Interface abstracts all of the above away,
such that the right form of verification can be plugged-in in each context.

### Data

What are the constraints for the backend service implementation? How many receipts do we expect?

Currently OP Mainnet has 120M blocks, or about ~4.5 GB of 32 byte block hashes and indexing structures.

1 hour of block hashes amounts to:
- single L2 chain: `60*60/2 * 32 = 58 KB`
- 100 L2 chains = 5.8 MB

There is an order of magnitude more log-events however:
- Worst case @ 375 gas / log:
  - 30M gas (current limit) -> ~80K log events / block.
  - 5M gas (current OP-Mainnet target throughput) -> ~13K log events / block.
- OP-Mainnet today:
  - OP Mainnet logs per day: [https://dune.com/queries/3699891/6225381/](https://dune.com/queries/3699891/6225381/)
    OP Mainnet at 4M logs per day. -> About 93 logs per block.
- Other L2 chains:
  - Base at 11M logs per day.
  - Zora at 0.5-2M logs per day.
  - Arbitrum at 8M logs per day.
  - Ethereum L1 at 3M logs per day.

The history of these log events sums up to:
- OP Mainnet history (from Dune query) = 1.9B logs.
- Size of just 32 bytes per log = ~ 60 GB.
- Size of receipts in geth (from recent `db inspect`) = 80+ GB.

Caching a day worth of event log hashes is ~130 MB for OP Mainnet.
The worst-case, if a chain is attacked with unbalanced log-event usage, has significantly more logs than that.
This would amount to `13k / 93 = ~ 140` times as many log-events,
or about 18 GB of log-event hashes for a full-day full-capacity attack on one standard-throughput chain.

A standalone-service that prefetches the logs should be able to support the above:
- A trivial amount of memory for the last ~1 hour of block-hashes.
- An in-memory cache for recent ~1 hour of log-events, expected to be the most commonly consumed,
  as well as cover all recent executing messages that are yet to be verified, or form a dependency that may reorg out.
- Disk storage, around ~65 GB per L2 chain of OP-Mainnet scale and history-size,
  for an append-only log of block-hashes and event-log hashes per chain.

Note that there does not have to be any indexing of hashes back to metadata,
as the executing-messages include the block-number and event-log index, for fast lookups.
And a binary-search over the append-only log can find the data quickly:
there is no need for a more free key-value database.
Optionally the hashes may be truncated to ~20 bytes, like ethereum addresses,
to save on storage while still being reasonably cryptographically secure.


## Responsibilities

The responsibilities of the superchain backend are to:
- (pre-)fetch executing messages of each chain.
- Track cross-unsafe-head of each chain.
- Track unsafe head of each chain, to handle reorgs.
- Resolve block and message safety.
- And possible future extensions, post-launch:
  - Track shreds of each chain (inputs to intra-block message safety).
  - Resolve block safety incl. cycles (intra-block message safety).

## Implementation ideas

### Verify as hashes, not messages

Log-events can be huge, their size is variable.
Importantly, an executing message already encodes what is being executed.
Verifying should not require retrieval of the full initiating message.

And there is no "partially valid":
the log-event contract-origin address and log-event topics and data can be hashed into one commitment.

Since initiating messages and executing messages are both log events,
and assuming intra-block messaging is allowed by the protocol, we can:

- use an executing message as initiating message
- execute the same message
- have a loop of executing messages between chains

We should version the hash, and type it, such that we can:

- Distinguish between "any" log event, and a log event that is an executing message and thus depends on another.
- Persist the hashes, and still be able to tell them apart after changing the interop system.

The hash should also be constructed like `H(origin,H(payload))` to not require the full payload,
but just the payload hash, to construct the final message hash.

### Data loading

Transitive dependencies make a lookup unbounded.
If a lookup will take long, we want to abort.
A tx can always be retried, it should not lock up resources.
Loading of initiating messages can take quite a while, this should not be the default critical path.

Ideally we prefetch the data, and persist it efficiently for later lookups and validation.

We can entirely separate all the data loading from validation:

- **Loading**: Load log-events for each new block, transform into interop-hashes,
  and append the block of hashes to a append-log.
- **Validating**: Traverse the append-log, and for each executing hash, check for matching initiating hashes.
  If at any point invalid, then truncate the log to remove any data that depends on it.
  *This is quite similar to what the L1-safedb (op-node FP support feature) does*, except we can fetch ahead,
  and store groups of initiated message hashes rather than L2 blockhashes.

This type of pre-fetched L2 data, in terms of interface is no different from having
an RPC endpoint that can check a log-event at an block/index and retrieve the cross-unsafe label.
But since it is pre-fetched, it can serve queries much faster.

### Optimistic verification

Not all L2 chains will synchronously be available and fully verified.
Optimistically we can pre-fetch all initiating/executing message hashes,
but verification depends on multiple L2s being available.
Thus, there may be initiating messages past the "cross-unsafe" progression,
that we can assume for some level of cross-L2 guarantee, but are not otherwise safe to use yet.

### Messaging Shreds

The idea of streaming partial blocks can be implemented as extension of the above described append-log model;
the tip of the append-log is maintained by the shreds, and does not have to be fetched anymore.
This can happen in-memory, with periodic flushes to disk.

### Messaging "Horizon"

When prefetching data, any individual L2 chain must not go too far ahead before other L2s catch up,
to keep work balanced and verifiable. The "horizon" is the furthest (into future) timestamp that can be seen,
and can be entirely verified with data up to that point, thanks to the timestamp invariant.

Using this "horizon" property, nodes can implement a form of back-pressure,
where it is known exactly whether or not all data has been retrieved to verify remaining open cross-L2 dependencies.

### Message safety is not different than block safety

Consider the following block safety levels, and their messaging equivalents:

```
enum SafetyLevel {
    INVALID, # conflicting/invalidated data, to be removed from the canonical view
    UNSAFE,  # not cross-verified anything, yet.
    CROSS_UNSAFE,  # cross-verified L2
    SAFE,  # verified L1 also
    FINAL,  # irreversible
}
```

When considering the global log of interop hashes per L2 chain,
the safety level label just moves along that log, like it does on the chain of blocks.

### Simple interface:

The verifier side is really all just one interface:

```
type MessageVerifier interface {
    Check(Identifier, PayloadHash) SafetyLevel
}
```

This can be backed by:

- Wrapper around an ethereum RPC:
    - using `eth_getLogs`, fetch log with 1-block 1-contract narrow scope, check if it matches,
      return safety-level of block it was included in.
    - using `eth_getTransactionReceipt` (if tx hash is known), fetch logs of specific tx, rely on metadata for event-index,
      return safety-level of block it was included in.
    - using `eth_getBlockReceipts`, fetch all logs of block, check the specific log,
      return safety-level of block it was included in.
- Wrapper around a append-log "database", as described in the ideas above:
  search the database for the data, check against computed message-hash, and then return safety level.
- Wrapper around a shreds reader: if we maintain the tip of the chain in memory based on streamed data,
  we can serve queries for that data, and otherwise fallback to another source.

In a fault-proof, a trustless version of the `eth_getBlockReceipts` can support the query,
as that verifies the log-index correctness relative to the start of the block, and inclusion-proof to the block etc.

### Batching

We may want to consider batch-queries as part of the interface;
query `[]BatchElem` (`BatchElem = (Identifier, PayloadHash)`), return `[]SafetyLevel`.

L2 Blocks already form a natural form of batching, this should be supported by the interface also,
but may not be applicable for per-message systems like the tx-pool.

## Fault proving

Note that chain-procesing is optimistic and asynchronous: chains are individually processed,
and cross-L2 messaging safety is deferred to an asynchronous process.

To support this in fault-proving, the "loop" of block-processing has to be enhanced with this cross-L2 verification.
To avoid reorgs in the fault-proof, the block-processing and cross-L2 verification should be interleaved,
to run one step at a time each.

With an additional simplification of not allowing equivocation of batch-data,
and falling back to a safe deposits-only L2 block, the proof is simplified to running one block of each
L2 and then performing an aggregate cross-L2 verification step of this block-height.

The "backend" here is backed by pre-image oracle queries that:
- inspect the emitted tentative event-log hashes of the latest block-height, to extract executing messages
- inspect the matching event-logs of the historical data, to retrieve initiating messages (if valid)


# Alternatives Considered

<!-- *List out a short summary of each possible solution that was considered.
Comparing the effort of each solution* --> 

## Original superchain-backend draft, with recursive resolution, RPC backing

Specs draft of @hamdiallam, end February:
interop: superchain backend service [`specs #58`](https://github.com/ethereum-optimism/specs/pull/58)
This describes a recursive message-safety resolver.
This is a DoS-prone default. Ideally the specs describe a system without recursion.
The spec, as well as the below implementations, also still assume the pre-3074 executing-message approach.

PR drafts from @hamdiallam, from Feb-April:
- feat(interop): superchain backend [`monorepo #9612`](https://github.com/ethereum-optimism/optimism/pull/9612) (Feb 20)
    - Standalone service.
    - RPC server with method to check safety of single message (non-transitive safety).
    - CLI config with RPCs.
    - RPC client binding for fetching EVM logs of a block (using `eth_getLogs`) and block header data.
    - Verbose executing-message parsing.
- feat(interop): verifier safe head progression [`monorepo #9753`](https://github.com/ethereum-optimism/optimism/pull/9753) (Mar 6)
    - Stacked on top of `#9612`.
    - Adds superchain backend as sub-component of op-node.
    - Adds superchain into derivation pipeline for safety-interface.
    - Duplication of some superchain-backend message safety,
      executing-message parsing and CLI code, possible due to git retarget issues.
- wip(interop) block dependency graph [`monorepo #10044`](https://github.com/ethereum-optimism/optimism/pull/10044) (April 4)
    - Stacked on top of `#9612`.
    - `BlockDependencies` that holds on to safety information.
    - Some doubts about the `dependents` map with key uniqueness / reorg safety.
    - Global lock on data.
    - Recursive block-safety algorithm, unable to handle loops.
    - Centered around block-safety.
    - Needs work for determining safety of initiating messages transitively efficiently.
    - All in-memory, no back-pressure or limits.

From the above we can use the draft of the standalone backend service, but will need to change:
- The executing message parsing: detect from receipts instead
- The safety-resolution: use an append-log model instead, to avoid recursion
- Maintain only recent history in memory, flush the older history to disk, to not go OOM.
- Revisit reorg protection: we do need checkpoints to determine how to roll back the log,
  but should be able to do so without the current separate data mappings.

# Proposed Solution

<!-- *A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API) is likely too low level.* -->

## Closing draft PRs

We close the specs PR and the 3 draft PRs.
Parts of the implementation are still applicable, this can be cherry-picked with attribution.

## Superchain Backend specs

A new specs PR should describe the interface (without enshrining batching),
as well as a simplified version of a standalone backend.

Also note that the specs should be updated to cover the edge-cases of intra-block event ordering.
An invariant for event-log index should be introduced.

The interface:
```
type SafetyLevel # see design-doc

type Verifier interface {
    CheckMessage(Identifier, PayloadHash) SafetyLevel
    CheckBlock(ChainID, BlockHash, BlockNumber) SafetyLevel
}
```

Summary of simplified implementation to include in specs:
- One append-log per chain
- Safety labels (pointer to block-number) per append-log
- Hashing method to turn event-log (initiating or executing message) into versioned hash
- Append-log entries as follows:
  - Block metadata (prefixed to be distinguishable from event-log hashes)
  - Event log hashes (concatenated)
- Binary search to find a log entry by block-number
- Reorgs by truncating the append-log to the last common block before the reorg divergence

## Superchain Backend implementation: `op-resolver`

> *Name suggestions welcome, `op-superchain` seems to generic however.*

This service resolves superchain dependencies, and implements the above proposed spec for a standalone implementation.

A later iteration of this can support the ideated shreds streaming,
coupled to the block-builder and horizontal tx-pool,
if and when interop enters super-low-latency territory with intra-block cross-L2 messaging.

Additionally we add a cache for initiating messages, to avoid frequent disk reads of the append-log.

We may be able to shard / distribute the service by isolating the maintenance of each chain append-log,
and aggregating the verification to handle cyclic-dependencies:
a service may publish the first unresolved "stuck" block of dependencies to another replica,
which can then resolve some of them, and forward to the next,
until all replicas have seen all messages at the "stuck" block height.
More optimized resolution may be possible, to be explored at a later point.
Cyclic dependencies are out-of-scope for now:
sharding without cycle handling should be trivial by attaching superchain-backends of different chains
to one another and checking message-safety through the standard backend interface.

### Data-model

- Cache for checks on initiating messages
    - Cache miss:
        - if tip of chain: check shreds
        - if within scope: wait for block fetching
        - if old: fetch individual message
- Initiating messages and executing messages have a total order within their block (event log index)
    - needs to be enforced to avoid time-travel hacks
      (e.g. emit an event that assumes admin permission, and then follow up by giving yourself admin permission)
    - can maybe be loosened for some specific bridges/assets, to allow flash-loans of these assets
- Messages are hashed into message-hashes; no point in keeping full message payload around
- Group messages by their blocks / chains, to not duplicate metadata
- Warning: executing events can also count as initiating events
    - We need to specify if we allow an executing message to consume itself,
      i.e. if initiating is always before executing, if log-index matches.
    - We can also have cross-chain loops; executing message is used as initiating message and executed by other chain,
      and executing message on other chain is what is consumed as initiating message by original executing message.

## Embedded RPC-based implementation

As alternative to running a full `op-resolver` service, we may support a fallback in the op-node.
Message safety is checked by relying on the local execution-engine
for serving full event-logs and block-safety labels of the local chain,
and then attaching to another external backend 
(possibly served by an op-node / execution-engine also) to verify the safety of external messages.

## Fault Proof implementation

As described in the implementation-ideas, the fault-proof work of the superchain-backend reduces to accumulation of
event-log hashes and aggregate checking of the initiating messages, at each L2 block height traversed in the proof.
This fits in the backend interface as follows:
- Cross-unsafe level safety is assumed for the previous block-height,
  as safety checks are interleaved with regular processing one step at a time, it is all synchronous.
  The "horizon" of new L2 blocks to cross-verify is essentially extended by 1 block at a time, to keep proving simple.
- The executing and initiating message checks need to be in order of event-log,
  to prevent any intra-block issues like cycles and time/ordering related cross-L2 tricks
  (not enforcing intra-block ordering would enable arbitrary flashloan like behavior).

## Integration into op-node

After the completion of the
[event-driven driver](https://github.com/ethereum-optimism/design-docs/blob/main/op-node-derivers.md)
it should be possible to plug in the superchain-backed as an asynchronous verifier in the Driver module of the op-node.

The backend events would roughly consist of:
- Consume events:
    - New unsafe block
    - Reorg (if we want to support flip-flop reorgs, and not just wait for new blocks)
- Produce events:
    - Cross-unsafe reorg
    - Unsafe to cross-unsafe promotion


# Risks & Uncertainties

<!-- *An overview of what could go wrong. Also any open questions that need more work to resolve.* -->

## Throughput

The embedded version in the op-node is too high latency:
interop messaging may be faster or higher-throughput than an inefficient standard-RPC based
implementation without prefetching or aggregating can keep up with.

We may introduce non-standard RPC methods,
to avoid overhead like fully encoding the receipts and hydrating their metadata.
The event-log-filter query is already as narrowly scoped as possible (single block / contract),
but is still effectively backed by receipts-retrieval in op-geth.

If this proves to be too slow, then there still is the standalone service that benefits from prefetching
and aggregate data handling (no receipts, just efficient message-hash lookups).
In the long-term, with the growth of interoperating chains, light-clients (for just the cross-L2 part)
may support the under-resourced nodes that are unable to run this service
in addition to the current rollup-node and execution-engine stack.
