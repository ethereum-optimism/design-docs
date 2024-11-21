# Deputy Guardian Module Improvements

## Context

The Foundation Safe is given the ability to act as the Security Council Safe for a limited number
of safety-net actions on OP Mainnet. For instance, the Foundation Safe can act as the Security
Council Safe to trigger the Superchain-wide pause function. These powers are given to the
Foundation Safe through the `DeputyGuardianModule` installed in the Security Council Safe.

## Problem Statement

The existing `DeputyGuardianModule` has a number of problems that we'd like to be able to solve in
a simple manner that makes as few contract changes as possible. We describe each of these problems
individually for clarity.

### Pre-Signed Pause Transactions

The Foundation Safe pre-signs certain transactions to reduce response time in case of an emergency.
Because pre-signed Foundation Safe transactions are dependent on the nonces of that Safe, these
pre-signed transactions become invalidated any time the Foundation Safe is required to sign
literally any other transaction. In an attempt to reduce the overhead from this state of affairs,
a *second* Foundation Safe sometimes referred to as the "Foundation Operations Safe" with the exact
same configuration as the original Foundation Safe was made to be the Deputy Guardian.

Even with this second Foundation Safe, pre-signed pauses are invalidated on a regular basis
whenever an upgrade touches the `DeputyGuardianModule`. Gripes with the pre-signed pause system
could fill a whole role of toilet paper and are not just limited to the issues noted above.

### Unclear Responsibilities

The `DeputyGuardianModule` has an unclear security model and underlying purpose. The original
intent behind the `DeputyGuardianModule` was to provide a fast way to respond to potential
incidents in a manner that can impact liveness but not system safety. A lack of strict clarity
behind this design has meant that the `DeputyGuardianModule` has become a source of security
implications that are not often easy to reason about.

For instance, the `DeputyGuardianModule` currently allow the Deputy Guardian account to change the
dispute game type that is respected within the `OptimismPortal` contract from the
`FaultDisputeGame` to the `PermissionedDisputeGame`. The `PermissionedDisputeGame` has a security
model that differs from the `FaultDisputeGame` which essentially means that the Deputy Guardian is
given the ability to change the security model of the system. This is a departure from the original
design of the `DeputyGuardianModule` which should only impact liveness and should not modify the
security model of the system.

## Proposed Solution

We propose resolving the above problems by modifying the existing `DeputyGuardianModule` to be more
restrictive in the actions it can carry out while also being more permissive in who can carry out
these actions.

The `DeputyGuardianModule` would be permitted to:

1. Trigger the Superchain-wide pause.
1. Blacklist an individual dispute game.

The `DeputyGuardianModule` would **no longer** be permitted to:

1. Undo the Superchain-wide pause ("unpause").
1. Change the respected game type.
1. Set the anchor state in the `AnchorStateRegistry`.

The `DeputyGuardianModule` would be accessible to:

1. Any member of the Security Council Safe.
1. An EOA operated by the Optimism Foundation.

Please note that this proposal supercedes
[Design Doc 162](https://github.com/ethereum-optimism/design-docs/pull/162) that introduced the
concept of the `DeputyPauseModule`.

### Reasoning

The above changes make the role of the `DeputyGuardianModule` significantly more obvious. The
`DeputyGuardianModule` is now designed to be a fast response mechanism that can ONLY impact
liveness and cannot impact safety.

1. We remove the ability to unpause the system because this makes it possible for the
   `DeputyGuardianModule` to diminish safety by unpausing the system when it was explicitly placed
   into a paused state for security reasons.
1. We remove the ability to change the respected game tybe because this allows the
   `DeputyGuardianModule` to modify the security model of the system and does not align with the
   goal of limiting the module to liveness-impacting actions.
1. We remove the ability to set the anchor state within the `DisputeGameFactory` because this can
   potentially impact safety if the provided anchor state game is sufficiently old.

## Alternatives Considered

### Deputy Pause Module

This proposal supercedes a previous proposal that would have installed a `DeputyPauseModule` into
the Foundation Safe. We would be able to get the benefits of this proposal while also simplifying
the role of the Deputy Guardian if we accepted this proposal instead.

## Risks & Uncertainties

### Audits

Any module will need to be carefully audited. The `DeputyGuardianModule` was previously audited but
would need to be re-audited as a result of these changes.

### Leaked Deputy

Our worst-case scenario is a leaked deputy private key. Since the `DeputyGuardianModule` would now
permit any member of the Security Council Safe or a dedicated Foundation EOA to trigger
liveness-impacting actions, such a leak would likely cause a temporary liveness failure. A majority
of the Security Council Safe would need to coordinate a transaction to remove or replace the
offending account and resolve the liveness failure.

### Compromised Deputy

A motivated attacker with access to a deputy's private key could try to drain the deputy's wallet
constantly to prevent it from being used to trigger transactions. Although this could be
side-stepped with private RPC/bundling tools, it would not be considered good practice to include
live third-party infrastructure in the hot path for critical security actions. It therefore seems
prudent that the deputy be able to act via signature instead of directly via `msg.sender` check.

If we allow the deputy to act via signature then we must also make sure that the signature includes
some sort of nonce to prevent the same signature from being used more than once.

### Process Updates

Various processes and runbooks will need to be updated to reflect this new state of affairs. We'll
be happy, but it will take some effort. All of these updates will need to be made before we
actually go live with these changes and we should run drills on testnet and propose that drills be
run on mainnet.
