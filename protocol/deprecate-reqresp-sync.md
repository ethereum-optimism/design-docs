# Deprecate request-response CL sync

# Summary

Deprecate CL-sync (request-response reverse-blocks sync, not block gossip)
in favor of EL-sync, with improved sync-target reliability.

This simplifies the op-node configuration, removes complicated sync-code,
and makes the op-node more robust against P2P-network interruption.

# Problem Statement + Context

Currently the op-node has a "request-response" P2P sub-system to fetch blocks,
verify them in reverse order against a sync target, and apply them to the EL node.
This sub-system is complicated and fragile, and has existing better alternatives.

## History

### gossip-only sync

The original op-node implementation supported GossipSub block distribution only.
And the P2P functionality of the EL was unused.

### req-resp reverse-sync

One of the last features to land before Bedrock launch was reverse-sync,
using the `request-response` protocol functionality of LibP2P.

Since block-signatures are not stored, blocks can only be authenticated by their block-header parent-hash relation.
This means that the op-node has to fetch blocks in reverse order: the blocks cannot immediately be processed.
Parallelizing the fetching of these unauthenticated blocks, and then resolving the canonical chain, is complicated.
And sync-depth is limited, due to memory limitations (the idea was to not introduce a database for just for syncing).

### Introduction of Snap-sync

Proto/Roberto worked on some snap-sync experimentation around Devconnect 2022 Istanbul.
This was successful, but integration was lacking, and it only worked for Base:
we had to jump through a few hoops, to handle stateless genesis for OP Mainnet,
which has a non-zero bedrock-migration "genesis" state.

[Mid 2023](https://github.com/ethereum-optimism/op-geth/pull/105) Sunnyside Labs then
owned Snap-sync development for a while, improving the usability.

### Introduction of SyncMode

[End 2023](https://github.com/ethereum-optimism/optimism/pull/8333) `trianglesphere` then improved the
integration again, by introducing a "sync-mode" configuration to op-node.

### P2P req-resp refresh attempt

[July 2024] New help with P2P was meant to onboard
[with context](https://www.notion.so/oplabs/op-node-P2P-context-b806628dd29c45acbe69dac2b02c63fb)
Followed by [a P2P req-resp rewrite](https://github.com/ethereum-optimism/optimism/pull/11781) that was never merged
due to various setbacks.

## Retrospective

- The stack started with block gossip without EL sync
- Then introduced snap-sync and introduced different sync-modes
- Then effectively reached a complexity-ceiling when more changes were needed

# Proposed Solution

Less code is better code.
The EL node has a specialized sync implementation.

Specifically, EL nodes can sync to the tip of the chain, when connected to healthy nodes,
and when given a full target block-header to sync to.

`op-geth` does so by maintaining a DB of synced blocks (i.e. memory is not a limit to syncing),
and maintaining a "skeleton chain" of block-headers to connect between the known chain and the target.
This also handles a moving target, and reorgs during sync.

### Ensuring a sync-target

With gossip-sub, target block-headers are generally available.

The main exception is when a node restarts, and when gossip-sub is idle (e.g. sequencer is offline),
then nodes might be unable to get a new sync-target.

We can solve this by implementing the following change:
- Cache the last gossiped block with signature.
- Change the gossip-relay rules:
  accept older blocks that don't meet the wallclock validation rules, but only up to a certain rate-limit.
  Whenever an older block is received than the cached last block,
  then gossipsub-`IGNORE` the older block, and publish the cached block.
  This prevents stale blocks from being replayed without bound,
  as the cached block will be replayed instead, which more nodes can agree on and filter out.
- (Maybe) introduce an extremely simple request-response protocol,
  "poke", that requests a node to share its last signed block.
  When idle, or just connecting, this can help start sync.

### Simplifications

To roll out, we need:
- Review sync errors / state returned by the engine API,
  and clarify in op-node metrics/logs what the sync-state is.
- Make EL-sync the default. We can keep CL-sync, but e.g. add a deprecation warning.
- Sync-test that we can rely solely on EL-sync.
- Update specs to deprecate the request-response block sync.
- Update docs to remove CL/EL-sync differentiation.

To polish, we can:
- Improve the tx-pool, to not be a yes/no filter, but allow e.g. filter by network IP and/or nodeID.
  This can make replica topologies simpler, to have more replicas publicly connected and serve EL-sync.
- Clean up the new-payload code-path in op-node, to use the same block-processing events as block-building,
  rather than the separate synchronous `NewPayload` call on the engine-controller.

To finalize, we:
- Remove CL-sync / EL-sync differentiation from op-node:
  - Remove the gossiped CL-sync payloads-buffer.
  - Remove the request-response block-serving.

## Resource Usage

The resource-usage of the op-node will improve downwards,
as it no longer has to buffer large amounts of blocks in memory.

## Single Point of Failure and Multi Client Considerations

Different EL nodes may be implement sync in different ways,
but all support the general block-by-block L1 sync protocol, and should be able to handle the syncing.
Worst-case sync is still the same, where the op-node derives block by block from L1 data, without P2P syncing.

# Alternatives Considered

Repairing and testing CL req-resp-sync at this point is going to require more resources,
and nets out to a worse outcome, where the op-node has additional code and resource-usage.

# Risks & Uncertainties

EL-nodes are assumed to be generally connected, and sufficiently available for sync,
since snap-sync has been in production for an extended time.

However, EL-node discovery can be improved to make more nodes visible,
and may be improved by [fixing this op-geth issue](https://github.com/ethereum-optimism/op-geth/issues/326)
that [introduces EL ENR discovery attribute](https://github.com/ethereum-optimism/specs/pull/150).

