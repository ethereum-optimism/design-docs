# Simplified Guardian System

## Purpose

Provides a design for a simplified Guardian role that aligns the OP Stack with the latest update to
[L2Beat's Stage 1 definition][1].

## Context

L2Beat recently published [an update][1] to the Stages Framework that is scheduled to take effect
in the middle of 2025. The OP Stack is impacted by this update. Modifications will need to be made
to guarantee that the OP Stack continues to be compatible with the Stage 1 definition.

## Problem Statement

L2Beat's updated definition specifically impacts ability to use the Deputy Guardian alongside the
default Guardian role. The Deputy Guardian conflicts with the updated Stage 1 definition because it
allows a role *other* than the Guardian (assumed to be held by a Security Council) to execute
unilateral actions that cannot always be easily reversed by the Guardian.

Specifically, L2Beat highlights the following problematic case for OP Mainnet:

1. Deputy Guardian (held by Optimism Foundation) triggers the fallback to the
  `PermissionedDisputeGame`.
2. Deputy Guardian convinces a blocking minority of the Security Council by some means to *not*
  switch back to the `PermissionlessDisputeGame`, to *not* challenge any of the dispute game
  proposals created by the permissioned Proposer role, and to not trigger the Superchain-wide pause
  mechanism.
3. Permissioned Proposer role publishes an invalid proposal and the Security Council fails to
  challenge the dispute game through any of its available mechanisms.
4. Malicious party is able to execute an invalid withdrawal based on the invalid proposal.

L2Beat's updated Stage 1 definition, once it goes into effect in the middle of 2025, would define a
system that *allows* this situation as a Stage 0 system. We must therefore make adjustments to the
incident response process so that OP Mainnet and other similarly configured systems remain Stage 1
when this definition goes into effect.

## Proposed Solution

### Remove the Deputy Guardian

We begin by removing the Deputy Guardian role entirely. The Guardian role would be held exclusively
by the 1/1 Security Council Operations Safe (held by the Security Council). All Guardian actions
except for the pause mechanism would therefore only be executable by a majority of the Security
Council. This eliminates the intial problem posed by the updated Stage 1 definition.

### Deputy Pause Module v2

We still want the Optimism Foundation to retain the power to quickly respond to potential
incidents. We would therefore install a modified version of the Deputy Pause Module into the
Security Council Operations Safe and permit the Optimism Foundation Safe (or whatever party
previously held the Deputy Guardian role for that system prior to this change) to manage the Pause
Deputy account. This permits the Pause Deputy to trigger the pause mechanism freely.

### Pause Expiry

Because the Pause Deputy could trigger the pause mechanism at any time and the unpause function
would require participation from a majority of the Security Council, we must either (1) allow a
minority of the Security Council to unpause the system or (2) have the pause automatically expire
after some period of time. Given the potential security concerns with allowing a minority of the
Security Council to unpause, we prefer option (2).

We therefore propose that the pause would expire after 6 months. Once the pause is used, the pause
cannot be called again unless it is reset by a majority of the Security Council (this applies in
all cases, regardless of whether the system unpauses due to expiry or due to explicit unpause).

### Cluster-wide Pause

Because the pause mechanism would become the main mechanism available to the Optimism Foundation,
the pause mechanism would also become the basis for an updated incident response process. However,
the Superchain-wide Pause is far too impactful to be the only available option during an incident.
Given that we suspect many incidents will take the form of *single* impacted chains or chain
clusters, it may be valuable to implement a Cluster-wide Pause on top of the Superchain-wide Pause.

A chain would be paused if *either* the Superchain-wide Pause is active or its Cluster-wide Pause
is active. We would implement the Cluster-wide Pause by allowing the Guardian to pause a specific
chain/cluster as identified by a `ETHLockbox` contract address. That is, the Guardian would call
`SuperchainConfig.pause(ethLockboxAddress)`. Other contracts would then check the status of the
pause by calling `SuperchainConfig.paused(myEthLockboxAddress)` which would return true if either
that specific system is paused or the entire Superchain is paused.

This has the advantage of guaranteeing that a cluster of chains that share a bridge and proof
system can easily be paused together as a result of their shared `ETHLockbox` address. Exact
details of this mechanism will be specified later.

## Considerations

### Pause Expiry Security Implications

Pause Expiry has non-trivial security implications in that a bug in the proof system alongside a 
malicious blocking minority of the Security Council could be catastrophic. L2Beat explciitly argues
that this security model is better than the model in which a malicious Foundation and a blocking
minority of the Security Model is catastrophic. Since this is fundamental to the updated Stage 1
definition, we are required to accept this conclusion. We may want to take this as motivation to
implement some other proof system as a backup to the primary proof system.

### Permissioned Proof System

A chain will be considered Stage 0 if it falls back to the Permissioned Proof System. We should
accept this potential for a downgrade and generally rely on the pause mechanism as the primary
incident response tool.

We propose the following incident response protocol:

- Always attempt to resolve an issue offchain if possible, before a game resolves incorrectly. This
  scenario would typically occur if the forecasted incorrect resolution is the result of an issue
  with offchain infrastructure rather than a bug in smart contracts.
- Determine if the issue is chain-specific (e.g., missing honest actor) or applies to all chains
  (e.g., contract bug in dispute game).
  - If chain-specific or cluster-specific, execute the chain-specific or cluster-specific pause.
  - If applies to all chains, execute the Superchain-wide pause.
- We immediately determine if we believe that the bug can be resolved within 2 weeks.
- If the bug can be resolved within 2 weeks (via Security Council emergency upgrade) then we
  keep the pause active until the bug is resolved, upgrade the game, or retire all existing games,
  and unpause.
- If the bug cannot be resolved within 2 weeks or if 2 weeks have actually passed since the pause
  was executed then we switch to the Permissioned Proof System, or retire all existing games, and
  unpause.

<!-- References -->
[1]: https://forum.l2beat.com/t/stages-update-a-high-level-guiding-principle-for-stage-1/338
