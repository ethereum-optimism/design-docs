# Expanded Permissioned Challenger

## Context

The OP Stack currently officially provides two dispute game types, the standard `FaultDisputeGame`
contract and the alternative `PermissionedDisputeGame`. The `PermissionedDisputeGame` is meant to
act either as the default game for chains that, for whatever reason, do not want to run a
permissionless game or as a fallback game type if bugs in the `FaultDisputeGame` are discovered.

The `PermissionedDisputeGame` for a chain specifies a `CHALLENGER` address that is allowed to
challenge a proposal created by the pre-defined `PROPOSER` address. Most OP Stack chains configure
this `CHALLENGER` address to be the same as the System Owner address.

## Problem Statement

Most OP Stack chains configure the `CHALLENGER` address to be the same as the System Owner address.
OP Mainnet currently also follows this pattern. However, OP Mainnet is unique in that the System
Owner address is a 2/2 multisig between the Optimism Foundation and the Optimism Security Council.
Combined with the Optimism Foundation's role as the Deputy Guardian, this can lead to some
interesting security edge cases.

Recent standardization over the last few months around ideal security models has led to the
conclusion that, until Stage 2, we should aim to operate under the assumption that a minority of
the Security Council can only cause a liveness failure (and not a safety failure). The current
model for the OP Mainnet means that a minority of the Security Council could cause a safety failure
if this minority is coordinating with a compromised Optimism Foundation multisig.

Specifically, the attack vector in question would function as follows:

1. Compromise or otherwise cause the Optimism Foundation multisig to act maliciously.
1. Use the Optimism Foundation's power as Deputy Guardian to trigger the fallback.
1. Create an invalid proposal and corresponding dispute game.
1. Cause a minority of the Security Council to *not* act to invalidate the proposal.

We want to resolve this discrepancy with minimal contract changes.

## Proposed Solution

We propose resolving this discrepancy by changing the address of the `CHALLENGER` to be a smart
contract that allows the Guardian OR any member of the Security Council Safe to trigger a
challenge. This means that either of these two groups could cause a liveness failure if the system
is using the `PermissionedDisputeGame` but also means that the Foundation Safe can no longer cause
a safety failure in coordination with a minority of the Security Council Safe.

Since this would be a change to the address of the `CHALLENGER`, we would not need to modify the
`PermissionedDisputeGame` contract itself. We would introduce a new contract that essentially has
the name/role `PermissionedChallenger` where the Guardian or any member of the Security Council
Safe can act as the `PermissionedChallenger`.

For simplicity, we constrain the `PermissionedChallenger` to calling the `move` function with a
pre-defined fake claim that can only be triggered against the top-level claim. Since no other moves
are possible, the `PermissionedChallenger` will always win.

The interface for the `PermissionedChallenger` would be something like:

```solidity
interface IPermissionedChallenger {
    function challenge(IDisputeGame _game) external;
}
```

We would additionally need to implement the proposal in
[Design Doc 138](https://github.com/ethereum-optimism/design-docs/pull/138) to guarantee that the
proposer cannot also play in the dispute game, which would potentially allow the proposer to
exploit bugs in the dispute game if they exist. Given that the `PermissionedDisputeGame` fallback
is primarily meant to be used in the case that a bug in the `FaultDisputeGame` is discovered, it
is very realistic that such a bug would exist if the `PermissionedDisputeGame` is being used.

## Alternatives Considered

### Challenger Array

We considered modifying the `PermissionedDisputeGame` to accept an array of challenger addresses
instead of just a single `CHALLENGER`. This has a number of downsides:

1. We would have to modify the `PermissionedDisputeGame`.
1. We would have to re-deploy the `PermissionedDisputeGame` any time that the Security Council Safe
   updates its set of members.
1. It would be harder to quickly understand who can act in the `PermissionedDisputeGame`.

## Risks & Uncertainties

### Liveness

Our primary risk with this proposal is that any member of the Security Council Safe can now cause a
liveness failure if the game is in the `PermissionedDisputeGame` mode. Although this is the desired
behavior, it also means that a malicious Security Council Safe member can cause a liveness failure
in the `PermissionedDisputeGame` until that member is replaced by a majority of other members.

### Audit

We may have to carry out a brief audit on the `PermissionedChallenger` contract. We do not expect
that this contract would be particularly complex, so such an audit would be relatively minimal.

### L2Beat

We should confirm with L2Beat that this proposal resolves the concerns that have been raised.

### Runbooks

We would need to update any existing runbooks with this new protocol.
