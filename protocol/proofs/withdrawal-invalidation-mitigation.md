# Withdrawal Proof Invalidation Mitigation

## Context

The `OptimismPortal` contract allows users to prove withdrawals by demonstrating that those
withdrawals exist inside of the state root of a `FaultDisputeGame` instance. Withdrawals can be
finalized if the `FaultDisputeGame` instance resolves in favor of the defender, the instance is not
blacklisted, and the game has sat around for the air-gap period.

The `DisputeGameFactory` can create different implementations of the `DisputeGame` which typically
have different code and are identified by an integer "game type". The `OptimismPortal` currently
defines a variable `respectedGameType` which determines the particular `DisputeGame` code that
users can utilize when proving and finalizing withdrawals. The "fallback" safety net action is the
ability for the Guardian or Deputy Guardian address to change the `respectedGameType` variable to
some alternative game type (e.g., the `PermissionedDisputeGame`) in the case that the
`FaultDisputeGame` contract is found to be buggy.

## Problem Statement

The fallback action of changing the `respectedGameType` variable updates another variable called
`respectedGameTypeUpdatedAt` that keeps track of exactly when the respected game type was changed.
Currently, the `OptimismPortal` contract enforces that withdrawals can only be proven or finalized
against `DisputeGame` contracts that were created with a timestamp greater than the
`respectedGameTypeUpdatedAt` variable. This effectively invalidates any withdrawal proofs that were
submitted prior to the activation of the fallback mechanism.

Withdrawal invalidation is a powerful tool in the incident response playbook as it can mitigate the
impact of a large number of invalid dispute games at the same time. However, the it also forces
users to resubmit their withdrawal proofs and wait an additional 7 days to execute a withdrawal.
Users have reported that this invalidation and additional delay period is a pain point. This user
impact means that the fallback is highly disruptive as a standard security mechanism.

We are in the process of further assessing the impact and severity of this pain point but are
creating a design document in the meanwhile to determine the lift to resolve this issue.

## Proposed Solution

We propose solving this problem by making withdrawal invalidation an optional part of the fallback
mechanism. We can achieve this by slightly modifying the rules for withdrawal verification.

We currently enforce the following set of rules (not including other rules like blacklist checks):

- Withdrawals must be proven against a dispute game of the `respectedGameType`.
- Withdrawals must be finalized against a dispute game of the `respectedGameType`.
- Withdrawals must be finalized against a dispute game created after `respectedGameTypeUpdatedAt`.

We propose removing the requirement that withdrawals must be finalized against a dispute game of
the `respectedGameType`. Withdrawals that have been previously proven against a dispute game when
that game *was* respected can be finalized even if that dispute game is no longer of the respected
game type.

We additionally propose that the `respectedGameTypeUpdatedAt` timestamp not be updated by default.
Instead, the timestamp can be selectively updated if withdrawal invalidation is actually required.
Combined with the first proposed change, this proposal means that older withdrawal proofs can
usually be finalized when the respected game type is changed but withdrawals can still be blanket
invalidated if necessary.

Finally we also propose to add the following to the set of verification rules:

- Withdrawals must be proven against a dispute game created after `respectedGameTypeUpdatedAt`.

This would have the effect of preventing users from being able to create withdrawal proofs that
could never actually be used to finalize a withdrawal. Although this is not strictly necessary, it
blocks off a footgun and we typically like to block off footguns where possible.

### Naming

We may want to consider renaming `respectedGameTypeUpdatedAt` to something more representative of
the new system given that the variable would not be updating every time that the
`respectedGameType` variable is changed. Alternatively, we could keep `respectedGameTypeUpdatedAt`
untouched and instead introduce a new variable that more accurately reflects the system. This would
also avoid a breaking change to the contract ABI and the semantics of the original variable.

## Alternatives Considered

No alternatives were considered as of the writing of this proposal.

## Risk & Uncertainties

### Security

The `OptimismPortal` is one of the most security-critical contracts within the OP Stack. Any
modifications to this contract would likely require a heavy audit.
