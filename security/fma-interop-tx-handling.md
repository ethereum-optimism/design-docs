# Interop Transaction Handling: Failure Modes and Recovery Path Analysis

| | |
|--------|--------------|
| Author | Axel Kingsley |
| Created at | 2025-03-31 |
| Needs Approval From | |
| Other Reviewers |   |
| Status | Draft |

## Introduction

This document covers new considerations for Chain Operators in Interop-enabled spaces when processing transactions.

## Context and Problem

In an OP Stack chain, block builders (Sequencers) build blocks from user-submitted transactions. Most
Chain Operators arrange their infrastructure defensively so that RPC requests aren't handled direclty by the
Supervisor, instead building the mempool over P2P.

In an Interop-Enabled context, Executing Messages (Interop Transactions) hold special meaning within the
protocol. For every Executing Message in a block, there must be a matching "Initiating Message" (plain log event)
which matches the specified index and content hash.

Including an Executing Message which *does not* match to an Initiating Message is considered to be invalid to the protocol.
The result is that the *entire block* which contains this Invalid Message is replaced, producing a reorg on the chain
at this height.

Because the consequenquence for an Invalid Message is so high, Chain Operators are highly incentivized to check Executing
Messages before they are added to a block. *However*, being excessive with these checks can cause interruption to the
chain's regular forward-progress. There must be a balance taken in checking messages.

### Two Extremes

To understand the purpose of these decisions, lets consider the extreme validity-checking policies we could adopt:

**Check Every Message Exhaustively**
- In this model, every Executing Message is checked constantly, at a maximum rate, and each message is checked as close to block-building as possible.
- The compute cost to check every message during block building adds time to *every* transaction, blowing out our ability
to build a block in under 2s.
- The validity of the included Executing Messages would be *as correct* as possible. However! Even *after* the block is built, the data being relied upon (cross-unsafe data) could change on the Initiating Chain if they suffer a reorg.
So while this policy is *most correct*, it is not *totally correct*

**Never Check Any Messages**
- In this model, we optimize only for not taking any additional compute tasks, instead just trusting every message.
- Naturally, there is no impact into block building or any other process, BUT...
- Blocks would easily become invalid, because an attacker could submit Invalid Messages, even just one per 2s, and prevent the Sequencer from ever building a valid block.

So, no matter what solution we pick, we deal with *some* amount of uncertainty and take *some* amount of additional compute load.

## The Solution Design

The [Interop Topology and Tx Flow for Interop Chains Design Doc](https://github.com/ethereum-optimism/design-docs/pull/218)
describes the solution design we plan on going with:

- All Executing Message are checked once at `proxyd` ingress.
- All Executing Message are checked once at Node Mempool ingress (not counting Sequencer).
- All Executing Message in Node Mempools are Batched at checked on a regular interval.
- If an Executing Message is ever Invalid, it is discarded and not retried.
- *No* Checks are done at Block Building time.

This FMA describes the potential negative consequences of this design. We have selected this design because it maximizes
the opportunities for Invalid Messages to be caught and discarded, while also leaving the block building hot-path from
having to take on new compute.

# FMs:

## FM1: Checks Fail to Catch an Invalid Executing Message
- Description
    - Due to a bug in either the Supervisor, or the Callers, an Executing Message
    was allowed into block building.
    - When this happens, the block which is built is invalid.
    - The Sequencer for the network can choose to build from the Replacement of the Invalid block,
    or from the Parent, if this is still the Unsafe Chain. If it continues to build from the Invalid
    block itself, all those appended blocks will be dropped.
- Risk Assessment
    - Has no effect on our ability to process transactions. Supervisor effects are described in
    the Supervisor FMA.
    - Negative UX and Customer Perception from building invalid block content.

## FM2: Checks Discard Valid Message
- Description
    - Due to a bug in either the Supervisor, or the Callers, some or all Executing Messages
    aren't being included in blocks.
    - When this happens, there is nothing invalid being produced by block builders, but no Interop
    Messages are being included.
- Risk Assessment
    - More Negative UX and Custoemr Perception if Interop Messages aren't making it into the chain.
    - Failed transactions would cause customers to redrive transactions, potentially overwhelming
    infrastructure capacity.

## FM3a: Transaction Volume causes DOS Failures of Proxyd
- Description
    - Due to the new validation requirements on `proxyd` to check Interop Messages, an influx of
    Interop Messages may arrive, causing `proxyd` to become overwhelmed with work.
    - When `proxyd` becomes overwhelmed, it may reject customer requests or crash outright, affecting
    liveness for Tx Inclusion and RPC.
- Risk Assessment
    - Low Impact ; Medium Likelihood
    - If a `proxyd` instance should go down, we should be able to replace it quickly, as it is a stateless
    service.
    - When `proxyd` goes down, it takes outstanding requests with it, meaning some requests fail to be handled,
    but excess load is shed.
- Mitigations
    - `proxyd` could feature a pressure-relief setting, where if too much time is spent waiting on the Supervisor,
    no additional Interop Messages will be accepted through this gateway.
    - We should deploy at least one "Utility Supervisor" to respond to Checks from `proxyd` instances.
    The size and quantitiy of the Supervisor(s) could be scaled if needed. (Note: A Supervisor also requires Nodes
    of each network to function)
## FM3b: Transaction Volume causes DOS Failures of Supervisor
- Description
    - In any section of infrastructure, calls to the Supervisor to Check a given Interop Message might overwhelm the
    Supervisor.
    - If this happens, the Supervisor may become slow to respond, slow to perform its other sync duties, or may crash outright.
    - When a Supervisor crashes, any connected Nodes can't keep their Cross-Heads up to date, and Managed Nodes won't get L1 updates either.
- Risk Assessment
    - Medium Impact ; Low Likelihood
    - The Supervisor is designed to respond to Check requests. Even though it hasn't been load tested in realistic settings, there is very little computational overhead when responding to an RPC request.
    - Supervisors can be scaled and replicated to serve high-need sections of the infrastructure. Supervisors
    sync identically (assuming a matching L1), so two of them should be able to share traffic.
- Mitigations
    - When the Supervisor is down, any block builder or mempool filter *should* treat unavailability as
    a negative - to protect correctness, when the Supervisor goes down, don't include any Interop Messages.
## FM3c: Transaction Volume causes DOS Failures of Node
- Description
    - Transactions make it past `proxyd` and arrive in a Sentry Node or the Sequencer.
    - Due to the high volume of Interop Messages, the work to check Interop Messages causes
    a delay in the handling of other work, or causes the Node to crash outright.
- Risk Assessment
    - Medium/Low Impact ; Low Likelihood
    - It's not good if a Sequencer goes down, but the average node can crash without issue.
    - Conductor Sets keep block production healthy even when a Sequencer goes down.
- Mitigations
    - Callers should use Batched RPC requests to the Supervisor when they are regularly validating groups
    of Transactions. This minimizes the amount of network latency experienced, allowing other work to get done.
    - Mempool transactions which fail checks should be dropped and not retried. This prevents malicious transactions
    from using more than one check against the Supervisor.