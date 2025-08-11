# Nonceless Execution

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Author Name_                                      |
| Created at         | _YYYY-MM-DD_                                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_                 |
| Need Approval From | _Reviewer Name_                                    |
| Status             | _Draft / In Review / Implementing transactions / Final_ |

## Purpose

Let governance multisigs execute approved transactions without being blocked by a single global nonce. Allow independent transactions to run in any order while keeping the same signature thresholds and replay protection.

## Summary

We add a Safe-compatible module that enables unordered execution. Instead of one nonce that forces everything to run in sequence, each approved transaction has its own identifier. Once it has enough signatures, it can be executed in any order. Signature thresholds stay the same. Executions are public and cannot be replayed.

## Problem Statement + Context

- Nonce contention: a single stuck transaction can block unrelated work.
- Operational latency: urgent changes wait behind unrelated items in the queue.
- Throughput: we need to run unrelated transactions at the same time without extra coordination.

## Proposed Solution

Introduce a Safe module that allows unordered execution of approved transactions:
- Unordered execution: transactions do not compete on a global nonce; each executes as soon as it is fully approved.
- Threshold-preserving: existing signature thresholds and signer sets continue to apply.
- Replay safety: each transaction is uniquely identified and cannot be executed more than once.
- Auditability: authorized transactions and executions are visible on-chain and traceable to their approvals.

## Scope

- Enable unordered execution for governance multisigs.
- Preserve existing thresholds, ownership, and incident response processes.

## Non-goals

- Changing thresholds or governance ownership models.
- Adding new signing tools or UIs.
- Redesigning incident response.

## Success Criteria

- Liveness: independent approved transactions can execute without waiting on others.
- Safety: no increase in unauthorized execution risk; replay attempts are rejected.
- Usability: no extra steps for signers; current approval workflows still work.
- Observability: clear on-chain signals for monitoring and post-mortems.

## Resource Usage

- On-chain: similar cost to current multisig execution, with small module overhead.
- Operational: less coordination overhead; faster handling of urgent items.

## Alternatives Considered

- Status quo (nonce ordering): no implementation effort, but it negatively impacts operations and UX due to the coordination and planning required.

## Risks & Uncertainties

- Unexpected effects due to unordered changes.
  - Mitigation: review, simulation, change windows, and clear runbooks.

## References

- Nonceless execution module (Safe-compatible): [UnorderedExecutionModule.sol](https://github.com/ethereum-optimism/optimism/blob/28f44ab50b01fb59f875c7b85d216cdce713b6dd/packages/contracts-bedrock/src/safe/UnorderedExecutionModule.sol#L1)
- Slides overview: [Nonceless Execution (Google Slides)](https://docs.google.com/presentation/d/1utbGigIbMRA7JGcKZ9ZcUgMCdIPpLDJSwWmxNF9hBJM/edit?slide=id.g3734216eca8_4_32#slide=id.g3734216eca8_4_32) 