# SuperDisputeGame Migration: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Author Name_                                      |
| Created at         | 2026-02-02                                         |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_                 |
| Need Approval From | _Reviewer Name_                                    |
| Status             | Draft                                              |

## Purpose

<!-- This section is also sometimes called "Motivations" or "Goals". -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

## Summary

Migrate existing single-chain dispute games to SuperDisputeGame as a prerequisite for interop. This unifies all chains onto one game type, enabling future interop sets to share a SuperDisputeGame. Changes must be backward compatible with OptimismPortal2 to avoid invalidating existing withdrawals.

## Problem Statement + Context

- **Current state**: Each chain has its own dispute game (FaultDisputeGame/PermissionedDisputeGame)
- **Interop requirement**: Chains need to share dispute infrastructure
- **Step 1**: Migrate individual chains to SuperDisputeGame (this doc)
- **Step 2 (future)**: Interop sets share a single SuperDisputeGame

### Key Constraints

- **Backward compatibility**: OptimismPortal2 must continue working with in-flight withdrawals
- **No withdrawal invalidation**: Existing proven withdrawals must remain valid
- Migration path must be safe for live chains

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

## Failure Mode Analysis

See [FMA: SuperDisputeGame Migration](../../security/fma-super-dispute-game-migration.md).

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
