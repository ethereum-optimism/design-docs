<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[Name of Failure Mode 1]](#name-of-failure-mode-1)
  - [[Name of Failure Mode 2]](#name-of-failure-mode-2)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)
  - [Appendix A: This is a Placeholder Title](#appendix-a-this-is-a-placeholder-title)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Stage 1.4: Kona and Asterisc FMA (Failure Modes and Recovery Path Analysis)

| | |
|--------|--------------|
| Author | Andreas Bigger |
| Created at | 2024-10-17 |
| Initial Reviewers | Matt Solomon |
| Need Approval From | Ben Clabby, Mark Tyneway |
| Status | Draft |


## Introduction

This document covers the initial deployment of [the Asterisc Fault Proof VM][asterisc] with [the Kona Fault Proof Program][kona].

Together, Kona and Asterisc form an alternative proof stack to the [op-program] and [cannon].
A secondary dispute game will be created that uses kona and asterisc.
The [op-challenger] supports playing _both_ game types - `op-program` + `cannon` as well as `kona` + `asterisc`.

Stage 1.4 ships these two new net components: [Kona Fault Proof Program][kona] and the [Asterisc Fault Proof VM][asterisc].
Both of these introduce points of failure that are covered in the Failure Modes and Recovery Paths below.

Stage 1.4 also requires the integration of the Kona + Asterisc Dispute Game into the [op-challenger] offchain
fault proof component. The Failure Modes and Recovery Paths below detail this as well.

Below are references for this project:

- ["Stage 1.4" internal project document](https://www.notion.so/oplabs/Stage-1-4-Partway-a6f57ad777b148dda01488f2646cff17)
- [Kona Documentation](https://anton-rs.github.io/kona/).
- [Kona Repository](https://github.com/anton-rs/kona).
- [Asterisc Repository](https://github.com/ethereum-optimism/asterisc)

[kona]: https://github.com/anton-rs/kona
[asterisc]: https://github.com/ethereum-optimism/asterisc
[cannon]: https://github.com/ethereum-optimism/optimism/tree/develop/cannon
[op-program]: https://github.com/ethereum-optimism/optimism/tree/develop/op-program


## Failure Modes and Recovery Paths

### Kona and Asterisc Hot Path Failure

- **Description:**
  The `op-program` + `cannon` dispute game breaks and the `kona` + `asterisc` fault dispute game is placed in the hot path.
  If `kona` + `asterisc` also fail, the permissioned fallback will need to be activated.

- **Risk Assessment:**
  Low likelihood, high impact.
  Low likelihood since `op-program` + `cannon` are already running in production and have meaningful improvements on the way that will increase resilience.
  If `kona` + `asterisc` fails while in the hotpath, the impact is high since the permissioned fallback will need to be triggered.
  This engages the security council, requires comms, and causes the chain to fallback to a weaker security model. 

- **Mitigations:** 
  Offline testing with the VM Runner for both `kona` + `asterisc` as well as `op-program` + `cannon`.
  The VM Runner also fully tests the integration of the `op-challenger` with these dispute games ([covered in a later section](#-Breaking-Backwards-Compatibility-for-the-op-challenger)).
  Action tests ensuring both proof systems follow spec.

- **Detection:**
  The game that broke `op-program` + `cannon` would be invalidated since the respected game type would be switched in the dispute game factory.
  Once `kona` + `asterisc` is the respected game type, the dispute monitor will watch for all new games and will alert if there is an issue with `kona` + `asterisc`.
  If `kona` + `asterisc` has an issue while in the hot path, and an invalid output / game is played, the dispute monitor and `op-challenger` would alert and detect invalid games.
  At this point, the permissioned fallback would need to be engaged since both proof systems are in repair.

- **Recovery Path(s):**
  Likely the only viable recovery path would be the permissioned fallback unless the `op-program` + `cannon` are fixed and able to be placed back into the hotpath.


### Contract Upgrades

- **Description:**
  The deployment of a new `FaultDisputeGame` instance, `RISCV` VM, and `DelayedWETH` is needed.
  Two upgrade actions are performed.
  1. Initialize anchor state for new game type in `AnchorStateRegistry`.
  2. Set game type of `kona` + `asterisc` to the FDG deployed before the upgrade.
  The deployment failing is recoverable by simply re-deploying.
  The upgrade touches components in the hot path so it is a failure mode covered here.

- **Risk Assessment:**
  Low likelihood, medium impact.
  The contract upgrades are tested and very simple.
  Contracts in the hot path are audited and have been in production so are unlikely to have critical bugs.
  // TODO: impact

- **Mitigations:** 

- **Detection:**

- **Recovery Path(s):**


### Divergent Kona Derivation Pipeline

- **Description:**
  The Kona derivation pipeline diverged in behavior from the op-program's derivation pipeline.

- **Risk Assessment:**
  Low likelihood, low impact.
  Low likelihood as the derivation pipeline is extensively unit tested, run in the VM Runner, and is action tested.
  Low impact since by default `kona` + `asterisc` are placed in the cold path.
  [The "Kona and Asterisc Hot Path Failure"](#-Kona-and-Asterisc-Hot-Path-Failure) above covers the failure mode for `kona` + `asterisc` when placed in the hot path.

- **Mitigations:**
  Frequent action tests covering the derivation pipeline.
  VM Runner exectutes the derivation pipeline as part of the kona fault proof program running ontop of asterisc.
  [Extensive test coverage](https://app.codecov.io/gh/anton-rs/kona/tree/main/crates) of kona.

- **Detection:**
  The VM runner would pick up an issue.
  The action tests or local testing may pick up an issue.
  Use of the derivation elsewhere, for example in alternative consensus clients like `op-rs`, would pick up an issue.

- **Recovery Path(s):**
  Fix the derivation pipeline divergence, release and deploy the updated kona + asterisc when in the coldpath.
  If `kona` + `asterisc` is placed in the hotpath, either the `op-program` + `cannon` would need to be falled back on otherwise fallback to the permissed dispute game.
  [The "Kona and Asterisc Hot Path Failure"](#-Kona-and-Asterisc-Hot-Path-Failure) above covers the failure mode for `kona` + `asterisc` when placed in the hot path.


### Kona Execution Diverges from op-program

- **Description:**
  The Kona fault proof program executes blocks in a stateless L2 block executor that uses optimism-revm as a backend.
  If optimism-revm or the L2 block executor diverge in behavior from the op-program, the kona + asterisc proof system will be invalid.

- **Risk Assessment:**
  Low likelihood, low impact.
  A small divergence in execution behavior can cause invalid game results.
  But since kona + asterisc is in the coldpath, it doesn't have impact on anything in the production system.

- **Mitigations:**
  State testing against op-revm.
  Frequent action tests against kona, which covers the L2 block executor.
  `op-reth` which uses the same op-execution environment as kona.

- **Detection:**
  Since kona + asterisc is in the coldpath by default, a divergence would only be seen by action tests or local testing.

- **Recovery Path(s):**
  If kona + asterisc is in the hotpath, the game would need to be invalidated and op-program + cannon placed in the hotpath while kona + asterisc is fixed.
  Once the execution fixes are released and deployed, kona + asterisc could be placed back in the hotpath.
  If the game is in the coldpath, it would need to be fixed and re-deployed.


### Breaking Backwards Compatibility for the op-challenger

- **Description:**
  The `op-challenger` has been updated to support the Kona + Asterisc Fault Dispute Game type ([Game Type 3]).

- **Risk Assessment:**
  Low likelihood as the action tests that are run in CI very frequently run the `op-challenger` with the kona + asterisc fault dispute game.
  Medium impact since the off-chain `op-challenger` service can be updated with enough buffer time due to the large game time and nature of the chess clock.

- **Mitigations:**
  The `op-challenger` is architected in such a way as to isolate game types and moreso the inididual game players.
  To allow the challenger to scale and play many different game types at once, a list of game types to play is passed into the `op-challenger` as a CLI flag.
  When games are detected on-chain, the `op-challenger` will create a "game player" for this game type.
  Players are run in individual threads so a breaking kona-asterisc game player in the challenger will not cause a liveliness issue for the `op-challenger` to play other game types.
  Another key mitigation is the VM runner. This is an offline simulation that runs asterisc + kona with the op-challenger. It's been running since around August 2024.

- **Detection:**
  The Dispute Monitor ([op-dispute-mon] service provides an isolated monitoring service that will detect if a game is not being played by the honest `op-challenger`. It will alert in this case.

- **Recovery Path(s):**
  If the `op-challenger` fails to play Kona + Asterisc fault dispute games, a fix will need to be made to the `op-challenger`.
  It does not require governance, but would need a new release of the `op-challenger` to be published and rolled out to infrastructure.

[op-dispute-mon]: https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon
[Game Type 3]: https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/types/types.go#L30


### Divergence between RISCV.sol and Asterisc

- **Description:**
  Asterisc is a "Fault Proof VM".
  It is an onchain RISCV instruction emulator with the on-chain component of the kona + asterisc fault dispute game being the `RISCV.sol` contract.
  `RISCV.sol` runs a single RISCV instruction on chain with merkleized VM memory.
  Effectively, the behavior between `RISCV.sol` and asterisc cannot diverge.

- **Risk Assessment:**
  Low likelihood, but high impact.

- **Mitigation:**
  Heavily test `RISCV.sol`. Audit `RISCV.sol`.
  Asterisc already runs through action tests frequently in CI validiting that it produces the expected output.

- **Detection:**
  Since `op-dispute-mon` uses the same asterisc backend that the `op-challenger` does, there may be divergent behaviour between asterisc and `RISCV.sol`.
  In the worst case, if a silent issue is exploited in `RISCV.sol`, a fault dispute game may resolve incorrectly.
  But since the kona + asterisc fault dispute game lives outside the hotpath, this game result is not used anywhere by the portal and can be discarded.

- **Recovery Path(s):**
  If a divergence between `RISCV.sol` and asterisc is found, either or both would need to be updated, and re-released.
  Since the kona + asterisc Fault Dispute Game would need to use the updated `RISCV.sol`, if it was fixed, an upgrade would need to be performed.
  If the kona + asterisc fault dispute game sits in the hotpath, the security council would need to fallback to the op-program + cannon fault dispute game, and only switch back once the fix has been released and deployed.


### Generic items we need to take into account:

- [X] [./failure-modes-analysis.md](./failure-modes-analysis.md).


## Audit Requirements

The `RISCV.sol` contract will need an audit for mainnet, though no audit is required for testnet.
The audit document will be handled separately from this FMA.


## Appendix

### Appendix A: op-challenger

[op-challenger] contains CLI flags to demonstrate specifying game types it should play.

[op-challenger]: https://github.com/ethereum-optimism/optimism/tree/develop/op-challenger
