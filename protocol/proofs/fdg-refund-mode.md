# FaultDisputeGame Refund Mode

## Context

Dealing with `FaultDisputeGame` bonds appropriately is an important part of the current OP Stack
incident response process. `FaultDisputeGame` stores player bonds inside of the `DelayedWETH`
contract which imposes a 7 day delay on bond withdrawals. During this delay period, the 2/2 Safe
managed jointly by the Security Council and the Optimism Foundation can hold bonds from specific
games or from all games at the same time. Held bonds are transferred to this 2/2 Safe where they
can be manually disbursed to the correct recipients.

## Problem Statement

Although `DelayedWETH` is an effective tool at preventing a buggy or malicious `FaultDisputeGame`
from distributing bonds to a potential attacker, the actual process of returning those bonds back
to the correct players is underspecified. Underspecified proceses can introduce significant stress
to already stressful situations where executive decisions around fallbacks must be made quickly and
confidently. We therefore want to introduce alternative low-stress options for bond recovery.

## Proposed Solution

We propose modifying the `FaultDisputeGame` to enter a "refund mode" if the game is blacklisted.
When the `FaultDisputeGame` is in this refund mode it will distribute bonds back to the users who
originally deposited those bonds.

### Preventing Complex States

`FaultDisputeGame` currently allows users to begin withdrawing bonds before the game has fully
resolved. It would likely be necessary to modify the `FaultDisputeGame` to block withdrawals until
full resolution so that we can more easily reason about the security properties of this change. It
may otherwise be possible to be caught in strange states.

For instance, `FaultDisputeGame` instances are usually expected to be blacklisted only after they
resolve incorrectly, so it may be possible for some users to begin withdrawing *before* a game is
blacklisted, then the game becomes blacklisted and those same users can withdraw their original
bonds. We can avoid this sort of double-counting and generally simplify the security model of the
contract by preventing bond withdrawals until the game has resolved.

We also likely need to prevent `FaultDisputeGame` contracts from entering refund mode if any
withdrawals have already started to avoid dealing with complex state. If any user is able to
withdraw before refund mode is activated then we assume that all withdrawals could've been
completed and there's no reason to activate refund mode for those contracts.

### Enabling Blanket Refunds

Existing blacklisting functionality targets specific `FaultDisputeGame` instances and cannot easily
handle cases where many games need to be blacklisted. If we were to adopt DD#76 alongside
[DD#77](https://github.com/ethereum-optimism/design-docs/pull/77) then we could use the proposed
`gameCreationValidFrom` timestamp to act as an implicit blacklist of all games created before this
timestamp. The `AnchorStateRegistry.isValidGame(...)` function would return `false` if the game is
explicitly blacklisted OR if the game was created before this validity timestamp. All games created
before `gameCreationValidFrom` would then enter refund mode.

### FaultDisputeGame Bond Unlock Modification

The `FaultDisputeGame` contract currently calls `DelayedWETH.unlock(...)` when it credits a user
with a bond. However, `DelayedWETH.unlock(...)` is responsible for (1) starting the 7 day
withdrawal period and (2) restricting how `FaultDisputeGame` can withdraw out of `DelayedWETH`. We
would need to move this unlock to only be possible after the `FaultDisputeGame` has resolved.
Additionally, this unlock would need to be re-triggered if refund mode is activated.

We need to make sure that there is never a case where a `FaultDisputeGame` can distribute bonds
under two different distribution modes. If the `FaultDisputeGame` has already started to distribute
bonds under the default distribution mode, it must not be able to switch into the refund mode. We
can do this with some sort of flag that tracks if the game has already started distributing bonds
normally.

## Risks & Uncertainties

### Security

`FaultDisputeGame` is, obviously, an important part of the security of the current OP Stack. Any
modifications to the `FaultDisputeGame` contract will likely need to be audited.

### AnchorStateRegistry modifications

[Design Doc #76](https://github.com/ethereum-optimism/design-docs/pull/76) proposes to shift the
state validation logic currently found within the `OptimismPortal` contract into the
`AnchorStateRegistry`. Adoption of DD#76 would significantly simplify the specification for this
proposal. One potential risk here is that this proposal effectively becomes a dependency of
DD#76 as it would likely be too messy if `FaultDisputeGame` needed to reference `OptimismPortal`.

### DelayedWETH Interactions

We believe that the changes described here would not impact the continued ability to use
`DelayedWETH` to act as a more serious fallback in the case that refund mode would not properly
refund users for some reason. We should double check to confirm that we would not see any
unexpected interactions between the two systems.

### Challenger and Dispute Monitor Changes

Changes that need to be made to the `FaultDisputeGame` bond unlock logic may require some small
modifications to `op-challenger` and `op-dispute-mon`. We may be able to avoid changes to
`op-challenger` if we move the `unlock` call into `claimCredit` and simply fail if the unlock
has already been made. Need to confirm here that `op-challenger` will just retry this claim
function over and over, but if that's the case then this works just fine.

`op-dispute-mon` would need to be modified slightly to account for the fact that (1) the credits
inside of `DelayedWETH` will only match *after* the game has resolved and (2) the unlocks may
become mismatched if refund mode is activated. Both situations can be managed but do require some
changes to `op-dispute-mon`.
