# Purpose

The purpose of this document is to progress a design originally proposed by @protolambda and contributed to
by Hamdi [here](https://github.com/ethereum-optimism/specs/pull/88). The goal is to reduce resource usage
by the derivation pipeline when invalid blocks are proposed. This is helpful for ensuring that the fault
proof program cannot be denial of service attacked as well as reducing the latency of reorgs, which is
especially important in the context of interop.

# Summary

New derivation pipeline invariants are introduced that are meant to reduce large resets of the derivation pipeline
when handling invalid batches.

# Problem Statement + Context

The derivation pipeline is set up as a series of stages. It takes work to progress from one stage to the next.
Sometimes this work involves many remote HTTP requests, other times it requires in memory manipulation of
data structures. It's not ideal when there is an error in a later stage that causes a lot of additional work
to be done in an earlier stage to recover. In practice, this looks like some sort of invalid batch was deserialized
and errors during its processing.

> The `DerivationPipeline` has a `Step()` function that is called over and over again to derive the chain,
which ultimately looks like mapping raw batch data into `ExecutionPayload`s. `DerivationPipeline.Step()`
wraps a call to `EngineQueue.Step()` which wraps a call to `AttributesQueue.NextAttributes()`.

Examples of types of errors:
- [Invalid transaction in block](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/batches.go#L156)
- [Invalid Execution Payload](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/engine_update.go#L94)
- [Invalid batch timestamp](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/attributes_queue.go#L84)
- [Batch contains deposit tx](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/batches.go#L160)


# Proposed Solution

The following 2 invariants are introduced to ensure timely progression of the derivation pipeline. In both cases, either a batch
is correctly progressed or it is mapped into a deposits only batch.

If the batch data [makes it](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/batch_queue.go#L223)
to the `BatchQueue`, then the batch is either properly deserialized and validated or is mapped into a "deposits only" batch.
This specifically describes the `BatchDrop` cases returned from [CheckBatch](https://github.com/ethereum-optimism/optimism/blob/8b5e3c264d2a3a114e7df5361acd8d4c1c700f81/op-node/rollup/derive/batches.go#L34). 

If the Execution API on the EL returns [INVALID](https://github.com/ethereum/go-ethereum/blob/7fd7c1f7dd9ba8d90399df2f080e4101ae37a255/beacon/engine/errors.go#L63), then the batch that produced the `ExecutionPayload` is mapped into a deposits only batch before
being attempted again as an `ExecutionPayload`. This particular check captures things like invalid nonces on transactions included
in a block.

With the launch of interop, an additional invariant will be added which involves mapping any batches that contain invalid executing
messages into a deposits only batch.

# Alternatives Considered

## Smaller Solution

It is possible to only enforce one of the two invariants, where we would want to drop the EL `INVALID` invariant in favor of
the batch queue invariant.

# Risks & Uncertainties

- This adds risk around unsafe head reorgs. To date, there has not been an unsafe head reorg. Cases where batches would be
dropped and then there is another chance to progress the safe head via submitting another batch would be turned into
unsafe reorgs. The software and ops have been hardened after running it in production for some time.
