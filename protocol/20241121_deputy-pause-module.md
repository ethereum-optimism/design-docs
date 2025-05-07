# Deputy Pause Module

## Context

The Foundation Safe is given the ability to act as the Security Council Safe for a limited number
of safety-net actions on OP Mainnet. For instance, the Foundation Safe can act as the Security
Council Safe to trigger the Superchain-wide pause function. These powers are given to the
Foundation Safe through the `DeputyGuardianModule` installed in the Security Council Safe.

## Problem Statement

The Foundation Safe pre-signs certain transactions to reduce response time in case of an emergency.
Because pre-signed Foundation Safe transactions are dependent on the nonces of that Safe, these
pre-signed transactions become invalidated any time the Foundation Safe is required to sign
literally any other transaction. In an attempt to reduce the overhead from this state of affairs,
a *second* Foundation Safe sometimes referred to as the "Foundation Operations Safe" with the exact
same configuration as the original Foundation Safe was made to be the Deputy Guardian.

Even with this second Foundation Safe, pre-signed pauses are invalidated on a regular basis
whenever an upgrade touches the `DeputyGuardianModule`. Gripes with the pre-signed pause system
could fill a whole role of toilet paper and are not just limited to the issues noted above.

## Proposed Solution

We propose eliminating the pre-signed pause system entirely by introducing a `DeputyPauseModule` to
be installed into the primary Foundation Safe. The `DeputyPauseModule` would allow a deputy account
to act as the Foundation Safe to trigger the Superchain-wide pause functionality. The deputy
account would not have any other capabilities.

The `DeputyPauseModule` solves a litany of problems. Because the deputy account could be made to be
an externally owned account, we gain the benefit of the pre-signed pause (immediate pause
execution) without all of the many downsides of a pre-signed transaction. We could also remove the
Foundation Operations Safe entirely, generally simplifying our multisig setup.

### EOA vs Smart Contract

We propose using an Externally Owned Account (EOA) instead of a smart contract as the deputy here.
Using an EOA is simpler and easier to reason about. The private key for this EOA can be stored
securely and made accessible to a limited set of security personel.

### Single Account vs Mapping

We propose having the `DeputyPauseModule` use a single account instead of a mapping of accounts
that are able to act as the deputy. A single account is easier to keep track of and having multiple
accounts does not decrease the risk involved with this module, it simply spreads it across more
private keys. Having multiple keys be able to act as the deputy here might have some slighty
benefits but this begins to scope creep beyond the original intention of replacing the pre-signed
pause functionality.

### Exact Proposed Modifications

We currently have two Safe accounts for the Foundation:

1. `0x847B5c174615B1B7fDF770882256e2D3E95b9D92` - known as the "Foundation Upgrade Safe" was
  deployed on 2024-02-08 and is used as one owner on the 2/2 System Owner Safe that can upgrade
  OP Mainnet (alongside the Security Council).
2. `0x9BA6e03D8B90dE867373Db8cF1A58d2F7F006b3A` - known as the "Foundation Operations Safe" was
  deployed on 2021-01-16, is used for pre-signed pauses, and is one signer on the 2/2 System Owner
  Safe that can upgrade Base.

We propose the following actions:

1. `DeputyPauseModule` will be installed into `0x9BA6`.
1. `0x847B` will be replaced with `0x9BA6` in the OP Mainnet 2/2.
1. `0x847B` will be deprecated generally.

Please note that step (2) requires action by the Security Council and governance communication. If
going through the whole process of replacing one of the 2/2 signers is too much of a pain, we can
also consider the following alternative:

1. `DeputyPauseModule` will be installed into `0x9BA6`.
1. `DeputyPauseModule` will be installed into `0x847B`.
1. Security Council sets `0x847B` as the Deputy Guardian.
1. Base replaces `0x9BA6` with `0x847B` in its 2/2.
1. `0x9BA6` is deprecated generally.

## Risks & Uncertainties

### Audits

Any module will need to be carefully audited. We've already had the `DeputyGuardianModule` audited
and the `DeputyPauseModule` would follow the example set by the `DeputyGuardianModule` closely. We
would not expect this to be a risky addition to the Foundation Safe.

### Leaked Deputy

Our worst-case scenario is a leaked deputy private key. An attacker would probably simply trigger
the pause if this ever happens. Since the attacker could theoretically continue to pause if the
system is unpaused, the appropriate response would be to first replace the module with a new one
that has a deputy key that has not been leaked and then to unpause the system.

### Compromised Deputy

A motivated attacker with access to the deputy key could also try to drain the deputy wallet
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
