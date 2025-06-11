# Interop Auto-Stop Mechanisms

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Axel Kingsley_                                     |
| Created at         | _2025-05-29_                                       |

## Purpose

This document outlines the design for an automatic stop mechanism that prevents further interop message processing when invalid messages are detected.
This serves as a protective failsafe to avoid continued or repeated inclusion of bad Interop Messages when instability is detected.


## Summary + Problem Statement + Context

When an invalid interop message is included in a block (and is gossiped to the network), there is no simple protocol-supported way to quickly repair that mistake. The Chain Operator must either:
- Reorg the Sequencer to remove the Invalid Message. Nodes won't pick up on this reorg until the Batch is submitted to the L1.
- Wait for the Invalid Message to be included in a Batch on L1. Nodes will then perform a Block Replacement, effectively reorging back to the Cross-Unsafe point.

In both cases, the network experiences a stall from the Block of the Invalid Message to the L1 publishing. Whether building an alternate chain or just letting Nodes derive empty Blocks,
there are negative UX externalities that become unavoidable once the Invalid Message is included.

So it then becomes very important to properly filter Interop Messages before they can be built into a block. From the [Tx Handling Topology Design Doc](./interop-topology.md), we decided to filter:
- On cloud ingress to proxyd
- On ingress to all mempools
- On regular interval on all mempools

And yet there *remains* the possibility that invalid Interop Messages will be included in a block. This is because `CrossUnsafe`, the safety level used for filtering,
may revert if the Remote Chain's contents should change. The Remote Chain has limited opportunity to do this outside of being compromised, but in general Sequencers could equivocate blocks, or post L1 content which conflicts with the Unsafe Chain.

So it then becomes *even more* important for Chain Operators to have a mechanism to protect themselves if one of their Remote Chains degrades filtering effectivness by issuing conflicting L2 data.
Furthermore, there is a vicious cycle aspect - reorgs are a cause of bad interop messages, and bad interop messages are a cause of reorgs. Notionally, it would be prudent for operators to disable Interop when the Superchain is unstable. 

## Proposed Solution

I propose an [Andon Cord](https://en.wikipedia.org/wiki/Andon_(manufacturing)) system which can effectively shut down Interop Inclusion.
It will be triggered in response to certain derivation events, and can also respond to automatic and manual API.

### Trigger Conditions

The auto-stop mechanism can be triggered through three distinct paths, and each trigger path serves a different purpose:
- Derivation events provide immediate response to known issues
- Monitoring signals offer proactive protection based on high levels of insight
- Manual control gives operators direct intervention capability

1. Derivation Events
   - When the Sequencer performs a Block Replacement during Derivation
   - This indicates that an Invalid Message was included and the chain is taking corrective action
   - The Sequencer can automatically trigger the stop as part of this recovery process

This may be the earliest moment AutoStop may trigger, if the invalid message is only discovered via L1 publishing.

2. Monitoring Service Signals
   - When the monitoring service detects patterns of invalid messages
   - These signals can be configured based on monitoring thresholds and patterns

The Monitoring Service may be able to identify Interop Messages which positively won't be Valid, beyond the `CrossUnsafe`
simply stalling. A simple trigger from the metrics service would be to trigger AutoStop any time any Interop Message on the
chain in question (OP Mainnet for us) would resolve to `Invalid`.

3. Manual Operator Control
   - Direct API calls from operators to enable/disable the stop
   - Useful for proactive measures when operators observe chain instability
   - Can be used to prevent potential issues before they manifest in blocks

AutoStop is currently designed *not* to Auto-Start again. This is because we are currently protecting ourselves against
an ambiguous concern. Operators should investigate and give the all-clear before resuming Interop. In the future, we should expect
our comfort level to be high enough to support resuming service automatically.

### Partner Triggers
Because the Superchain is a collection of autonomous-yet-interdependent chains, there may be cause for *other* chain operators to
trigger AutoStop from the outside in a way that affects all partners.

To that end, we can expose the AutoStop Manual Controls through a key-protected endpoint which is shared privately to trusted partners,
and vise-versa, so that any actor can initiate AutoStop for all chains. *Without* this feature, chains may still individually protect themselves,
but this adds a layer of altruism. If no business relationship justifies this endpoint, or if we find it is not a productive feature,
we can remove access to the endpoint easily.


### Implementation

AutoStop is mostly a feature of the Supervisor, and to implement it we will need to add features to the Supervisor and surrounding components.

1. Supervisor to include AutoStop API
    - AutoStop is a simple boolean flag with an Admin API for getting and setting the flag AutoStop
    - When AutoStop is enabled, all `CheckAccessList` API calls return a custom error code indicating AutoStop
2. Supervisor to self-trigger AutoStop when a Replacement Block is issued
3. Supervisor to communicate AutoStop to Managed Nodes
    - A new Managed API should be created to indicate to Nodes that some outside signal is stopping Interop
    - If the Managed Node is the Sequencer, it should check if AutoStop is on, and prune all Interop Transactions during Block Building
    - Other Managed Nodes could ignore the AutoStop signal, because they do not control inclusion
4. Proxyd to communicate AutoStop to Customers
    - When Proxyd checks messages, it can return the AutoStop error back to customers.
    - I think Proxyd may already pass through errors
5. Interop-mon to call AutoStop for a set of Supervisors on certain Metrics discoveries
    - The Monitoring Service can take a list of Supervisor URLs (and likely JWTs) to call AutoStop on all of them when issues arise
    - This set *must* include all filtering Supervisors (the ones who serve Proxyd and mempools), plus the Supervisor
6. Simple Scripting for Checking and Setting AutoStop manually

With the primary source of AutoStop coming from Supervisor, all downstream components can deny messages in a coordinated way:
- New Interop Messages get gracefully declined at the gate
- Old Interop Messages get dropped by mempools as their status is re-checked
- Sequencer diverts any Interop Messages that would have been built into the block

And similarly, operators can return the system to service with scripted calls to the Supervisors, quickly unblocking all gates.

A complete guide on how to evaluate, respond to, and use AutoStop must also be written into a runbook which guides Operators.

## Future Scope
For the initial launch of Interop, we need a basic-yet-reliable tool, so we should avoid heavy sophistication, especially at high engineering cost.
However, this feature could be enhanced in the future in the following ways:
- Batcher Short-Publishing: if an invalid message is detected and we are already taking defensive measures, it may make sense to additionally
have the Batcher intentionally publish the batch containing the invalid message as soon as possible. If the last block of the batch is the
one containing the Invalid Message, then we minimize the amount of wasted block building is published. This also shortens the time to
Invalid Messages being detected, as it puts the content on the L1 as soon as possible. This feature is not direclty related to Auto *Stop*,
and could potentially be achieved by other means.
- AutoResume (mentioned already): once we are comfortable with the dangers of Invalid Messages, we could allow the system to resume automatically.
- Per-Chain Stopping: if the cause of instability is a specific chain, it may make sense for operators to only deny interop messages from *that* chain.
Supervisor would have to handle a more complex AutoStop concept, and the Metrics Service would have to have more conditions before calling the API.
- AutoStop as an AutoReorg: in some of these trigger cases, the Sequencer could not only become more protective when an invalid message is discovered,
but could also proactively reorg their local chain to begin building a replacement. This feature *requires* Batcher to be hooked to the Sequencer,
and has its own negative UX externalities (users are no longer transacting on the state they saw in the Unsafe Chain)

## Summary of Solution
- Create an Admin API of the Supervisor which creates graceful AutoStop errors for downstream filter consumers
- Have AutoStop triggered by Derivation Events, Monitoring Service, and Operators
- Have AutoStop affect Sequencer BlockBuilding behavior to not include Interop Messages

## Alternatives Considered
Most alternatives focused on *where* to put this enable/disable flag. We had at one point thought that only Proxyd would need to support this, but that is
ineffective for any messages which had just passed by. Similarly, only doing it in the mempool left pressure on Proxyd ingress. And implementing the same flag in multiple places
seemed complicated.

Similarly, alternatives focused on *when* AutoStop would trigger. Manual Operation only puts us at risk of high response times. The selected design takes three sources because they are
distinct, but all easy to implement. Metrics and Manual both use the API, while Derivation Trigger is internal to the Supervisor.

## Risks & Uncertainties
- The Interop Monitoring Service is unfinished in a branch. We need to make sure it works as expected and integrate it
- We also need to decide *exactly* the conditions for triggering AutoPause. This document suggests some obvious ones, but because it has significant effect on the network,
we should put a level of detail on these triggers which is not in scope for this document (though it could be, ask me to include it if you like)
- Even when we catch an Invalid Message and become protective, we are still encountering some amount of stall/reorg behavior. Not much to do about that here,
but a feature can't be reactive to invalid messages *and* prevent them.
- Other chain operators should protect themselves like we do, but we can't control them. Luckily, these protection measures are effective even if other chains don't follow them,
so while the superchain may suffer interop outages from poor operator practices, those chains which *do* protect themselves will not suffer general liveness issues.
- Testing AutoStop will require special Devnets which can pass through Invalid Messages to be built into blocks.