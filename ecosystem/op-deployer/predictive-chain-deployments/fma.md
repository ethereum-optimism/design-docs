# Breaking the Cyclic Dependency: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: L1 Anchor Reorg](#fm1-l1-anchor-reorg)
  - [FM2: Predicted ≠ Deployed L1 Addresses](#fm2-predicted-%E2%89%A0-deployed-l1-addresses)
  - [FM3: Compromised L1 RPC](#fm3-compromised-l1-rpc)
  - [FM4: Wrong Prestate Hash](#fm4-wrong-prestate-hash)
  - [FM5: Genesis Timestamp Overrun](#fm5-genesis-timestamp-overrun)
  - [FM6: Stale Prestate After Re-run of `prepare`](#fm6-stale-prestate-after-re-run-of-prepare)
  - [FM7: Wrong startingAnchorRoot](#fm7-wrong-startinganchorroot)
- [Audit Requirements](#audit-requirements)
- [Appendix](#appendix)
  - [Appendix A: CREATE2 Address Prediction](#appendix-a-create2-address-prediction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                    |
| ------------------ | ---------------------------------- |
| Author             | Joxess, 0xOneTony, 0xiamflux       |
| Created at         | _2026-05-26_                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_ |
| Need Approval From | _Security Reviewer Name_           |
| Status             | _Draft_                            |

## Introduction

The Predictive Chain Deployments feature reorders `op-deployer`'s pipeline to compute two values off-chain before calling `OPCM.deploy()`: the `startingAnchorRoot` (the dispute system's starting anchor) and the `absolutePrestate` (the fault proof's agreed starting state). This makes the permissionless dispute game valid from L2 block 0.

The new risk comes from deriving those values off-chain. One external dependency now sits on the critical path before any transaction lands on L1: the L1 RPC, used for the `eth_call` dry-run. The failure modes below cover that dependency and the new sequencing.

References:

- [PCD design](./pcd-design.md)

## Failure Modes and Recovery Paths

### FM1: L1 Anchor Reorg

- **Description:** Ethereum reorgs past the anchor block after `op-deployer` picks it. The anchor hash is baked into `rollup.json` and the prestate. A reorg invalidates both, leaving the chain unable to start.
- **Risk Assessment:** Low likelihood, high impact. The anchor is picked at the `safe` tag, which requires one epoch (~6 minutes) to reach. Reorgs past the safe tag are rare on Ethereum mainnet.
- **Mitigations:**
  1. Only blocks at the `safe` tag or deeper are eligible as the anchor.
  2. A `BLOCKHASH` opcode check in the deploy transaction reverts if the anchor block no longer exists at deployment time.
- **Detection:** If the reorg occurs before `OPCM.deploy()`: the `BLOCKHASH` opcode check in the deploy transaction reverts, aborting the deployment. If the reorg occurs after `OPCM.deploy()` mines: `op-node` fails to initialize because the anchor block hash in `rollup.json` no longer exists in the canonical L1 chain.
- **Recovery Path(s):** If the reorg occurs before `OPCM.deploy()`, abort and restart with a new anchor block. If `OPCM.deploy()` already mined with an invalid anchor, full redeployment is required.

### FM2: Predicted ≠ Deployed L1 Addresses

- **Description:** The `eth_call` dry-run and the real `OPCM.deploy()` emit different L1 contract addresses. The L2 genesis was built against the predicted addresses, so any mismatch makes the chain unsafe to operate.
- **Risk Assessment:** Low likelihood, high impact. CREATE2 is deterministic given the same `msg.sender` and `saltMixer`. The most likely cause is a dry-run sent from a different address than the real broadcast.
- **Mitigations:**
  1. The `eth_call` dry-run must use the same `from` address as the real `OPCM.deploy()` broadcast.
  2. A pre-broadcast preflight in `continue` re-checks the predicted addresses against current L1 state before the actual deployment.
  3. Post-deploy validation compares predicted vs. deployed addresses.
- **Detection:** The pre-broadcast preflight in `continue` re-runs the dry-run and aborts before the deploy if the predicted addresses no longer match the committed genesis. Post-deploy validation is the backstop.
- **Recovery Path(s):** If the preflight catches the mismatch, fix the sender address and re-run `continue`; nothing was deployed. If it is caught only post-deploy, full redeployment with the correct sender address is required.

### FM3: Compromised L1 RPC

- **Description:** A compromised L1 RPC returns fabricated results for the `eth_call` dry-run. The genesis is built against those false addresses. The mismatch surfaces only after `OPCM.deploy()` when post-deploy validation runs.
- **Risk Assessment:** Low likelihood, high impact. The chain is unusable if the fabricated addresses differ from what OPCM actually deploys.
- **Mitigations:**
  1. Use a trusted or self-hosted L1 RPC endpoint.
  2. Post-deploy validation is the backstop for this failure mode.
- **Detection:** Post-deploy validation compares every address the dry-run returned against what `OPCM.deploy()` actually emitted. A fabricated address produces an immediate mismatch.
- **Recovery Path(s):** Full redeployment against a trusted RPC.

### FM4: Wrong Prestate Hash

- **Description:** The prestate hash written to the state is incorrect, either a bad hash passed through the command flag or a bad override resolved from the intent. The deployed dispute game carries the wrong `absolutePrestate` (the agreed starting MIPS machine state for fault proof disputes). Fault proofs fail silently from genesis.
- **Risk Assessment:** Low likelihood, high impact. The fault proof system is broken from block 0 with no on-chain guard to catch it at deploy time.
- **Mitigations:**
  1. Source any override from a trusted, reproducible build.
  2. Before running `continue`, the reviewer should reproduce the prestate build locally, and compare the result against the value in the state.
- **Detection:** Replay the prestate build using the same `genesis.json`, `rollup.json`, and `depsets.json` from `prepare`. Compare the result against the `absolutePrestate` in the state, and after deployment against the value in the deployed dispute game contract. A mismatch confirms a wrong hash.
- **Recovery Path(s):** Full redeployment with the correct prestate hash sourced from a verified build.

### FM5: Genesis Timestamp Overrun

- **Description:** Steps 3–8 take longer than `X` (the configured offset between the anchor timestamp and `genesis_time`). `genesis_time` is already in the past when `OPCM.deploy()` mines. The deployment succeeds with no on-chain guard catching the overrun. `op-node` fills the elapsed gap with empty L2 blocks before user transactions can land.
- **Risk Assessment:** Low likelihood, low impact. A 10-minute overrun produces ~300 empty blocks at the default 2-second `L2BlockTime`. The chain operates correctly. No state is corrupted.
- **Mitigations:**
  1. Set `X` conservatively above the typical end-to-end pipeline runtime, including the prestate build.
  2. Operators on slower hardware should raise `X` via the override flag before deploying.
- **Detection:** `op-node` logs indicate it is behind and filling gap blocks. The gap size is `(mining_time - genesis_time) / L2BlockTime` blocks.
- **Recovery Path(s):** No action required if the empty-block gap is acceptable. For deployments where the gap is unacceptable, redeploy with a larger `X`.

### FM6: Stale Prestate After Re-run of `prepare`

- **Description:** `prepare` is re-run, for example to recover a stuck deployment or after a reorg, and re-picks a fresh anchor. That produces a new `genesis_time` and therefore new `genesis.json` and `rollup.json`. If a prestate built against the previous `genesis_time` is reused instead of rebuilt, the deployed `absolutePrestate` does not match the committed `rollup.json`. Fault proofs are broken from genesis.
- **Risk Assessment:** Low likelihood, high impact. Requires `prepare` to be re-run with a new `genesis_time` while a prestate from the prior run is still present. Impact is high as fault proofs are broken from block 0.
- **Mitigations:**
  1. Re-running `prepare` with a fresh anchor invalidates the prestate in the state, forcing a rebuild before `continue` proceeds.
  2. The prestate is built from the committed `rollup.json`, so rebuilding after any `genesis_time` change keeps the prestate and the deployed artifacts consistent.
- **Detection:** Reproduce the prestate from the current committed `genesis.json`, `rollup.json`, and `depsets.json` and compare against the `absolutePrestate` in the state. A mismatch means the prestate is stale relative to the current artifacts.
- **Recovery Path(s):** Rebuild the prestate from the current artifacts and re-commit the hash before running `continue`.

### FM7: Wrong startingAnchorRoot

- **Description:** The `startingAnchorRoot` committed to the state is incorrect. `OPCM.deploy()` seeds `AnchorStateRegistry` with this anchor, and every dispute game bootstraps its proof from it. Because the anchor does not match the real genesis, the program is told to start from a state that never existed, so honest proposals that descend from the real genesis cannot be proven and fault proofs are broken from block 0.
- **Risk Assessment:** Low likelihood, high impact. The output root is computed deterministically from the genesis block in the `op-deployer` pipeline which makes the most probable causes are either a derivation bug or a wrong value written in state files.
- **Mitigations:**
  1. Before running `continue`, the reviewer should recompute `outputRoot = keccak256(version || stateRoot || messagePasserStorageRoot || blockHash)` from the committed genesis block and compare against the `startingAnchorRoot` in the state.
- **Detection:** Recompute the `startingAnchorRoot` from the same values the pipeline computes it from. Post-deploy validation reads the anchor from `AnchorStateRegistry.getStartingAnchorRoot()` and compares it against recomputed anchor root for the genesis block. A mismatch confirms a wrong anchor root.
- **Recovery Path(s):** Depends on where in the pipeline the bad anchor is caught:
  1. **Before `OPCM.deploy()`.** A reviewer recomputing the `startingAnchorRoot` flags the mismatch while the anchor still lives only in the configuration artifacts. Fix the value and continue the pipeline.
  2. **During post-deploy validation** Full redeployment is required with the correct anchor root.

## Audit Requirements

TBD

## Appendix

### Appendix A: CREATE2 Address Prediction

OPCM proxy deployments use CREATE2 with a salt derived from `msg.sender` and the `saltMixer` string in `FullConfig`. Given the same sender, config, and L1 state, the EVM produces the same addresses every time without writing to state. This is what makes the `eth_call` dry-run reliable: it runs the full deployment logic against current L1 state, returns the resulting `ChainContracts` struct, and leaves no trace on-chain.

A dry-run from a different `from` address produces a different CREATE2 salt and different addresses. Any genesis built from those addresses will be invalid the moment the real deployment lands.
