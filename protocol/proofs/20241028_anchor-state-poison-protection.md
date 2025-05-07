# AnchorStateRegistry Poisoning Mitigation

## Purpose

Provides a design for an updated version of the `AnchorStateRegistry` with a clearer role within
the system as a whole that is not subject to the "poisoning" problem found in the current design.

## Context

The `FaultDisputeGame` contract is responsible for managing the process of disputing a fault in the
computation of a state transition function. The `FaultDisputeGame` runs the state transition
function from some starting state that is known to be valid. We refer to this starting state as the
"anchor state". In the context of the OP Stack, the anchor state will originally either be the
genesis state or the state of the system at some finalized L2 block height that is considered to be
correct by social consensus.

`FaultDisputeGame` contracts pull the current anchor state from the `AnchorStateRegistry` contract.
As the system runs, the anchor state is continuously updated to reflect the latest finalized state
as viewed by the proof system at any given time. `FaultDisputeGame` instances that resolve in favor
of the defender will trigger the `AnchorStateRegistry` to update the anchor state to the newly
finalized state.

## Problem Statement

The `AnchorStateRegistry` can become "poisoned" because the ultimate view of state validity
currently actually lives inside of the `OptimismPortal` contract. An anchor state is considered
"poisoned" if the `OptimismPortal` contract has blacklisted the dispute game that caused that
anchor state to be recognized by the `AnchorStateRegistry`. Any new `FaultDisputeGame` contracts
that load in this anchor state will operate over a starting state that the `OptimismPortal`
considers to be invalid and will therefore also be considered invalid.

Although a basic mitigation was added in the Granite upgrade that made it possible to reset the
anchor state to some different (valid) state, the fundamental problem still remains that poisoned
`FaultDisputeGame` contracts may continue to resolve and further poison the anchor state registry.
In this cascading case, the only real solution at the moment is to switch to a new game type as
each game type has its own anchor state mapping and therefore is not impacted by any poisoned
anchor states in other game types.

Our problem statement here is therefore that we wish to be able to prevent or mitigate anchor state
poisoning more effectively so that we can reduce the surface for issues within the fault proof
system as a whole.

## Proposed Solution

We propose solving this problem by updating the `AnchorStateRegistry` to more generally be the
source of truth for the validity of states from the perspective of the L1 proof system. All
functionality in the `OptimismPortal` related to state validity would be moved to the
`AnchorStateRegistry` contract. This proposed solution would simplify the `OptimismPortal`
contract, clarify the concept of state validity from the PoV of the proof system, and solve the
state poisoning issue within the `AnchorStateRegistry`.

### Changes to the AnchorStateRegistry

The `AnchorStateRegistry` would be modified to take over the blacklisting, respected game type,
and air-gap roles that are currently handled by the `OptimismPortal` contract. The
`AnchorStateRegistry` would expose several new getter functions that can be used by the
`OptimismPortal` contract to determine if a given dispute game can be permitted to prove or
finalize a withdrawal.

The `AnchorStateRegistry` would continue to keep track of the results of `FaultDisputeGame`
contracts via the current "push" model where the `FaultDisputeGame` attempts to trigger an update
in the `AnchorStateRegistry` on game finalization. However, instead of immediately updating the
anchor state when a game is finalized, the `AnchorStateRegistry` would instead store the result in
a log of game results. Game results would only be promoted to anchor states if they've passed the
air-gap window without being blacklisted.

We can avoid making significant changes to the current challenger setup by using the call to the
`tryUpdateAnchorState` function that the `FaultDisputeGame` already makes to both store the game
result in the game array. The `AnchorStateRegistry` could then iterate over some number of elements
in this array every time `tryUpdateAnchorState` is triggered to try to find a new anchor state. As
games tend to be published at a relatively regular interval, this means that the
`AnchorStateRegistry` will tend to find new anchor states regularly without too much trouble. We
can also expose a new permissionless method on the `AnchorStateRegistry` that allows anyone to show
that a newer anchor state is valid to manually push up the anchor state if necessary.

The `AnchorStateRegistry` would no longer keep track of different anchor states for different game
types. Each game type would share a single anchor state. The `AnchorStateRegistry` would only use
the results from the currently respected game type to update the anchor state. Note that this
proposal does NOT support the ability for multiple proof systems to independently update the anchor
state (i.e., possibility for multiple respected game types at the same time) as that feature was
not discussed during the design review meetings.

### Changes to the OptimismPortal

State validity rules like blacklisting and air-gap window inside of the `OptimismPortal` would be
removed and outsourced to the `AnchorStateRegistry`. New public getter functions in the
`AnchorStateRegistry` would provide the information that the `OptimismPortal` needs to determine
whether a given dispute game contract can be used to finalize a withdrawal.

### Changes to the FaultDisputeGame

No changes to the `FaultDisputeGame` contract would necessarily be required. `anchors` function can
be made backwards compatible to always return the single anchor state now that anchor states will
not be tracked on a per-game basis. We can optionally update the `FaultDisputeGame` to use whatever
new function exposes the single `anchor` if desired.

### Changes to the DeputyGuardianModule

The `DeputyGuardianModule` will have a few light changes that will mean that the DGM will need to
be redeployed. Any redeploy of the DGM breaks PSPs and means we would need to regenerate new PSPs.

## Upgrade and Migration Path

We must guarantee that the security properties of the previous system are maintained into the
updated system. In particular, any games that have been blacklisted in the old system must also be
blacklisted in the updated system. Additionally, it may be necessary to forward certain getter
functions currently available on the `OptimismPortal` contract like `disputeGameBlacklist` over to
the `AnchorStateRegistry`.

## Alternatives Considered

### Use OptimismPortal in AnchorStateRegistry

We considered updating the `AnchorStateRegistry` to ask the `OptimismPortal` if a particular game
has been blacklisted. A naive version of this alternative can prevent blacklisted games from being
used for the anchor state as long as they've been blacklisted before the air-gap window starts.
Since we do expect some games to be blacklisted *after* the air-gap window starts (but before it
expires) this does not fully solve the problem.

We can also make the changes to the `AnchorStateRegistry` described above *without* moving the
blacklisting and air-gap logic out of the `OptimismPortal` contract. This would still fix the
poisoning issue but would introduce additional complexity into the `AnchorStateRegistry` without
reducing any complexity in the `OptimismPortal` contract. Would additionally spread responsibility
for anchor state validity across multiple contracts, which isn't ideal. However, this would have
the benefit of not needing to touch the `OptimismPortal` at all which reduces bug surface and risk.

## Risks & Uncertainties

### Security

`OptimismPortal` is one of the single most security-critical contracts within the OP Stack system.
Any change that touches the `OptimismPortal` will likely need to be carefully audited.

### Backwards Compatibility

Moving the blacklisting capability over to the `AnchorStateRegistry` will introduce some changes to
client code that interacts with this capability.
