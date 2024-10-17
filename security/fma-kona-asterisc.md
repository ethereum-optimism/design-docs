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
| Initial Reviewers | *Reviewer Name 1, Reviewer Name 2* |
| Need Approval From | Ben Clabby, Mark Tyneway |
| Status | Draft |


## Introduction

This document covers the initial deployment of [the Asterisc fault proof VM](https://github.com/ethereum-optimism/asterisc) with [the Kona fault proof program](https://github.com/anton-rs/kona).

Together, Kona and Asterisc form an alternative proof stack to the [op-program](https://github.com/ethereum-optimism/optimism/tree/develop/op-program) and [cannon](https://github.com/ethereum-optimism/optimism/tree/develop/cannon). A secondary dispute game will be created that uses kona and asterisc, with the `op-challenger` supporting playing _both_ game types - `op-program` + `cannon` as well as `kona` + `asterisc`.

Since two main components are being shipped in Stage 1.4 there are two main places the system can break: the kona fault proof program, and the asterisc VM. The below Failure Modes and Recovery Paths go over those situations.

Below are references for this project:

- ["Stage 1.4" internal project document](https://www.notion.so/oplabs/Stage-1-4-Partway-a6f57ad777b148dda01488f2646cff17)
- [Kona Documentation](https://anton-rs.github.io/kona/).
- [Kona Repository](https://github.com/anton-rs/kona).
- [Asterisc Repository](https://github.com/ethereum-optimism/asterisc)


## Failure Modes and Recovery Paths


### Divergent Kona Derivation Pipeline

- **Description:** The Kona derivation pipeline diverged in behavior from the op-program's derivation pipeline.

- **Risk Assessment:** Low likelihood, low impact. A small divergence in derivation can cause invalid game results. But since kona + asterisc is in the coldpath, it doesn't have impact on anything in the production system.

- **Mitigations:** Frequent action tests covering the derivation pipeline.

- **Detection:** Since kona + asterisc is in the coldpath by default, a divergence would only be seen by the action tests or local testing.

- **Recovery Path(s):** Fix the derivation pipeline divergence, release and deploy the updated kona + asterisc when in the coldpath. If kona + asterisc is placed in the hotpath, either the op-program + cannon would need to be falled back on otherwise fallback to the permissed dispute game.


### Kona Execution Diverges from op-program

- **Description:** The Kona fault proof program executes blocks in a stateless L2 block executor that uses optimism-revm as a backend. If optimism-revm or the L2 block executor diverge in behavior from the op-program, the kona + asterisc proof system will be invalid.

- **Risk Assessment:** Low likelihood, low impact. A small divergence in execution behavior can cause invalid game results. But since kona + asterisc is in the coldpath, it doesn't have impact on anything in the production system.

- **Mitigations:** State testing against op-revm. Frequent action tests against kona, which covers the L2 block executor. op-reth which uses the same op-execution environment as kona.

- **Detection:** Since kona + asterisc is in the coldpath by default, a divergence would only be seen by action tests or local testing.

- **Recovery Path(s):** If kona + asterisc is in the hotpath, the game would need to be invalidated and op-program + cannon placed in the hotpath while kona + asterisc is fixed. Once the execution fixes are released and deployed, kona + asterisc could be placed back in the hotpath. If the game is in the coldpath, it would need to be fixed and re-deployed.


### Breaking Backwards Compatibility for the op-challenger

- **Description:** The `op-challenger` has been updated to support the Kona + Asterisc Fault Dispute Game type ([Game Type 3](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/types/types.go#L30)).

- **Risk Assessment:** Low likelihood as the action tests that are run in CI very frequently run the `op-challenger` with the kona + asterisc fault dispute game. Medium impact since the off-chain `op-challenger` service can be updated with enough buffer time due to the large game time and nature of the chess clock.

- **Mitigations:** The `op-challenger` is architected in such a way as to isolate game types and moreso the inididual game players. To allow the challenger to scale and play many different game types at once, a list of game types to play is passed into the `op-challenger` as a CLI flag. When games are detected on-chain, the `op-challenger` will create a "game player" for this game type. Players are run in individual threads so a breaking kona-asterisc game player in the challenger will not cause a liveliness issue for the `op-challenger` to play other game types.

- **Detection:** The Dispute Monitor ([`op-dispute-mon`](https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon)) service provides an isolated monitoring service that will detect if a game is not being played by the honest `op-challenger`. It will alert in this case.

- **Recovery Path(s):** If the `op-challenger` fails to play Kona + Asterisc fault dispute games, a fix will need to be made to the `op-challenger`. It does not require governance, but would need a new release of the `op-challenger` to be published and rolled out to infrastructure.


### Divergence between RISCV.sol and Asterisc

- **Description:** Asterisc is a "Fault Proof VM". It is an onchain RISCV instruction emulator with the on-chain component of the kona + asterisc fault dispute game being the `RISCV.sol` contract. `RISCV.sol` runs a single RISCV instruction on chain with merkleized VM memory. Effectively, the behavior between `RISCV.sol` and asterisc cannot diverge.

- **Risk Assessment:** Low likelihood, but high impact.

- **Mitigation:** Heavily test `RISCV.sol`. Audit `RISCV.sol`. Asterisc already runs through action tests frequently in CI validiting that it produces the expected output.

- **Detection:** Since `op-dispute-mon` uses the same asterisc backend that the `op-challenger` does, there may be divergent behaviour between asterisc and `RISCV.sol`. In the worst case, if a silent issue is exploited in `RISCV.sol`, a fault dispute game may resolve incorrectly. But since the kona + asterisc fault dispute game lives outside the hotpath, this game result is not used anywhere by the portal and can be discarded.

- **Recovery Path(s):** If a divergence between `RISCV.sol` and asterisc is found, either or both would need to be updated, and re-released. Since the kona + asterisc Fault Dispute Game would need to use the updated `RISCV.sol`, if it was fixed, an upgrade would need to be performed. If the kona + asterisc fault dispute game sits in the hotpath, the security council would need to fallback to the op-program + cannon fault dispute game, and only switch back once the fix has been released and deployed.


### Generic items we need to take into account:

- [X] [./failure-modes-analysis.md](./failure-modes-analysis.md).


## Audit Requirements

*Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning.*


## Appendix

### Appendix A: op-challenger

[op-challenger](https://github.com/ethereum-optimism/optimism/tree/develop/op-challenger) contains CLI flags to demonstrate specifying game types it should play.
