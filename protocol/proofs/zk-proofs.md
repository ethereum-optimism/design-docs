# ZK Proofs: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Paul Dowman                                      |
| Created at         | 2025-11-12                                       |
| Initial Reviewers  | _TBD_                 |
| Need Approval From | _TBD_                                    |
| Status             | Draft |

## Purpose

There are two motivations to move to ZK proofs:
1. Faster withdrawals (reduces the cost of capital for market makers and liquidity bridges).
2. Eventually removing the need to maintain a custom VM (Cannon) *if* we deprecate the fault dispute game.

## Summary

We would like to support ZK proofs as an option for the OP Stack. The optimistic fault dispute game has served us well but there are now several ZK VMs available that are widely used and well tested, and there are at least two full proof systems that work with the OP Stack: [Kailua](https://boundless-xyz.github.io/kailua/) (RISC Zero / Boundless) and [OP Succinct](https://succinctlabs.github.io/op-succinct/). 

This proposal is to use one of those as the starting point, but bring it into the Optimism contracts repo, and evolve it into a generic ZK proof for the OP Stack, that will support multiple ZK provers (as options initially, not as a multi-proof system in this proposal).


## Problem Statement + Context

### Problem 1: Withdrawal Delays

Currently withdrawals from OP Stack chains take approximately 7 days in the normal case. This withdrawal delay increases the cost of capital for market makers and other liquidity providers such as bridges. For example if there’s a large price movement somewhere else, market makers are unable to quickly move capital through the canonical bridge for an OP Stack chain. And liquidity provider bridges must reserve a larger pool of capital.

The withdrawal delay is caused by two factors:
- The game resolution time `MAX_CLOCK_DURATION` (usually 3.5 days), sometimes referred to as the challenger period, the amount of time that we wait for a challenger to respond.
- The finality delay, sometimes referred to as the "air gap", `DISPUTE_GAME_FINALITY_DELAY_SECONDS` (3.5d), the amount of time before the game’s claim is considered valid by the portal.

There would still need to be a finality delay (until we add a mechanism like a second proof to remove or reduce it), but the game would be resolved immediately by posting a ZK proof. (So game resolution time would be reduced to the amount of time needed to generate the proof.)


### Problem 2: Cannon Maintenance Costs and MIPS Architecture Risk

Cannon is a lot of effort to maintain. We at least need to implement new system calls every time there’s a Go compiler update. 

And the MIPS architecture has poor compiler support. Go 1.25 had a MIPS bug that wasn't fixed until Go 1.25.4, and it’s “tier 3” for Rust, which means they rely on community contributions.


## Proposed Solution

We propose to use one of the existing ZK proofs systems with only minor modifications to serve as a foundation for an OP Stack ZK proof system that supports multiple provers and verifiers.

OP Succinct seems to be the best candidate because it's more minimal. Kailua has some great features such as the ability to prove a smaller number of blocks for optimistic proposals, and a bond structure that's more capital efficient and solves the risk of attackers creating an overwhelming number of invalid proposals. However, OP Succinct's more minimal approach leaves room to solve these outside of the dispute game implementation itself, which may end up being a more desirable design in the long run.

Because ZK proving costs are still hard to estimate we would use the optimistic version ("OP Succinct Lite"), which still allows the option of eagerly posting ZK proofs for fast finality.

The steps would be:
1. Add OP Succinct Lite contracts to the Optimism repo, renamed as a generic ZK dispute game.
2. Apply minor modifications:
    - Refactor to apply the [same deployment pattern as `FaultDisputeGame`](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/proofs/dispute-game-creators.md).
    - Remove `AccessManager` to only support permissionless proposals.
    - Add configuration to use an immutable verifier contract at a specific version, rather than the upgradeable verifier gateway contract.
3. Use OPCM for deployment.

An immediate next phase would integrate the Risc Zero VM and Boundless prover network into the same contracts as an alternative proof.

A future phase could implement an improved bond system like Kailua.


### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
