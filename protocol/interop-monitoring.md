# Interop Monitoring Service

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Mark Tyneway, Axel Kingsley_                                     |
| Created at         | _2025-03-19_                                       |

## Purpose

This document is meant to align on a strategy for monitoring interop and propose a
Monitoring Service for Executing Messages.

## Summary + Problem Statement + Context

Given assumptions in the [cloud topology](https://github.com/ethereum-optimism/design-docs/pull/218),
it is generally not possible to guarantee that invalid `Executing Message`s do not finalize without multiple
implementations of `op-supervisor`. With only a single implementation, a bug becomes consensus.
In the worst case, this can mint an infinite amount of ether. Given this risk, we need to have monitoring,
alerting, and a runbook for handling invalid `Executing Message`s being included in the chain.

We want to be alterted when there is an invalid `Executing Message`. We are implementing preventative
measures, but the downside risk is existential if an invalid `Executing Message` finalizes,
so we need to have ways to detect and prevent that.

## Proposed Solution

We should implement a monitoring service that validates all of the `Executing Message` logs
produced by the entire Superchain and validates them against transaction access lists
and remote nodes. We use this service to alert oncall engineers as well as potentially automatically
pausing the batcher/transaction ingress if an invalid `Executing Message` is included.

This "Executing Message Monitor" should have the following features:

### Monitoring Strategies like `dispute-mon`

Dispute Monitor is a service already implemented and deployed for tracking Fault Proof Disputes.
Rather than just be a simple alert when games are invalid, it serves up various statistics that an operator
can refer to in order to determine network health:
- How many games are being monitored, how many of each status
- How many Incorrect Forecasts or Incorrect Results
- Warning and Error Logs from the Monitor

Executing Message Monitor can crib directly from these statistics, but focused on Interop:
- How many `Executing Message`s are emitted by the `CrossL2Inbox` per block per chain
- How many `Executing Message`s Messages point at each Chain in the dependency set
- How many `Executing Message`s are known valid
- How many `Executing Message`s are known invalid
- How many `Executing Message`s are not yet known valid/invalid
- How many `Executing Message`s *changed validity* over time (indicating remote reorg)

By tracking these metrics individually, we can see at a glance the state of Cross-Validation, and identify underlying issues quickly.
For example, if the Executing Messages on a given chain start showing up invalid, it may indicate a failure of Tx filtering.
Or, if the *Initiating Messages* for a chain show a pattern of invalidity, it may indicate that Initiating chain is equivocating or reorging.

In particular, a change between Valid and Invalid status is especially noteworthy, as it demonstrate a high likelihood of reorg.

Because these metrics are dimensioned across both the Executing and Initiating side, we can tell whether the issue lies with the producer,
or the consumer.

Almost all `Executing Message` metrics emitted by the Executing Message Monitor should have dimensions:
- What chain the `Executing Message` in question is on
- What chain the `Executing Message` is referring to (the chain of the initiating message)
- Timestamp of Block

Additionally, we should alert when either the Monitor itself, or the underlying Node is down, to let operators know
when we are flying blind.

### Long Term Monitoring of `Executing Message`s

Executing Messages can change validity over the course of the Unsafe Chain,
data is not allways sufficiently available to validate `Executing Message`s, and transitive `Executing Message`s can
cause cascades of Valid/Invalid messages.

Therefore, it is insufficent to check a message just once. Instead, every Executing Message
detected by the Executing Message Monitor will be considered an ongoing process, like games are
for the Dispute Monitor. From the time the `Executing Message` is discovered, until the `Executing Message` is included by a
Cross-Safe block height which is now L1 finalized, the `Executing Message` should be repeatedly re-checked.

This means that when the status of the `Executing Message` flips, special alerts can be emitted to indicate
a remote reorg has likely occured. Or, when a single invalid message creates a cascade of
invalidation, each `Executing Message` can resolve individually.

### Access List Confirmation

The [access list](https://github.com/ethereum-optimism/design-docs/blob/9e919c5b173fe8fc89949b012f6f70a0bc3247f6/protocol/interop-access-list.md)
design guarantees the fact that all executing messages can be validated without the need to execute the transaction. Any calls to the `CrossL2Inbox`
that do not include the statically declared executing message in the access list will revert rather than needing to be dropped. This prevents
failing Interop Transactions from putting unpaid load onto the block builder.

Given that the decided upon approach depends strictly on the current EVM resource pricing via storage slot cost introspection, we should have
monitoring to alert us if someone is able to trick the `CrossL2Inbox` into producing an `Executing Message` when the access list entry is
not declared or differs from the Executing Message. We think this is impossible, but given this is such a critical security property, it is important to monitor.

Each message can be checked for this once, when it is detected and added to the monitoring set.

### Alert Behaviors

Though it will need evaluation over time, we already know the sorts of operator responses we want when certain situations are detected
by the monitor.

[**Note: this section is better detailed through the Interop: AutoStop design**](https://github.com/ethereum-optimism/design-docs/pull/287)

We want to be able to detect when an invalid `Executing Message` is included in an unsafe block and trigger an altert to the
oncall engineering team. It is preferable to not waste blobs and trigger an unsafe head reorg by batch submitting the invalid block as soon as possible,
therefore the operator may want to accelerate batch submission when this alert arrives. Unless nodes on the network are 
able to accept an Unsafe->Unsafe block replacement (and they are not), the Sequencer's only path forward is to see the 
invalid block commited to L1, at which point it will be replaced. Doing this faster will minimize reorg sizes.

We may also want to consider a way to alert partners in the interop set ahead of time that an unsafe head reorg is coming
if an invalid `Executing Message` is observed in an unsafe block. If they turn off their cross chain message ingress fast enough,
it could be possible that they can prevent a contingent reorg. The liveness of the chain can continue with no issues until the
remote chain goes through its unsafe head reorg, then it can open up its cross chain message ingress again.

Finally, when Invalid Messages occur, it is prudent to shut off additional Executing Messages. Admin APIs should be established which:
- Shut off Executing Message Ingress at `proxyd`
- Force remove Executing Messages from block builder mempools.
These triggers should occur automatically when an invalid `Executing Message` is discovered at the Unsafe Block stage, in order to reduce cascades.

If an invalid `Executing Message` ends up in a safe block, it is an expectation of the Protocol that the block is Invalid,
and must be replaced with a Deposit Only Block. This situation should page the operator to monitor the situation, and every
individual invalid `Executing Message` in a Safe Block should be very easy to see and monitor individually. The operator is monitoring
to ensure a Block Replacement occurs and the invalid messages are no longer part of the canonical chain.

If Cross-Validation should promote the block to Cross-Safe, this is an all-hands-on-deck consensus bug, which would naturally
have its own alerts associated in addition to the prior expectation of an operator monitoring the situation.

#### Clear Logs
When issues would arise that would generate an alert, the Monitor should also be printing clearly actionable logs which can be checked.
This would take the form of individual Invalid messages, or individual Invalid->Valid state transitions. Then operators can proceed to tirage
with high precision data.

### Resource Usage

This new service will need minimal CPU/Disk and can be stateless. It will need a connection to one of each Node
for the Superchain it is monitoring.

The service may use significant memory to store the ongoing statuses of potentially many Executing Messages across the chains
through their life-cycle.

### Availability and Reliability

This service must be able to detect *all* interop messages during their lifecycle. To that end, the service must be able to
backfill blocks on startup, so that temporary outages do not create blind spots in monitoring.

Only one monitor needs to be running if the backfill system works appropriately. Otherwise, a secondary backup monitor
may be advisable to keep gaps from forming.

## Monitoring Expiry

This service will need a way to prune old Executing Messages from being monitored once the lifecycle is over. To do that,
the monitoring service should pay attention to the *Finalized L2 Heads* of each chain, and stop monitoring Executing Messages
which were created prior to that finalized head.

## Summary of Solution

Create `xmsg-mon` in the image of `dispute-mon` to track all in-flight Executing Messages for a Superchain, for their entire
Unsafe -> Safe -> Finalized lifecycle. Create Alerting against it which pages operators when Invalid Messages advance into blocks.

Furthermore, Admin APIs should be established to shut off `proxyd` and `mempool` acceptance of Executing Messages, to swiftly respond
when the Monitoring Service detects invalid messages in blocks. (See: Interop AutoStop)

## Alternatives Considered

No real alternatives considered. Monitoring should happen as a matter of course when deploying new services.

Having additional Cross-Validation software besides Supervisor would lessen the criticality of this software.

## Risks & Uncertainties

- The Monitoring Service may be insufficent, and we may not catch what we need to. Real experience will inform updates to this service.
- The Monitoring Service may cause a lot of RPC traffic and generate a lot of data, putting strain on the infrastructure.
- The speed of the Monitoring Service may be insufficent for operators to take meaningful action