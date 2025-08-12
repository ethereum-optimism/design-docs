# Nonceless Execution

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Alberto Cuesta Cañada                              |
| Created at         | 2025-08-12                                         |
| Initial Reviewers  | [ ] John Mardlin                                   |
|                    | [ ] Kelvin Fichter                                 |
|                    | [ ] Matt Solomon                                   |
| Need Approval From | _Reviewer Name_                                    |
| Status             | Draft                                              |

## Purpose

Let governance multisigs execute approved transactions without being blocked by a single global nonce. Allow independent transactions to run in any order while keeping the same signature thresholds and replay protection.

## Summary

We add a Safe-compatible module that enables unordered execution. Instead of one nonce that forces everything to run in sequence, each approved transaction has its own identifier. Once it has enough signatures, it can be executed in any order. Signature thresholds stay the same. Executions are public and cannot be replayed.

## Problem Statement + Context

Transaction ordering is a huge hassle that sucks up our time and causes delays and rework. When governance needs to execute multiple independent transactions, they get stuck waiting for each one to complete before the next can begin. This creates unnecessary bottlenecks where urgent fixes wait behind routine updates, and coordination overhead grows with every additional transaction in the queue.

## Proposed Solution

Introduce a Safe module that allows unordered execution of approved transactions. Transactions do not compete on a global nonce and each executes as soon as it is fully approved. Existing signature thresholds and signer sets continue to apply. Each transaction is uniquely identified and cannot be executed more than once, while authorized transactions and executions remain visible on-chain and traceable to their approvals.

## Scope

- Enable unordered execution for governance multisigs.
- Preserve existing thresholds, ownership, and incident response processes.

## Non-goals

This change does not involve changing thresholds or governance ownership models, adding new signing tools or UIs, or redesigning incident response.

## Resource Usage

On-chain costs are similar to current multisig execution with small module overhead. Operationally, this reduces coordination overhead and enables faster handling of urgent items.

## Alternatives Considered

The status quo of nonce ordering requires no implementation effort but negatively impacts operations and UX due to the coordination and planning required.

## Risks & Uncertainties

- Although this has very little impact on signing flows, it does significantly change transaction execution (and approval with nested safes). Hard to say how heavily it would impact on the superchain-ops repo.
- Is this something that can be adopted piecemeal, ie. could we manage with safes that do and do not use this module? I think we could likely autodetect when a Safe supports nonceless execution, but more branches is still harder to maintain.
- Unexpected effects due to unordered changes.

## References

- [Unordered Execution of Safe Transactions](https://www.notion.so/oplabs/Unordered-execution-of-Safe-transactions-20ef153ee1628054ade1e7e8beeadfef#20ef153ee162800689d1cd1268c85f91)
- Nonceless execution module (Safe-compatible): [UnorderedExecutionModule.sol](https://github.com/ethereum-optimism/optimism/blob/28f44ab50b01fb59f875c7b85d216cdce713b6dd/packages/contracts-bedrock/src/safe/UnorderedExecutionModule.sol#L1)
- [Nonceless Execution Overview (Google Slides)](https://docs.google.com/presentation/d/1utbGigIbMRA7JGcKZ9ZcUgMCdIPpLDJSwWmxNF9hBJM/edit?slide=id.g3734216eca8_4_32#slide=id.g3734216eca8_4_32) 