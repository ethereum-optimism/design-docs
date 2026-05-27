# Breaking the Cyclic Dependency: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: L1 Anchor Reorg](#fm1-l1-anchor-reorg)
  - [FM2: Predicted ≠ Deployed L1 Addresses](#fm2-predicted-%E2%89%A0-deployed-l1-addresses)
  - [FM3: Compromised L1 RPC](#fm3-compromised-l1-rpc)
  - [FM4: Build Service Returns Wrong Prestate Hash](#fm4-build-service-returns-wrong-prestate-hash)
  - [FM5: Build Service Unavailability](#fm5-build-service-unavailability)
  - [FM6: Genesis Timestamp Overrun](#fm6-genesis-timestamp-overrun)
- [Audit Requirements](#audit-requirements)
- [Appendix](#appendix)
  - [Appendix A: CREATE2 Address Prediction](#appendix-a-create2-address-prediction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                    |
| ------------------ | ---------------------------------- |
| Author             | _Author Name_                      |
| Created at         | _YYYY-MM-DD_                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_ |
| Need Approval From | _Security Reviewer Name_           |
| Status             | _Draft_                            |

## Introduction

The Predictive Chain Deployments feature reorders `op-deployer`'s pipeline to compute `startingAnchorRoot` and `absolutePrestate` off-chain before calling `OPCM.deploy()`. This enables the permissionless dispute game from L2 block 0.

The pipeline introduces two new external dependencies on the critical path before any transaction lands on L1: the L1 RPC (for the `eth_call` dry-run) and the hosted prestate build service. The failure modes below cover where those dependencies and the new sequencing can go wrong.

References:

- [design.md](./pcd-design.md)

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
  2. Post-deploy validation compares predicted vs. deployed addresses.
- **Detection:** Post-deploy validation catches any mismatch immediately after deployment.
- **Recovery Path(s):** Full redeployment using the correct sender address.

### FM3: Compromised L1 RPC

- **Description:** A compromised L1 RPC returns fabricated results for the `eth_call` dry-run. The genesis is built against those false addresses. The mismatch surfaces only after `OPCM.deploy()` when post-deploy validation runs.
- **Risk Assessment:** Low likelihood, high impact. The chain is unusable if the fabricated addresses differ from what OPCM actually deploys.
- **Mitigations:**
  1. Use a trusted or self-hosted L1 RPC endpoint.
  2. Post-deploy validation is the backstop for this failure mode.
- **Detection:** Post-deploy validation compares every address the dry-run returned against what `OPCM.deploy()` actually emitted. A fabricated address produces an immediate mismatch.
- **Recovery Path(s):** Full redeployment against a trusted RPC.

### FM4: Build Service Returns Wrong Prestate Hash

- **Description:** The hosted build service returns an incorrect prestate hash without signaling an error. The deployed dispute game carries the wrong `absolutePrestate` (the agreed starting MIPS machine state for fault proof disputes). Fault proofs fail silently from genesis.
- **Risk Assessment:** Low likelihood, high impact. The fault proof system is broken from block 0 with no on-chain guard to catch it at deploy time.
- **Mitigations:**
  1. Use a trusted build service endpoint.
  2. Before submitting `OPCM.deploy()`, reproduce the prestate build locally and compare the returned hash against the local result.
- **Detection:** After deployment, replay the prestate build locally using the same `genesis.json`, `rollup.json`, and `depsets.json` that were submitted to the build service. Compare the result against the `absolutePrestate` written into the deployed dispute game contract. A mismatch confirms the build service returned a wrong value.
- **Recovery Path(s):** Full redeployment with the correct prestate hash sourced from a verified local build.

### FM5: Build Service Unavailability

- **Description:** The hosted build service is unreachable. The prestate build step cannot complete, blocking deployment.
- **Risk Assessment:** Medium likelihood (external dependency), medium impact. Deployment is blocked; no deployed chain is harmed.
- **Mitigations:**
  1. Configure the build service URL to point to an alternative endpoint.
- **Detection:** The prestate build step returns a connection error. `op-deployer` exits with a failure message identifying the step.
- **Recovery Path(s):** Retry once the build service is available. Each pipeline step checks whether its output already exists in state before running, so a retry resumes from the failed step without re-executing earlier stages.

### FM6: Genesis Timestamp Overrun

- **Description:** Steps 3–8 take longer than `X` (the configured offset between the anchor timestamp and `genesis_time`). `genesis_time` is already in the past when `OPCM.deploy()` mines. The deployment succeeds with no on-chain guard catching the overrun. `op-node` fills the elapsed gap with empty L2 blocks before user transactions can land.
- **Risk Assessment:** Low likelihood, low impact. A 10-minute overrun produces ~300 empty blocks at the default 2-second `L2BlockTime`. The chain operates correctly; no state is corrupted.
- **Mitigations:**
  1. Set `X` conservatively above the typical end-to-end pipeline runtime, including the prestate build.
  2. Operators on slower hardware should raise `X` via the override flag before deploying.
- **Detection:** `op-node` logs indicate it is behind and filling gap blocks. The gap size is `(mining_time - genesis_time) / L2BlockTime` blocks.
- **Recovery Path(s):** No action required if the empty-block gap is acceptable. For deployments where the gap is unacceptable, redeploy with a larger `X`.

## Audit Requirements

TBD

## Appendix

### Appendix A: CREATE2 Address Prediction

OPCM proxy deployments use CREATE2 with a salt derived from `msg.sender` and the `saltMixer` string in `FullConfig`. Given the same sender, config, and L1 state, the EVM produces the same addresses every time without writing to state. This is what makes the `eth_call` dry-run reliable: it runs the full deployment logic against current L1 state, returns the resulting `ChainContracts` struct, and leaves no trace on-chain.

A dry-run from a different `from` address produces a different CREATE2 salt and different addresses. Any genesis built from those addresses will be invalid the moment the real deployment lands.
