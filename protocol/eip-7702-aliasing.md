# Address Aliasing for EIP-7702

## Purpose

Describes and unifies an updated logic for address aliasing that properly accounts for the Pectra
hardfork and the inclusion of [EIP-7702][1].

## Problem Statement

[EIP-7702][1] introduces a new mechanism that allows EOAs to specify a delegation that causes the
EOA to operate as a pseudo-contract that uses the bytecode of the delegation target. A signature
from the private key can revoke the delegation immediately and at any time. As a result, EOAs that
have code as a result of EIP-7702 are still functionally EOAs in the sense that they are ultimately
fully controlled by the account's private key.

The `OptimismPortal` will alias the address in the forced transaction that it generates if the
sender is not an EOA. We do this check because we do not allow smart contracts on L1 to act as the
same contract on L2. Contracts on different chains are fundamentally different objects and should
not be treated as the same even though they have the same address.

Given the change introduced by EIP-7702, we need to figure out how to properly handle EIP-7702
delegated accounts.

## Proposed Solution

Our existing code would produce the following scenario:

1. `EOA #1` triggers `OptimismPortal`
2. `OptimismPortal` generates forced tx and does not alias the address of `EOA #1`
3. `EOA #1` delegates via EIP-7702
4. `EOA #2` triggers `EOA #1` (possible because of step 3) which triggers `OptimismPortal`
5. `OptimismPortal` generates forced tx and DOES alias the address of `EOA #1`

We believe that this is not the correct behavior and that the `OptimismPortal` should not alias the
addresses of EOAs that have been delegated via EIP-7702 because a delegated EOA is still
fundamentally under the control of the private key for that EOA. That is to say, the private key
for the EOA can always revoke the delegation and execute any arbitrary transaction anyway, so
there's no value in preventing the EOA from executing arbitrary transactions via the forced txn
route.

We therefore propose that the `OptimismPortal` be updated to NOT alias the addresses of any 7702
delegated accounts even though these accounts have code.

### Additional Changes

Some other components of the OP Stack also check that the sender is an EOA:

- The `PreimageOracle` requires that preimages be published by an EOA. However, this is a legacy
  behavior and is not strictly necessary. We keep this behavior out of an abundance of caution. We
  do not believe there is any reason to keep this check or that there is any downside to
  incorrectly classifying 7702 accounts as EOAs for this purpose.
- The `StandardBridge` and the `ERC721Bridge` both check that accounts attempting to trigger the
  `bridgeETH`, `bridgeERC20`, and `bridgeERC721` convenience functions are not smart contracts.
  These contracts use this behavior to try to prevent users from accidentally triggering deposits
  where both the sender and recipient are the `msg.sender` because of the aliasing behavior in the
  `OptimismPortal` contract. Preventing 7702 addresses from using these convenience methods is not
  a significant issue because the `bridgeETHTo`, `bridgeERC20To`, and `bridgeERC721To` methods
  exist as an alternative.

Although none of these contracts *need* to be updated, we also propose that we have a unified
mental model around EOA checks and generally consider both EOAs and 7702 delegated EOAs to
"officially" be EOAs for the purposes of these checks.

## Considerations

### Coordination With Offchain Labs

Offchain Labs also uses the address aliasing mechanism described here. We are in the process of
reaching out to the Offchain Labs team to (possibly) get in sync about a decision here.

### Addressing Additional Changes

All of the "additional changes" described above are technically optional, but we recommend
resolving them all simultaneously because it would reduce the fix into an easily testable EOA
checking library that could be reused everywhere.

<!-- references -->
[1]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md
