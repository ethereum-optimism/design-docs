# Limit PermissionedDisputeGame Play to Challenger

## Context

The `PermissionedDisputeGame` is a version of the `FaultDisputeGame` that limits the parties that
are allowed to execute moves within the game. The `PermissionedDisputeGame` is designed to act as
either a fallback for the `FaultDisputeGame` or an independent game type if a chain is not using
the permissionless `FaultDisputeGame` model. The `PermissionedDisputeGame` is intended to mirror
the behavior of the legacy `L2OutputOracle` whereby an authorized `Proposer` can create proposals
and an authorized `Challenger` can remove those proposals.

## Problem Statement

The `PermissionedDisputeGame` currently allows *both* the `Proposer` and the `Challenger` to
participate in the game and create moves. The original intent of this behavior was to have the
`PermissionedDisputeGame` act as a fully-fledged allowlisted dispute game. However, a number of
changes to product needs and security modeling have popped up.

- The concept of an allowlisted dispute game seems less important as the permissionless game
  becomes increasingly battle-tested. Current product need for an allowlisted game is unclear and
  the need for a such a game where only two participants can play is even less clear.
- The `Proposer` tends to be a hot wallet which is easier to compromise compared to the
  `Challenger` which is usually a multisig. In the case that the `PermissionedDisputeGame` is being
  used as a fallback mechanism, the game is likely being used because the `FaultDisputeGame` has a
  known bug. If the `Proposer` is compromised then it could be used to try to exploit this known
  bug, which ultimately defeats the purpose of the fallback.

We therefore propose to modify the `PermissionedDisputeGame` to resolve these issues.

## Proposed Solution

Given the above developments, this design doc proposes to remove the ability for the `Proposer` to
submit moves. This would have the effect of aligning the `PermissionedDisputeGame` with the
behavior of the original `L2OutputOracle` where a challenge by the `Challenger` role has ultimate
authority.

If future product research demonstrates a clear need for an allowlisted dispute game, we would
recommend that an entirely new game type be written (as opposed to repurposing the
`PermissionedDisputeGame`).

## Risks & Uncertainties

### Security Modeling

This proposal slightly changes the security model of the `PermissionedDisputeGame` so that the
`Proposer` can no longer counteract a malicious `Challenger`. However, this is exactly the security
model of the `L2OutputOracle` and therefore more accurately mirrors what existing chain operators
think that a permissioned proof model would look like.

As previously stated, if there is clear product need for an allowlisted and fully fledged dispute
game, we would recommend creating a new game type that clearly supports this behavior.
