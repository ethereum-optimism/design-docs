# Super Root Withdrawal Proofs

## Purpose

Proposes a modification to the `OptimismPortal` that would add a new overload of
`proveWithdrawalTransaction` that would allow users to prove against Super Roots.

## Context

Interop introduces the concept of the "Super Root" - a collection of Output Roots from all of the
different chains in the interop set. The interop fault proof system argues over a super root rather
than an individual Output Root. The `OptimismPortal` contract only supports proving withdrawals
against a specific Output Root and will need to be modified to support the Super Root.

## Problem Statement

We need to modify the `OptimismPortal` contract to support proving withdrawals against a Super
Root. Users would not be able to correctly execute withdrawals without this modification. This
means that *some* solution to the problem is a basic requirement for interop.

## Proposed Solution

We propose to allow users to prove against Super Roots by adding a new overload of
`proveWithdrawalTransaction` that allows a user to prove the existence of an Output Root inside of
the Super Root before performing the standard withdrawal validation inside of the Output Root.

We also specifically propose that this be implemented on the *base* `OptimismPortal` instead of
introducing the code to the `OptimismPortalInterop` portal. Doing so means that we are including
extra code on the `OptimismPortal` that is technically unnecessary until interop ships but also
means that we do not need to ship any additional changes to the `OptimismPortal` before interop.

We could expose a new public variable `usingSuperRoots` that would toggle this function on and off
and would also act as a signal for client-side tooling to automatically switch between the two
available functions. When `usingSuperRoots` is `false`, the version of `proveWithdrawalTransaction`
for Super Roots would revert. When `usingSuperRoots` is `true`, the legacy version of the function
would revert.

## Alternatives Considered

### Using an Inherited Contract

We could implement this function inside of an inherited contract instead of putting it in the base
contract. However, this would mean performing two sets of upgrades to the `OptimismPortal` before
interop ships on an already tight timeline.

## Considerations

### Unused Code in Production

This proposal means we are shipping code that cannot yet be used into production. We believe this
trade-off is worthwhile because it means that we would not have to ship the code later on.

### Changes in Super Root Structure

If the Super Root structure changes, the `OptimismPortal` code will have to change too. Keeping the
code inside of an inherited contract would make this easier to do and wouldn't involve any type of
upgrade to mainnet systems. We can mitigate this by getting strong commitment from the designers of
the proof system that the Super Root will not change before interop goes live.

### Audits

We would want to audit this code *as if* the Super Root logic were already live. This is OK because
we can blackbox the Fault Dispute Game. We also need to audit that the Super Root logic cannot be
used unless it has been activated and that the inclusion of both functions is not dangerous for any
reason.

### Cross-Chain Withdrawals

We have decided that we will NOT allow the `OptimismPortal` for one chain to process withdrawals
from another chain when interop launches because of security implications with the correctness of
the cross-domain message sender. The Super Root withdrawal proof must therefore guarantee that a
given withdrawal was actually created on the specific chain that corresponds to the given
`OptimismPortal`. This means that we need to include a chain ID in the `OptimismPortal`.

## Failure Modes

### Bug in Proof Validation Logic

The only significant failure mode of this change is a bug in the new proof validation logic.
Although unlikely, the impact of this bug is only a HIGH because of existing withdrawal validity
monitoring. Withdrawal validity monitoring will detect invalid withdrawals regardless of whether
the proof verification bug is because of the output root logic or the merkle trie.
