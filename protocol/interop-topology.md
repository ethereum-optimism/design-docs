# Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

This document exists to drive consensus and act on a reference of the preferred topology
of a cloud deployment of an interop enabled OP Stack cluster.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

The production topology of an interop cluster checks the validity of cross chain transactions at:
- cloud ingress (`proxyd`)
- sentry node mempool ingress
- sentry node mempool on interval
- sequencer node mempool on interval

The validity of cross chain transactions are not checked at:
- sequencer node mempool ingress
- block builder

It is safe to not check at block building time because if the check passes at ingress
time and passes again at block building time, it does not exhaustively cover all cases.
It is possible that the remote reorg happens **after** the local block is sealed.
In practice, it is far more likely to have an unsafe reorg on the remote chain that is not
caught **before** the local block is sealed because the time between checking at the mempool
and checking at block building is so small. Therefore we should not add a remote RPC request
as part of the block building happy path, but we should still implement and benchmark it.

TODO: include information on recommendation around standard mode/multi node/running multiple supervisors.
Should include information on the value props of each so we can easily align on roadmap.

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

It is a goal to remove as many async, blocking operations from the hot path of
the block builder as possible. Validating an interop cross chain transaction
requires a remote RPC request to the supervisor. Having this as part of the hot
path introduces a denial of service risk. Specifically, we do not want to have so
many interop transactions in a block that it takes longer than the blocktime
to build a block.

To prevent this sort of issue, we want to move the validation on interop
transactions to as early as possible in the process, so the hot path of the block builder
only needs to focus on including transactions.

For context, a cross chain transaction is defined in the [specs](https://github.com/ethereum-optimism/specs/blob/85966e9b809e195d9c22002478222be9c1d3f562/specs/interop/overview.md#interop). Any reference
to the supervisor means [op-supervisor](https://github.com/ethereum-optimism/design-docs/blob/d732352c2b3e86e0c2110d345ce11a20a49d5966/protocol/supervisor-dataflow.md).

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

## Solution

### `op-supervisor` alternative backend

We add a backend mode to `op-supervisor` that operates specifically by using dynamic calls to `eth_getLogs`
to validate cross chain messages rather than its local index. This could be accomplished by adding
new RPC endpoints that do this or could be done with runtime config. When `op-supervisor` runs in this
mode, it is a "light mode" that only supports `supervisor_validateMessagesV2` and `supervisor_validateAccessList`
(potentially a subset of their behavior). This would give us a form of "client diversity" with respect
to validating cross chain messages. This is a low lift way to reduce the likelihood of a forged initiating
message. A forged initiating message would be tricking the caller into believing that an initiating
message exists when it actually doesn't, meaning that it could be possible for an invalid executing
message to finalize.

TODO: feedback to understand what capabilities are possible

### `proxyd`

We update `proxyd` to validate interop messages on cloud ingress. It should check against both the indexed
backend of `op-supervisor` as well as the alternative backend.

### Sentry Node Mempool Ingress

We update the EL clients to validate interop transactions on ingress to the mempool. This should be a different
instance of `op-supervisor` than the one that is used by `proxyd` to reduce the likelihood of a nondeterministic
bug within `op-supervisor`.

### Sentry Node + Sequencer Mempool on Interval

We update the EL clients to validate interop transactions on an interval in the mempool. Generally the mempool
will revalidate all transactions on each new block, but for an L2 that has 1-2s blocktime, that is quite often.
It may be the case that we could get away with a batch RPC request every 1-2s, but generally we should not do
`n` RPC requests on each block where `n` is the number of transactions that include statically declared executing
messages.

### Block Building

Lets say that it takes 100ms for the transaction to be checked at `proxyd`, checked at the mempool of the sentry node,
forwarded to the sequencer and pulled into the block builder. The chances of the status of an initiating message
going from existing to not existing during that timeframe is extremely small. Even if we did check at the block builder,
it doesn't capture the case of a future unsafe chain reorg happening that causes the message to become invalid.
Because it is most likely that the remote unsafe reorg comes after the local block is sealed, there is no real
reason to block the hot path of the chain with the remote lookups.

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

Doing a remote RPC request is always going to be an order of magnitude slower than doing a local lookup.
Therefore we want to ensure that we can parallelize our remote lookups as much as possible. Block building
is inherently a single threaded process given that the ordering of the transactions is very important.

## Single Point of Failure and Multi Client Considerations

There is a single point of failure with the `op-supervisor`. Adding an alternative backend that dynamically
fetches data using `eth_getLogs` rather than looking up its local database helps to approximate a second implementation.

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Block Building

The main alternative to not validating transactions at the block builder is validating transactions
at the block builder. We would like to have this feature implemented because it can work for simple networks,
as well as act as an ultimate fallback to keep interop messaging live, but we do not want to run it as
part of the happy path.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

We really need to measure everything to validate our hypothesis on the ideal architecture.
To validate the ideal architecture, we need to measure it and then try to break it.