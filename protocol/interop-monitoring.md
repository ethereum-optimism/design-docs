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
it is generally not possible to guarantee that invalid `ExecutingMessage`s do not finalize without multiple
implementations of `op-supervisor`. With only a single implementation, a bug becomes consensus.
In the worst case, this can mint an infinite amount of ether. Given this risk, we need a monitoring,
alerting and runbook for handling invalid `ExecutingMessage`s being included in the chain.

We want to be alterted when there is an invalid `ExecutingMessage`. We are implementing preventative
measures, but the downside risk is existential if an invalid `ExecutingMessage` finalizes,
so we need to have ways to detect and prevent that.

## Proposed Solution

We should implement a monitoring service that validates all of the `ExecutingMessage` logs
produce by the entire cluster and validates them against transaction access lists
and remote nodes. We use this service to alert oncall engineers as well as potentially automatically
pausing the batcher/transaction ingress if an invalid `ExecutingMessage` is included.

This "Cross Message Monitor" should have the following features:

### Monitoring Strategies like `dispute-mon`

Dispute Monitor is a service already implemented and deployed for tracking Fault Proof Disputes.
Rather than just be a simple alert when games are invalid, it serves up various statistics that an operator
can refer to in order to determine network health:
- How many games are being monitored, how many of each status
- How many Incorrect Forecasts or Incorrect Results
- Warning and Error Logs from the Monitor

Cross Message Monitor can crib directly from these statistics, but focused on Interop:
- How many Executing Messages emitted by the CrossL2Inbox per block per chain
- How many `Executing Message`s Messages point at each Chain in the Superchain
- How many `Executing Message`s are known valid, per safety level
- How many `Executing Message`s are known invalid, per safety level
- How many `Executing Message`s are not yet known valid/invalid, per safety level
- How many `Executing Message`s *changed validity* over time (indicating remote reorg)
- How many `Executing Message`s were resolved via Block Replacement

Almost all `Executing Message` Metrics emitted by the Cross Message Monitor should have dimensions:
- What chain the `Executing Message` in question is on
- What chain the `Executing Message` is referring to (the chain of the initiating message)
- Timestamp of Block

### Long Term Monitoring of `Executing Message`s

Executing Messages can change validity over the course of the Unsafe Chain,
data is not allways sufficiently available to validate `Executing Message`s, and transitive `Executing Message`s can
cause cascades of Valid/Invalid messages.

Therefore, it is insufficent to check a message just once. Instead, every Executing Message
detected by the Cross Message Monitor will be considered an ongoing process, like games are
for the Dispute Monitor. From the time the `Executing Message` is discovered, until the `Executing Message` is included by a
Cross Safe block height which is now L1 finalized, the `Executing Message` should be repeatedly re-checked.

This means that when the status of the `Executing Message` flips, special alerts can be emitted to indicate
a remote reorg has likely occured. Or, when a single invalid message creates a cascade of
invalidation, each `Executing Message` can resolve individually.

### Access List Confirmation

The [access list](https://github.com/ethereum-optimism/design-docs/blob/9e919c5b173fe8fc89949b012f6f70a0bc3247f6/protocol/interop-access-list.md)
design guarantees the fact that all executing messages can be validated without the need to execute the transaction. Any calls to the `CrossL2Inbox`
that do not include the statically declared executing message in the access list will revert rather than needing to be dropped. This prevents
a denial of service attack where the MEV searcher can simply produce an invalid `ExecutingMessage` after their MEV attempt fails.

Given that the decided upon approach depends strictly on the current EVM resource pricing via storage slot cost introspection, we should have
monitoring to alert us if someone is able to trick the `CrossL2Inbox` into producing an `ExecutingMessage` when the access list entry is
not declared. We think this is impossible, but given this is such a critical security property, it is important to monitor.

Each message can be checked for this once, when it is detected and added to the monitoring set.

### Alert Behaviors

Though it will need evaluation over time, we already know the sorts of operator responses we want when certain situations are detected
by the monitor.

#### Unsafe Blocks

We want to be able to detect when an invalid `ExecutingMessage` is included in an unsafe block and trigger an altert to the
oncall engineering team. It is preferable to not waste blobs and trigger an unsafe head reorg by batch submitting the invalid block as soon as possible,
therefore the operator may want to accelerate batch submission when this alert arrives. Unless nodes on the network are 
able to accept an Unsafe->Unsafe block replacement (and they are not), the Sequencer's only path forward is to see the 
invalid block commited to L1, at which point it will be replaced. Doing this faster will minimize reorg sizes.

We may also want to consider a way to alert partners in the interop set ahead of time that an unsafe head reorg is coming
if an invalid `ExecutingMessage` is observed in an unsafe block. If they turn off their cross chain message ingress fast enough,
it could be possible that they can prevent a contingent reorg. The liveness of the chain can continue with no issues until the
remote chain goes through its unsafe head reorg, then it can open up its cross chain message ingress again.

Finally, when Invalid Messages occur, it is prudent to shut off additional Executing Messages. Admin APIs should be established which:
- Shut off Executing Message Ingress at `proxyd`
- Force remove Executing Messages from block builder mempools.
These triggers should occur automatically when an invalid `ExecutingMessage` is discovered at the Unsafe Block stage, in order to reduce cascades.

#### Safe/Finalized Blocks

If an invalid `ExecutingMessage` ends up in a safe block, it is an expectation of the Protocol that the block is Invalid,
and must be replaced with a Deposit Only Block. This situation should page the operator to monitor the situation, and every
individual invalid `Executing Message` in a Safe Block should be very easy to see and monitor individually. The operator is monitoring
to ensure a Block Replacement occurs and the invalid messges are no longer known to the chain.

If Cross-Validation should promote the block to Cross-Safe, this is an all-hands-on-deck consensus bug, which would naturally
have its own alerts associated in addition to the prior expectation of an operator monitoring the situation.

### Resource Usage

This new service will need minimal CPU/Disk and can be stateless. It will need a connection to one of each Node
for the Superchain it is monitoring.

## Summary of Solution

Create `xmsg-mon` in the image of `dispute-mon` to track all in-flight Executing Messages for a Superchain, for their entire
Unsafe -> Safe -> Finalized lifecycle. Create Alerting against it which pages operators when Invalid Messages advance into blocks.

Furthermore, Admin APIs should be established to shut off `proxyd` and `mempool` acceptance of Executing Messages, to swiftly respond
when the Monitoring Service detects invalid messages in blocks.

## Alternatives Considered

No real alternatives considered. Monitoring should happen as a matter of course when deploying new services.

Having additional Cross-Validation software besides Supervisor would lessen the criticality of this software.

## Risks & Uncertainties

- The Monitoring Service may be insufficent, and we may not catch what we need to. Real experience will inform updates to this service.
- The speed of the Monitor may be insufficent for operators to take meaningful action