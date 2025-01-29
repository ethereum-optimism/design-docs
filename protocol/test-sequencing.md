# Summary

Interop testing requires completion of a long
[checklist](https://github.com/ethereum-optimism/optimism/issues/13855) of particularly constructed blocks and reorgs.
To implement these tests, we need new tooling that can include transactions and perform reorgs in exact ways.

While implementing the test tooling, we have an opportunity to prepare separation of sequencing from the op-node,
to reduce op-node complexity and encapsulate sequencing concerns better.

# Problem Statement + Context

Sequencing in the op-stack can be bucketed into different types:
- Legacy sequencing (tx-pool, op-node with op-geth, recommit, priority fee)
- Flashblocks (rollup-boost proxy between op-node and op-geth, mux to builder node, 250ms incremental block building)
- Testing (action-tests, next-up interop work)

The testing type of work is in need of change, to provide the level of block-building control
that [action-tests](https://github.com/ethereum-optimism/optimism/tree/develop/op-e2e/actions)
have to fully end-to-end tests like those backed by
[Kurtosis](https://github.com/ethereum-optimism/optimism/tree/develop/kurtosis-devnet).

Instead of building a limited testing tool, we can consider building a full sequencer service:
- Op-node complexity can be reduced.
- Separate sequencing from verifying, to not unnecessarily disrupt sequencing work.
  This is a known issue seen by chains who have heavy / spiking verification workloads.
- Block-building in op-geth is not a good starting point.
    - On every tx, geth [hashes the full block-header again, unnecessarily](https://github.com/ethereum-optimism/op-geth/pull/462).
    - Block-building does not prepare a cache for block-processing.
    - Block recommit code is complicated, wrangled into the geth block-building `miner` package.
      Regardless of the event-control we add to op-node, this legacy geth code makes testing impractical.

# Proposed Solution

This is an idea to separate the sequencing from op-node into its own service with test-RPC.
We could call this service `op-sequencer`, and maybe use a different name while
it's only servicing testing and is still more experimental.

For testing and experiments, the service should build blocks manually, with full control over reorgs/inclusion/ordering.

For regular operation, and rbuilder-backed flashblocks,
we can make the service fall back to only run the block-proposing part of sequencing: see "engine mode" section.

## Increments

### Testing scope

Test-functionality: Build the block without the engine API.

The test-functionality is inspired a lot by the action-tests:
having exact control over block-construction is very useful for testing.
And by putting it in a service, we can use it more places than action-tests, like Kurtosis / devnets.

Interop testing:
the [checklist](https://github.com/ethereum-optimism/optimism/issues/13855) is long.
Most of the tests require either specific tx-inclusion, specific reorgs, or a combination of both.

We need some tool or thing to build these blocks with these transaction/reorg controls.

Instead of using the engine-API, we build the block against a forked state.
We already do this kind of thing in the
[`op-run-block`](https://github.com/ethereum-optimism/optimism/tree/develop/op-chain-ops/cmd/op-run-block) debug tool.

Building the testing functionality consists of:
- Host a tiny subset of the EL RPC:
    - `eth_sendRawTransaction`: for tx flow. Op-geth nodes could configure this as sequencer-endpoint RPC to relay txs to.
    - `eth_chainId`: maybe, for chain check.
- Import the geth tx-pool. The tx-pool journal and pool maintenance code is already quite decoupled in geth.
- Connect to geth node, via debug RPC, for remote state-access:
    - We can seal the block with a `stateRoot` this way.
- Reuse code from op-node to connect to op-signer, to sign blocks.
- Add a RPC method to op-node to insert the block into the canonical chain, and publish it to gossip, with a sequencer signature.
  Related problem: the existing op-node `admin_postUnsafePayload` (used by op-conductor) method doesn't
  include the block signature, so it cannot relay the block via gossip to other nodes.
- Host new endpoints:
    - `test_open(opts: {l2Parent, l1Origin}) -> jobid`: start building a block
    - `test_includeBest(jobid)`: include best next tx from the tx-pool
    - `test_includeFirst(jobid, address)`: include best next tx from specific address from tx-pool
    - `test_includeByHash(jobid, txHash)`: include specific tx from tx-pool
    - `test_includeRaw(jobid, rawTx)`: include specific given tx
    - `test_cancel(jobid)`: stop building a block
    - `test_seal(jobid)`: complete building a block
    - `test_auto(active)`: automatically repeat block-building
    - `test_inspect(jobid) -> partial block`: inspect thus-far built block

These RPC calls can then be used by tests to include the transactions in the desired order, in the specific blocks,
to create the right test scenario. E.g. coordinating a cyclic transaction dependency,
or including an otherwise invalid transaction (invalid executing message).

#### L1 mode

L1-block building shares a lot of code with L2, and can be done for testing as well.
No system-transactions or formatted extra attributes like the Holocene `extraData` need to be inserted.

One way to use this for L1 would be as "mock": to make the chain of blocks proactively.
This can support shadow-forking (since L1 consensus is not followed), and avoid the need to run a beacon-node at all.

Another way might be to intercept the engine-API `engine_forkchoiceUpdated(fcState, attributes)`,
which can then queue the block-building work and complete it when there is a matching test block-building job.
This is similar to the sidecar design in OP-Stack.
However, this then makes the op-node lead, and prevents reorgs from being initiated by the test tool.
The "mock" L1 mode will thus be prioritized.

#### Compared to op-wheel

`op-wheel` may be time to deprecate. Full deprecation is out of scope for this design doc,
but decision is made to not add additional things to `op-wheel`.

The `op-wheel` CLI consists of `db` (patch geth DB contents) and `engine` utils.

The forkchoice / reorg CLI tools are still used,
but may be possible to move to op-node or op-geth.

The `engine` CLI tools are mostly one-off commands,
not suited to build blocks with particular transactions, nor a kurtosis standalone service.

There is an `engine auto` command for automatic block-building of L1 blocks (no L2 system deposits)
on an interval using the engine-API.
And it provides none of the testing or production sequencer functionality.

### Production

What if we want to make the test sequencer a production sequencer?

The following integration work is needed:
- Use `auto` sequencing mode by default
- Detect when op-node is deriving conflicting blocks, and automatically back-off.
    - When the last seen parent-block is not the tip of the chain.
    - When `unsafe == safe`, aka likely syncing from L1 data.
- Host the same sequencer admin RPC as op-node, to hande sequencer rotation:
    - `admin_startSequencer`: start
    - `admin_stopSequencer`: stop
    - `admin_sequencerActive`: sequencer state
    - `admin_postUnsafePayload`: publish a block, for failover
    - `admin_overrideLeader`: used by op-conductor
    - `admin_conductorEnabled`: used by op-conductor
- Connect to geth P2P, to read/write the shared tx-pool.
- Support the DA-throttling RPC.
  Or better, implement proper multi-block DA congestion safe-guards.

### Engine mode

To not only build blocks locally, we can make the service connect to a regular engine-API endpoint,
and use that to build the blocks.

To keep the op-node in sync, forkchoice updates will need to be directed at a new op-node RPC,
to keep the op-node forkchoice in sync, while passing through the calls to the actual engine API.

This won't support the test capabilities, but does enable migration away from op-node sequencing.

The service essentially becomes an interface for the op-conductor setup to work with any EL builder setup.

**This includes rollup-boost**: without modifications to rollup-boost, we can migrate sequencing away from the op-node.
[Flashblocks utilizes a sidecar design](https://github.com/ethereum-optimism/design-docs/pull/86),
called rollup-boost, that enables flash-blocks to be flexibly attached between the services.

### Interop experimentation

With some communication between the build processes, we can coordinate transaction-inclusion across chains.
E.g.:
- Host multiple builders in the same Go process.
- Connect multiple instances with streams.

Following a stream (the "shred" draft in older interop specs),
we can signal when events happen, and when they are rolled back (callstack revert, or tx not included).
The other builder can then use the assumed events, or even await them, to build the block of the other chain.

In the extreme, this allows synchronous EVM calls between chains, without interop spec changes.

With the remote-state block-building, and the tx-pool access without running a full node,
this could potentially also serve as a replica-node, following the "horizontal tx-pool" design.

## Resource Usage

For testing, this separates experimental code from the production sequencing,
to not affect resource-usage or related processing timing constraints.

By encapsulating the sequencer work, we can monitor and scale it independent of regular verifier work.

Long-term this can open up more scalable block-building for interop.

## Single Point of Failure and Multi Client Considerations

If we decide to productionize this service after test-tooling, we can support the same set of admin RPCs,
allowing sequencer-rotation to work the same as it does today.

# Alternatives Considered

## Extending op-geth

The block-building described here conflict with that within the `miner` package of geth:
- the `miner` package block-building recommits blocks.
- the `miner` package pulls in transactions from the pool eagerly.
- the `miner` package is a difficult diff to upstream geth, something we do not want to expand.

## Extending op-wheel

The `op-wheel` service set up as a CLI tool, not as a service.
`op-wheel` has a range of functions that can be deprecated.
Expanding the tool now takes on debt of the tool, and makes it more difficult to deprecate parts.

## Extending op-node sequencer internals

The op-node only has access to the building of the initial block template,
and the start/stop timing of block building jobs.
It does not have tx-pool access, ability to order transactions, or EVM or state access.
The sequencer code-path in op-node is a critical-path, and must not break due to test changes.

## Direct engine-API vs op-node RPC

The service needs to connect to the op-node to apply forkchoice updates,
since it otherwise can conflict with the verifier work.

This means that the "engine mode" cannot use the engine-API directly, if there is a live op-node.

## As sidecar between op-node and op-geth

Inserting this as sidecar between op-node and op-geth is possible, but adds extra complexity, especially w.r.t. reorgs.
See exploration in the "L1 mode" section about this.

# Risks & Uncertainties

Slow remote-state access can limit the usefulness in non-test settings.
When the node is close, this may not be a problem.
With the test usage first, we can see what sequencing in this manner is like,
before making any changes to production sequencing code paths.
One mitigation to slow state-access might be to serve remote state in a more efficient format or better transport.

Alternative block-building code might conflict,
but known changes like the rollup-boost/flashblocks design uses a sidecar approach and can be supported still.
