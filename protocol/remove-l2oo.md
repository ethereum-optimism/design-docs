# Purpose

To outline the steps required to remove the pre-fault proofs support from the code base while maintaining support
for chains that are not yet able to use permissionless fault-proofs.

# Summary

With the fault proof system approved by Optimism Governance and being shipped to op-mainnet, we want to remove legacy
code to reduce maintenance costs. However, not all chains have upgraded to fault proofs yet and some configurations,
such as alt-DA, are not yet fault provable. Therefore, we need to maintain a supported configuration for chains that
are not using the permissionless fault proofs system.

# Problem Statement + Context

The fault proof system was developed in parallel with the legacy `L2OutputOracle` based proposals to ensure that chains
with and without fault proofs could both be supported. Most of the fault proof system is in new contracts such as
`DisputeGameFactory` and `FaultDisputeGame` but there are a few places where multiple implementations were required to
support the with and without fault proof cases:

* `OptimismPortal` is the pre-fp portal and `OptimismPortal2` is the post-fp version
* `op-proposer` contains support for publishing both via `L2OutputOracle` and the `DisputeGameFactory` via a CLI flag
* `make devnet-allocs` and `Deploy.s.sol` include feature toggle to enable or disable fault proofs
* `L2OutputOracle` is no longer required

To minimise maintenance costs we now want to consolidate all those cases to just retain the fault proofs version.

## Fault Proof Blockers

There are a few reasons that chains may not be able to move to the full fault proofs system yet:

* Custom gas tokens are currently only implemented in the pre-fp `OptimismPortal`
* Alt-DA has not been fully integrated into the fault proof program so alt-DA chains are not yet compatible with FP
* Running `op-challenger` to ensure all invalid claims are countered requires a significant amount of capital to be
  available (from the aggregate of all honest actors). Smaller chains may not have sufficient capital available to
  honest actors to ensure the chain remains secure when using a permissionless fault proof system.

# Alternatives Considered

## Maintain the Legacy Code

We could maintain support for the pre-FP `OptimismPortal`, giving more time for alt-DA to integrate with fault proofs
and chains to migrate to fault proofs.

* New features that integrated with the portal would not be available unless we duplicated the development effort.
* Incurs maintenance cost to continue maintaining this code

## Utilise Permissioned Game

The `PermissionedDisputeGame` can be used to limit participation in the fault proof system to only trusted actors.
This provides a model that is conceptually equivalent to using the `L2OutputOracle` but utilising the fault proof
contracts.

With the permissioned game, only the `Proposer` role can create new games and only the `Challenger` or `Proposer` role
can post claims in the game. Thus, like with the `L2OutputOracle`, only the `Proposer` can post output roots and the
`Challenger` can invalidate them. While the dispute game is created and theoretically could be played out, since the
`Proposer` and `Challenger` are the only possible participants and both are trusted roles, this could be used even in
situations where the chain is not fault provable.

### Costs

Currently bonds are hard coded in the `FaultDisputeGame` so these would have to be paid and then claimed back for each
proposal. This increases the capital requirements for the proposer, but not to the same extent as would be required for
a permissionless fault dispute system.

Creating a dispute game is more expensive than posting an output root to the `L2OutputOracle` and the total process
requires three transactions instead of one - create the game, resolve the root claim, resolve the game itself. However,
the `DisputeGameFactory` doesn't require posting output roots at fixed intervals like the `L2OutputOracle` does so
low utilisation chains could reduce costs by proposing less often or potentially even modifying `op-proposer` to only
propose if the withdrawals root has changed.

### Air Gap Configuration

There are two relevant configuration options for controlling the timing of games:

* `faultGameMaxClockDuration` the maximum time allowed to challenge a claim. The `Challenger` must respond within this
  time to invalidate an output root.
* `disputeGameFinalityDelaySeconds` the "air gap" time between when a game finalizes and when it can be used for
  withdrawals. The `Guardian` role can blacklist the game prior to or during this period.

On op-mainnet, both of these are set to 84 hours given a total of 1 week delay. With the `L2OutputOracle`
the `Challenger` role has the full 7 days to respond whereas with this setup the `Challenger` would have 3.5 days and
the `Guardian` another 3.5 days. These values can be adjusted per chain however to adjust the time the `Challenger` has
to respond, including setting either to 0.

Since the audits focussed on the `Guardian` override paths and assumed games could resolve incorrectly, it's recommended
to either maintain the op-mainnet equal split or to set the `faultGameMaxClockDuration` to 0
and `disputeGameFinalityDelaySeconds` to 7 days. This effectively eliminates the `Challenger` role but gives
the `Guardian` 7 days to blacklist the game.

Node operators would need to run `op-challenger` to resolve games as `op-proposer` doesn't currently perform that role.
The challenger would run as the `Challenger` role and would then automate countering claims if required.

## Custom Privileged Dispute Game

A simple dispute game could be implemented which allows only the `Proposer` to create a game and be immediately resolved
as valid. The `disputeGameFinalityDelaySeconds` would then be set to 7 days, giving time for the `Guardian` to blacklist
the game if required. The `Challenger` role would not be used in this setup.

This requires some additional development and would potentially need governance approval but would reduce the costs and
complexity of the system as `op-challenger` wouldn't be required to perform game resolution.

# Proposed Solution

Given the long term desire for all chains to utilise fault proofs, utilising the permissioned game for chains that
currently need to is recommended.

There is already work planned to make alt-DA chains fault provable.

## Custom Gas Token

The custom gas token support just needs to be ported over to `OptimismPortal2`. It is intended to be maintained long
term and, once ported, is compatible with fault proofs.

*A high level overview of the proposed solution. When there are multiple alternatives there should be an explanation of
why one solution was picked over other solutions. As a rule of thumb, including code snippets (except for defining an
external API) is likely too low level.*

# Risks & Uncertainties

The main risk is that the increased cost and operational complexity of using the `PermissionedDisputeGame` would be a 
problem for some chains. Delaying upgrading L1 contracts may be an option, though would prevent adoption of future
features that modify the L1 contracts.
