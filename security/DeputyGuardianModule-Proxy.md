# Proxify the `DeputyGuardianModule`

## Context

The `DeputyGuardianModule` is an important component for the Superchain security model.
The `DeputyGuardianModule`, also known as _DGM_, is a smart contract deployed on L1 (currently [here](https://etherscan.io/address/0xc6901f65369fc59fc1b4d6d6be7a2318ff38db5b)) that has, for example, the ability to widely pause the Superchain.
The _DGM_ is owned by the _Foundation Operation Safe_ so only operation run from the _Foundation Operation Safe_ can be executed into the _DGM_.
To make sure we can widely pause the Superchain rapidly enough in case of an emergency, we need to presign the pause transaction from the signers of the _FoS_. These transactions are also known as `PSP` for PreSigned Pause transactions.
This process requires a ceremony and this is slow and tedious to put in place.

The list of available actions that the `DeputyGuardianModule` can execute are:
| Function Name | Description |
| ---------------------- | ---------------------------------------------------------- |
| setRespectedGameType() | Change the current `gameType` used by the `OptimismPortal` |
| blacklistDisputeGame() | Blacklist a disputeGame into the `OptimismPortal` |
| unpause() | Unpause the withdrawal of the Superchain |
| pause() | Pause the withdrawal of the Superchain |
| setAnchorState()| Set a state into `AnchorStateRegistry` |

## Problem Statement

The existing `DeputyGuardianModule` is not _proxified_. This is inconvenient each time we want to upgrade the `DeputyGuardianModule` to add a new feature or to fix a potential bug.
As this is not a proxy this require to redeploy the `DeputyGuardianModule` on L1. And also, to update the `Guardian` contract to add the new `DeputyGuardianModule` address as authorized module.
Moreover, changing the `DeputyGuardianModule` will break the current PSPs setup as these pause transactions are presigned with the **previous** `DeputyGuardianModule`.
Thus, we need to generate new PSPs through the tedious process of a ceremony to add the new `DeputyGuardianModule` for making the PSPs valid again.
We are clearly seeing that this is not a sustainable solution for the long term.

Additionally, we have to simulate the new PSPs and share them with other member of the Superchain.

## Proposed Solution

Here we propose some modification of the architecture with a proxy contract on top of the `DeputyGuardianModule`, this will allow us to upgrade the `DeputyGuardianModule` and keeping the same address.
Thus, this will not invalidate the PSPs and we will not need to regenerate new PSPs each time we upgrade the `DeputyGuardianModule`.

The owner of the proxy contract `DeputyGuardianModuleProxy` would be the `Security Council` as the `DeputyGuardianModule` is doing action into the `Guardian` contract that is owned by the `Security Council`.
This will means that only the security council would be able to upgrade the implementation of the `DeputyGuardianModuleProxy`.

## Alternatives Considered

No other alternatives were considered for now.

## Risks & Uncertainties

### Security Model

We don't see any particular risk with this current design for the long term as this is a common pattern to use a proxy contract for upgrading a contract.

A potential issue can occur with the PSPs coverage.
During the upgrade the PSPs are going to be invalid so we need to make sure to presign new PSPs with the new `DeputyGuardianModuleProxy` address and simulate the PSPs accordingly.
