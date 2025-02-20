# Interop Launch Sequence

## Purpose

This document describes a proposed sequence of events that will occur when interop goes live on
mainnet. The exact sequence is subject to change, but this document is meant to provide a scaffold
that helps drive other design documents.

## Proposed Sequence

- Far before the launch of interop, for all chains, in an OPCM action per chain:
  - An `ETHLockbox` contract is deployed for the chain.
  - The Upgrade Controller approves the `OptimismPortal` address to use the `ETHLockbox`.
  - Chains are upgraded to the new `OptimismPortal` implementation.
    - Because `usingSuperRoots` is `false` by default, this will not result in any immediate impact
      on the proving codepath that users will have to use.
    - `OptimismPortal` will point to the `ETHLockbox`.
  - The Upgrade Controller triggers the `migrate` function on the `OptimismPortal` which causes the
    `OptimismPortal` to transfer all of its ETH into the `ETHLockbox`.
- Shortly before the scheduled activation of interop on L2:
  - The Superchain-wide pause is activated.
    - TODO: Decide if this is really the correct way to do this, or if there's some other mechanism
      that works well enough. We essentially need to stop users from using the old proof system
      while the new proof system is spun up.
  - TODO: Needs more context from protocol & proofs.
- At interop activation time on L2:
  - Interop transactions can now happen.
  - Interop is officially considered active on L2, though withdrawals from the new interop system
    cannot yet be processed on L1.
  - TODO: Needs more context from protocol & proofs.
- After interop activates on L2, in a single OPCM action:
  - A new `AnchorStateRegistry` is deployed with a valid initial anchor state.
    - TODO: Decide if the anchor state is set as an authorized input by governance or by some sort
      of more automated transition process.
  - A new `DisputeGameFactory` is deployed and the game implementations and init bonds for the
    Super Root versions of the `FaultDisputeGame` and `PermissionedDisputeGame` are set.
- At interop launch time, in a single OPCM action:
  - An `ETHLockbox` contract is deployed.
  - The Upgrade Controller triggers an `upgrade` function on all `OptimismPortal` contracts that
    changes the `ETHLockbox` inside of that portal to the new shared lockbox, updates the
    references to the `DisputeGameFactory` and `AnchorStateRegistry` contracts, and sets the
    `usingSuperRoots` variable to `true`.
  - The Upgrade Controller approves all portal contracts to use the new lockbox.
  - The Upgrade Controller approves all other lockbox contracts to migrate into the new lockbox.
  - The Upgrade Controller triggers the migration function on all other lockbox contracts to
    transfer ETH from those lockboxes into the single shared lockbox.
  - Interop is "fully" deployed and withdrawals from the interop system can be processed on L1.
- At some time after interop launches, for all chains, in an OPCM action per chain:
  - A new `AnchorStateRegistry` is deployed with a valid initial anchor state.
    - TODO: Same question as before.
  - A new `DisputeGameFactory` is deployed and the game implementations and init bonds for the
    Super Root versions of the `FaultDisputeGame` and `PermissionedDisputeGame` are set.
  - The Upgrade Controller for that chain triggers and `upgrade` function on the `OptimismPortal`
    contract that updates the references to the `DisputeGameFactory` and `AnchorStateRegistry`
    contracts, and sets the `usingSuperRoots` variable to `true`.
