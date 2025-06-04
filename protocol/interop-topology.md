# Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

This document exists to drive consensus and act on a reference of the preferred topology
of a cloud deployment of an interop enabled OP Stack cluster.

# Summary

The creation of Interop transactions opens Optimism Networks to new forms of undesirable activity.
Specifically, including an interop transaction carries two distinct risks:
- If an interop transaction is included which is *invalid*, the block which contains it is invalid too,
and must be replaced, causing a reorg.
- If the block building and publishing system spends too much time validating an interop transaction,
callers may exploit this effort to create DOS conditions on the network, where the chain is stalled or slowed.

The new component `op-supervisor` serves to efficiently compute and index cross-safety information across all chains
in a dependency set. However, we still need to decide on the particular arrangement of components,
and the desired flow for a Tx to satisfy a high degree of correctness without risking networks stalls.

In this document we will propose the desired locations and schedule for validating transactions for high correctness
and low impact. We will also propose a desired arrangement of hosts to maximize redundancy in the event that
some component *does* fail.

# Problem Statement + Context

Breaking the problem into two smaller parts:

## TX Flow - Design for Correctness, Latency
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

## Redundancy - Design for Maximum Redundancy
It is a goal to ensure that there are no single points of failure in the network infrastructure that runs an interop network.
To that end, we need to organize hosts such that sequencers and supervisors may go down without an interruption.

This should include both Sequencers, arranged with Conductors, as well as redundancy on the Supervisors themselves.

# Proposed Solutions

## TX Ingress Flow
There are multiple checks we can establish for inflowing Tx to prevent excess work (a DOS vector) from reaching the Supervisor.

### `proxyd`

We can update `proxyd` to validate interop messages on cloud ingress. It should check against both the indexed
backend of `op-supervisor` as well as the alternative backend.
Because interop transactions are defined by their Access List,
`proxyd` does not have to execute any transactions to make this request.
This filter will eliminate all interop transactions made in bad faith, as they will be obviously invalid.

It may be prudent for `proxyd` to wait and re-test a transaction after a short timeout (`1s` for example)
to allow through transactions that are valid against the bleeding edge of chain content. `proxyd` can have its own
`op-supervisor` and `op-node` cluster (and implicitly, an `op-geth` per `op-node`), specifically to provide cross safety queries without putting any load on other
parts of the network.

### Sentry Node Mempool Ingress

We can update the EL clients to validate interop transactions on ingress to the mempool. This should be a different
instance of `op-supervisor` than the one that is used by `proxyd` to reduce the likelihood of a nondeterministic
bug within `op-supervisor`. See "Host Topology" below for a description of how to arrange this.

### All Nodes Mempool on Interval

We can update the EL clients to validate interop transactions on an interval in the mempool. Generally the mempool
will revalidate all transactions on each new block, but for an L2 that has 1-2s blocktime, that could be frequent if the
RPC round-trip of an `op-supervisor` query is too costly.

Instead, the Sequencer (and all other nodes) should validate only on a low frequency interval after ingress.
The *reasoning* for this is: 

Lets say that it takes 100ms for the transaction to be checked at `proxyd`, checked at the mempool of the sentry node,
forwarded to the sequencer and pulled into the block builder. The chances of the status of an initiating message
going from existing to not existing during that timeframe is extremely small. Even if we did check at the block builder,
it doesn't capture the case of a future unsafe chain reorg happening that causes the message to become invalid.
Because it is most likely that the remote unsafe reorg comes after the local block is sealed, there is no real
reason to block the hot path of the chain with the remote lookups. If anything, we would want to coordinate these checks
with the *remote block builders*, but of course we have no way to actually do this.

### Batching Supervisor Calls

During ingress, transactions are independent and must be checked independently. However, once they've reached the Sequencer
mempool, transactions can be grouped and batched by presumed block. Depending on the rate of the check, the Sequencer
can collect all the transactions in the mempool it believes will be in a block soon, and can perform a batch RPC call
to more effectively filter out transactions. This would allow the call to happen more often without increasing RPC overhead.

### Note on Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

Doing a remote RPC request is always going to be an order of magnitude slower than doing a local lookup.
Therefore we want to ensure that we can parallelize our remote lookups as much as possible. Block building
is inherently a single threaded process given that the ordering of the transactions is very important.

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

## Solution Side-Ideas

Although they aren't strictly related to TX Flow or Redundancy, here are additional ideas to increase the stability
of a network. These ideas won't be brought forward into the Solution Summary or Action Items.

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


# Solution Summary

We should establish `op-supervisor` checks of transactions at the following points:
- On cloud ingress to `proxyd`
- On ingress to all mempools
- On regular interval on all mempools

Additionally, regular interval checks should use batch calls which validate at least a block's worth of the mempool
at a time.

`op-supervisor` checks of transactions should *not* happen at the following points:
- sequencer node mempool ingress
- block building (as a synchronous activity)

When we deploy hosts for an inter-operating set of `n` chains, we should currently use a "Full Validation Set" of one Supervisor plus `n `Managed nodes,
to maximize redundancy and independent operation of validators. When Sequencers are deployed, Conductors should manage
individual Managed Nodes *across* Supervisors.

# Alternatives Considered

## Checking at Block Building Time (Tx Flow Solution)

The main alternative to not validating transactions at the block builder is validating transactions
at the block builder. We would like to have this feature implemented because it can work for simple networks,
as well as act as an ultimate fallback to keep interop messaging live, but we do not want to run it as
part of the happy path.

## Multi-Node (Host Redundancy Solution)

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
be a diversification of Node:Supervisor. At the point where we split them up, it makes no sense to have higher quantities
than 1:1:1 Node:Chain:Supervisor.

# Risks & Uncertainties

We really need to measure everything to validate our hypothesis on the ideal architecture.
To validate the ideal architecture, we need to measure it and then try to break it.

Incorrect assumptions, or unexpected emergent behaviors in the network, could result in validation not happening at the right times,
causing excessive replacement blocks. Conversely, we could also fail to reduce load on the block builder, still leading to slow
block building or stalls.

Ultimately, this design represents a hypothesis which needs real testing before it can be challenged and updated.

# Action Items from this Document
- Put the `proxyd` check in place
- Put an interval check in place in the mempool
- Remove build-time checks
- Test RPC performance (data collectable via Grafana)
- Consider and add Supervisor-health as a trigger for Conductor Leadership Transfer
