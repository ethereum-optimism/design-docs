# Expanded DelayedWETH Capabilities

## Context

The `DelayedWETH` contract acts as a holding contract for the bonded ETH submitted by any
participant in a `DisputeGame` contract. As of the Granite upgrade, each implementation of the
`DisputeGame` uses its own `DelayedWETH` contract. Each `DelayedWETH` contract is `Ownable` and is
subject to a number of safety net actions by the owner address (holding balances from specific
addresses or removing all ETH held in the `DelayedWETH` contract). `DelayedWETH` is additionally
subject to the Superchain-wide pause mechanism within the `SuperchainConfig`. Bonds cannot be
retrieved by game participants if the Superchain pause is active.

## Problem Statement

Existing security mechanisms within the `DelayedWETH` contract are inconvenient and slow down the
incident response process. Specifically:

- The `DelayedWETH.hold(address target)` function creates an approval from the `target` address to
  the `owner` address. Since the `target` can simply remove this approval, `hold` MUST be triggered
  alongside a `transferFrom` in the same transaction. This adds unnecessary complexity to something
  that could simply be a single transaction.
- The various security mechanisms inside of `DelayedWETH` can only be triggered by the System Owner
  which has a slow SLA. The Deputy Guardian can act to trigger the Superchain-wide pause, but this
  is highly disruptive to the entire Superchain ecosystem.

Additionally, it should be noted that all ETH inside a `DelayedWETH` contract is pooled together
regardless of which games are using the contract. We currently mitigate this by using one
`DelayedWETH` contract per game type, but this expands the number of contracts that the System
Owner and Guardian roles must manage by at least 2 per additional chain. However, the modifications
required to appropriately address this issue are sufficiently extensive that they are considered
out of scope for this proposal.

## Proposed Solution

Here we propose some modifications to `DelayedWETH` that help to resolve the two noted limitations:

1. We propose that `DelayedWETH.hold` executes a `transferFrom` automatically to reduce the number
  of required transactions from 2 to 1 when holding ETH from a single dispute game. `hold` will
  specifically transfer to the `owner` address.
2. We propose a new `pause` function on the `DelayedWETH` contract itself that pauses *just* the
  `DelayedWETH` contract and supplements the Superchain-wide pause functionality. This `pause`
  function would be executable by the Guardian role and could therefore be triggered quickly in
  case the System Owner is unable to execute another security action in time (e.g., holding bonds
  from a particular dispute game).

### Updated Pause Behavior

We propose slightly modifying the behavior of the `pause` functionality for the `DelayedWETH`
contract as follows:

- As previously stated, the `Guardian` role will also be able to trigger the `pause` functionality
  for a particular `DelayedWETH` contract. The `DelayedWETH` contract will be in a paused state if
  the `Guardian` triggers the pause OR if the Superchain-wide pause is triggered.
- When the `pause` is active, users will not be able to trigger the `withdraw` function (same as
  before) EXCEPT the `owner` will still be able to trigger `withdraw` (may be useful in certain
  situations).
- When the `pause` is active, users other than the `owner` will not be able to transfer any tokens
  within the contract.

## Alternatives Considered

### Wider Rewrite

We considered the alternative to rewrite the `DelayedWETH` contract more broadly to be able to
easily manage multiple game types across multiple chains within a single `DelayedWETH` instance.
However, this would require significant modifications to the `DelayedWETH` contract and would act
as an iterative improvement to the status quo. We still believe that a larger rewrite should be
considered for the near future.

## Risks & Uncertainties

### Security Model

We see little risk with the modification to the `hold` function. Introducing a new `pause` function
slightly changes the existing security model (though not significantly). The Guardian role can
already pause the `DelayedWETH` contract by triggering the Superchain-wide pause. Allowing this
role to pause the `DelayedWETH` contract without pausing the rest of the system should not open up
capabilities that the Guardian does not already have. Comments on this subject are welcome.
