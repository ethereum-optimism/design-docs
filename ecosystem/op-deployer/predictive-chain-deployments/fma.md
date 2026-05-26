# Breaking the Cyclic Dependency: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: L1 Anchor Reorg](#fm1-l1-anchor-reorg)
  - [FM2: Predicted ≠ Deployed L1 Addresses](#fm2-predicted-%E2%89%A0-deployed-l1-addresses)
  - [FM3: Compromised L1 RPC](#fm3-compromised-l1-rpc)
  - [FM4: Build Service Bad Behavior](#fm4-build-service-bad-behavior)
  - [FM5: Build Service Unavailability](#fm5-build-service-unavailability)
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

This document covers the failure modes for the Predictive Chain Deployments feature in `op-deployer`. The feature reorders the deployment pipeline to compute `startingAnchorRoot` and `absolutePrestate` off-chain before calling `OPCM.deploy()`, enabling the permissionless dispute game to be active from L2 block 0.

References:

- [design.md](./pcd-design.md)

## Failure Modes and Recovery Paths

### FM1: L1 Anchor Reorg

- **Description:** If Ethereum reorgs past the anchor block after `op-deployer` picks it, the chain will be unable to start. The anchor's hash is baked into `rollup.json` and the prestate, so a reorg invalidates both.
- **Risk Assessment:** Low likelihood (anchor is picked at the `safe` tag, which requires one epoch ~6 minutes to finalize), high impact (requires full redeployment).
- **Mitigations:**
  1. Only `safe`-tag blocks or deeper are eligible as the anchor block.
  2. A `BLOCKHASH` opcode check in the deploy transaction can cause it to revert if the anchor block has been reorged out.
- **Detection:** Post-deploy validation confirms deployed L1 addresses match predicted addresses; a mismatch indicates the anchor was invalid.
- **Recovery Path(s):** If the reorg occurs before `OPCM.deploy()`, abort the run and restart with a new anchor block. If `OPCM.deploy()` has already been mined with an invalid anchor, full redeployment is required.

### FM2: Predicted ≠ Deployed L1 Addresses

- **Description:** Mismatch between `eth_call` dry-run and real `OPCM.deploy()` emissions. The L2 genesis was built against the predicted addresses, making the deployed chain unsafe to operate.
- **Risk Assessment:** Low likelihood (CREATE2 is deterministic given same `msg.sender` and salt), high impact (chain is unusable).
- **Mitigations:**
  1. The `eth_call` dry-run must be sent `from` the same address that broadcasts the real `OPCM.deploy()`.
  2. Post-deploy validation compares predicted vs. deployed addresses.
- **Detection:** Post-deploy validation will catch any mismatch immediately after deployment.
- **Recovery Path(s):** Full redeployment with the correct sender address.

### FM3: Compromised L1 RPC

- **Description:** A compromised L1 RPC endpoint might return false chain contracts in response to the `eth_call` dry-run, causing the genesis to be built against incorrect predicted addresses.
- **Risk Assessment:** Low likelihood, high impact (chain is unusable; may not be detected until post-deploy validation).
- **Mitigations:**
  1. Operators should use trusted or self-hosted L1 RPC endpoints.
  2. Post-deploy validation will surface the mismatch.
- **Detection:** Post-deploy validation (Step 10).
- **Recovery Path(s):** Full redeployment against a trusted RPC.

### FM4: Build Service Bad Behavior

- **Description:** The hosted build service returns an incorrect prestate hash as a silent failure, causing `absolutePrestate` in the deployed dispute game to be wrong. Fault proofs would fail silently.
- **Risk Assessment:** Low likelihood, high impact (fault proof system is broken from genesis).
- **Mitigations:**
  1. Operators should use a trusted build service endpoint.
  2. Independently verify the prestate hash against a local build.
- **Detection:** Fault proof disputes will fail to resolve correctly. No automated detection is defined in this milestone.
- **Recovery Path(s):** Requires redeployment with the correct prestate hash.

### FM5: Build Service Unavailability

- **Description:** The hosted build service is unreachable. Step 8 cannot be completed, blocking the full permissionless deployment path.
- **Risk Assessment:** Medium likelihood (external dependency), medium impact (deployment is blocked but no chain is harmed).
- **Mitigations:**
  1. Operators can configure `--op-program-svc-url` to point to an alternative endpoint.
- **Detection:** Step 8 returns an error; `op-deployer` exits with a failure message.
- **Recovery Path(s):** Retry once the build service is available. The pipeline's `should`\* idempotency pattern allows safe retry from the failed step.

## Audit Requirements

## Appendix

### Appendix A: CREATE2 Address Prediction

The `eth_call` dry-run of `OPCM.deploy()` works because OPCM proxy deployments use CREATE2 with a salt that mixes in `msg.sender` and a `saltMixer` string from `FullConfig`. Given identical inputs (same sender, same config, same L1 state), the EVM deterministically produces the same addresses without writing to state. The dry-run must therefore use the same `from` address as the real broadcast, or the predicted addresses will differ.
