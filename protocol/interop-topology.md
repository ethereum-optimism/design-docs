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
There are two sets of problems to solve in our topology and software arrangement:

## Design Tx Ingress Flow for Optimal Latency, Correctness
It is a goal to remove as many sync, blocking operations from the hot path of
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

## Arrange Infrastructure to Maximize Redundancy
It is a goal to ensure that there are no single points of failure in the network infrastructure that runs an interop network.
To that end, we need to organize hosts such that sequencers and supervisors may go down without an interruption.

# Proposed Solutions

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

## TX Ingress Flow
There are lots of gates and checks we can establish for inflowing Tx to prevent excess work (a DOS vector) from reaching the Supervisor.

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

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

Doing a remote RPC request is always going to be an order of magnitude slower than doing a local lookup.
Therefore we want to ensure that we can parallelize our remote lookups as much as possible. Block building
is inherently a single threaded process given that the ordering of the transactions is very important.

### Single Point of Failure and Multi Client Considerations

There is a single point of failure with the `op-supervisor`. Adding an alternative backend that dynamically
fetches data using `eth_getLogs` rather than looking up its local database helps to approximate a second implementation.

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

## Host Topology / Arrangement

In order to fully validate a Superchain, a Supervisor must be hooked up to one Node per chain (with one Executing Engine behind each).
We can call this group a "full validation stack" because it contains all the executing parts to validate a Superchain.

In order to have redundancy, we will need multiple Nodes, and also *multiple Supervisors*.
We should use Conductor to ensure the Sequencers have redundancy as well.
Therefore, we should arrange the nodes like so:

|             | Chain A | Chain B | Chain C |
|------------|---------|---------|---------|
| Supervisor 1 | A1      | B1      | C1      |
| Supervisor 2 | A2      | B2      | C2      |
| Supervisor 3 | A3      | B3      | C3      |

In this model, each chain has one Conductor, which joins all the Sequencers for a given network. And each heterogeneous group of Sequencers is joined by a Supervisor.
This model gives us redundancy for both Sequencers *and* Supervisors. If an entire Supervisor were to go down,
there are still two full validation stacks processing the chain correctly.

There may need to be additional considerations the Conductor makes in order to determine failover,
but these are not well defined yet. For example, if the Supervisor of the active Sequencer went down,
it may be prudent to switch the active Sequencer to one with a functional Supervisor.

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Block Building

The main alternative to not validating transactions at the block builder is validating transactions
at the block builder. We would like to have this feature implemented because it can work for simple networks,
as well as act as an ultimate fallback to keep interop messaging live, but we do not want to run it as
part of the happy path.

## Multi-Node (redundancy solution)

One request that has been made previously is to have "Multi-Node" support. In this model,
multiple Nodes for a single chain are connected to the same Supervisor. To be clear, the Supervisor software
*generally* supports this behavior, with a few known edge cases where secondary Nodes won't sync fully.

The reason this solution is not the one being proposed is two-fold:
- Managing multiple Nodes sync status from a single Supervisor is tricky -- you have to be able to replay
all the correct data on whatever node is behind, must be able to resolve conflicts between reported blocks,
and errors on one Node may or may not end up affecting the other Nodes. While this feature has some testing,
the wide range of possible interplay means we don't have high confidence in Multi-Node as a redundancy solution.
- Multi-Node is *only* a Node redundancy solution, and the Supervisor managing multiple Nodes is still a single
point of failure. If the Supervisor fails, *every* Node under it is unable to sync also, so there must *still*
be a diversification of Node:Supervisor. At the point where we split them up, it makes no sense to have higher quanitities
than 1:1:1 Node:Chain:Supervisor.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

We really need to measure everything to validate our hypothesis on the ideal architecture.
To validate the ideal architecture, we need to measure it and then try to break it.