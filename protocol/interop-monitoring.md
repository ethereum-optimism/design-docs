# Interop Monitoring

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Mark Tyneway_                                     |
| Created at         | _2025-03-19_                                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_                 |
| Need Approval From | _Reviewer Name_                                    |
| Status             | _Draft_ |

## Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

This document is meant to align on a strategy for monitoring interop. Given assumptions in
the [cloud topology](https://github.com/ethereum-optimism/design-docs/pull/218), it is generally
not possible to guarantee that invalid `ExecutingMessage`s do not finalize without multiple
implementations of `op-supervisor`. With only a single implementation, a bug becomes consensus.
In the worst case, this can mint an infinite amount of ether. Given this risk, we need a monitoring,
alerting and runbook for handling invalid `ExecutingMessage`s being included in the chain.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

We implement a monitoring service that validates all of the `ExecutingMessage` logs
produce by the entire cluster and validates them against transaction access lists
and remote nodes. We use this service to alert oncall engineers as well as automatically
pausing the batcher/transaction ingress if an invalid `ExecutingMessage` is included.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

We want to be alterted when there is an invalid `ExecutingMessage`. We implement preventative
measures, but the downside risk is existential if an invalid `ExecutingMessage` finalizes,
so we need to have ways to detect and prevent that.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

A service is implemented that is able to observe all blocks and logs produced by the cluster.
It is responsible for doing two things:
- Guaranteeing that every executing message has a corresponding access list entry
- Double checking that every executing message is valid

### ExecutingMessages and Access Lists

The [access list](https://github.com/ethereum-optimism/design-docs/blob/9e919c5b173fe8fc89949b012f6f70a0bc3247f6/protocol/interop-access-list.md)
design guarantees the fact that all executing messages can be validated without the need to execute the transaction. Any calls to the `CrossL2Inbox`
that do not include the statically declared executing message in the access list will revert rather than needing to be dropped. This prevents
a denial of service attack where the MEV searcher can simply produce an invalid `ExecutingMessage` after their MEV attempt fails.

Given that the decided upon approach depends strictly on the current EVM resource pricing via storage slot cost introspection, we should have
monitoring to alert us if someone is able to trick the `CrossL2Inbox` into producing an `ExecutingMessage` when the access list entry is
not declared. We think this is impossible, but given this is such a critical security property, it is important to monitor.

### Double Checking Message Validity

#### Unsafe Blocks

We want to utilize the cloud architecture in [this doc](https://github.com/ethereum-optimism/design-docs/pull/218) to ensure that
no invalid `ExecutingMessage`s are ever included in a block. No matter what tradeoffs we make, it is impossible to guarantee there
will not be a contingent reorg because an unsafe head reorg can happen after all cross chain transaction validity checks passed.

We want to be able to detect when an invalid `ExecutingMessage` is included in an unsafe block and trigger an altert to the
oncall engineering team. Additionally, we should consider pausing the batcher automatically if an invalid `ExecutingMessage`
has been detected in an unsafe block and triggering an unsafe head reorg. It is preferable to not waste blobs and trigger
an unsafe head reorg by batch submitting the invalid block. The problem with this is that it is indistinguishable from
a malicious sequencer triggering a reorg to extract MEV. Either way an unsafe head reorg is going to happen, its just whether
or not its due to the data being posted and then resulting in a replacement deposits only block or if its manually done
by the sequencer offchain.

We may also want to consider a way to alert partners in the interop set ahead of time that an unsafe head reorg is coming
if an invalid `ExecutingMessage` is observed in an unsafe block. If they turn off their cross chain message ingress fast enough,
it could be possible that they can prevent a contingent reorg. The liveness of the chain can continue with no issues until the
remote chain goes through its unsafe head reorg, then it can open up its cross chain message ingress again.

#### Safe/Finalized Blocks

If an invalid `ExecutingMessage` ends up in a safe block, that means that a bug becomes consensus (without multiclient).
This is very bad. Our monitoring should be able to observe this. The worst case thing that can happen in this case is
an attacker uses a fast liquidity bridge like Across to quickly send funds out of the cluster after the finalization of
an invalid `ExecutingMessage` that mints a ton of ether out of thin air. If we detect an invalid `ExecutingMessage`
finalizing to be safe/finalized, we should pause all transaction ingress to the sequencer and effectively stop producing
blocks. This reduces the chances that the attacker is able to initiate a fast bridge out of the cluster. This strategy
is aligned with our philosophy of favoring safety over liveness.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

A new service needs to be implemented and operated in the cloud. This service
can be stateless, it mostly needs to do consistent network access to full nodes.
The cost of the full nodes is going to be the majority of the cost in operating
this service.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

This is meant to detect the single source of failure with the `op-supervisor`. Having client diversity
for monitoring would be a nice to have.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

No real alternatives considered.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

- If an invalid `ExecutingMessage` finalizes, it would be a very bad look to roll back the chain. It may be the best solution
- Need to observe the latency of validating the `ExecutingMessage`s to ensure that this is all feasible