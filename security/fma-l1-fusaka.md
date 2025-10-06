# Upgrade 16b (L1 Fusaka Defense): Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Generic Items](#generic-items)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |                     |
| ------------------- | ------------------- |
| Author              | George Knee         |
| Created at          | 2nd October 2025    |
| Needs Approval From |                     |
| Other Reviewers     |                     |
| Status              | DRAFT               |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers Upgrade 16b, which involves the following changes:

- (`op-batcher`) Ability to submit cell proof blob sidecars (not just blob proof sidecars)
- (Consensus Layer) Ability to compute an accurate reflection of the L1 Blob Base Fee on L2, even when L1 activates Blob Parameter Only forks.
- (Smart Contracts) For chains using Fault Proofs, L1 contracts updates which reference an updated absolute prestate hash.

Each change has its own section below with a list of Failure Modes.

Below are references for this project:

- [Github tracker](https://github.com/ethereum-optimism/optimism/issues/17471)
- [Specs clarification](https://github.com/ethereum-optimism/specs/pull/790)
- [OP Contracts Manager specs](https://specs.optimism.io/experimental/op-contracts-manager.html?highlight=opcm#op-contracts-manager)
- [Optimism Docs on Pectra (including L1 activation times)](https://oplabs.notion.site/fusaka-upgrade-readiness-doc)

## Failure Modes and Recovery Paths

### FM1: `op-batcher` release not rolled out in time

- **Description:**  
  There is a chance we don't ship the changes we need to `op-batcher` before the forks activate on L1.  
  Also a problem if node or chain operators fail to update in time, even if our releases are available.  
  The impact would be a safe head stall and sequencer window expiry / reorg if the L1 rejects the batcher's blob transactions.  
  This can happen even in the patched batcher, if a config flag is not flipped.

- **Risk Assessment:** High severity, Medium Likelihood
- **Mitigations:**
  - Get a basic release candidate ready as early as possible
  - Communicate widely and clearly about the need to update and any live ops that need to happen
  - Check with L1 nodes on already forked networks: do they accept legacy batcher blob transactions?
  
- **Detection:**
  - Usual alerts and dashboards
- **Recovery Path(s):** 
  - Even an old batcher can switch to calldata as a good mitigation

---

### FM2a: `op-node` or `kona` node release not rolled out in time (blob verification)

- **Description:**  
  There is a chance we don't ship the changes we need to `op-node` or `kona` before the forks activate on L1.  
  Changes to the beacon API https://ethereum.github.io/beacon-APIs/?urls.primaryName=dev are breaking with respect
  to the current op-node, op-program and kona. Both rely on the `/eth/v1/beacon/blob_sidecars` API which is being
  deprecated. It will be replaced with `/eth/v1/beacon/blobs`, which has the notable difference that it does 
  not return KZG commitments or KZG blob proofs (see [EIP 4844](https://eips.ethereum.org/EIPS/eip-4844)). The
  affected components will be updated to i) pull blobs from the new endpoint once it is available and ii) perform
  blob verification _without_ the aid of a KZG proof. If either fix fails to deploy in time there would be a
  safe head stall and possible eventual sequencer window expiry.

- **Risk Assessment:** Medium severity, Medium Likelihood  
- **Mitigations:**
  - Prepare release candidate early and test on L1-Fusaka-enabled devnets
  - Communicate clearly about upgrade requirements
  
- **Detection:**
  - Usual monitoring / alerts
- **Recovery Path(s):** 
  - Switch the batcher to use calldata

### FM2c: `op-node` or `kona` node release not rolled out in time (blob base fee calculation)

- **Description:**  
  There is a chance we don't ship the changes we need to `op-node` or `kona` before the forks activate on L1.  
  For example, the code we need to patch `op-node` exists in go-ethereum, but is only released late in the development cycle.  
  Also a problem if node or chain operators fail to update in time, even if our releases are available.  

  The impact would be:
  - Incorrect blob base fee → L2 users overcharged
  - Possible denial of service due to high L2 fees
  - Chain split if nodes update uncoordinatedly

  We have seen this failure mode [play out on Sepolia](https://www.notion.so/oplabs/OP-Sepolia-L1-Cost-Blob-Schedule-Bug-1adf153ee1628009b429ee10828fb300#1adf153ee1628009b429ee10828fb300), requiring a hardfork to fix.  

  With the current blob parameter schedule, only BPO1 causes the problem, not Fusaka itself.

- **Risk Assessment:** High severity, Medium Likelihood  
- **Mitigations:**
  - Prepare release candidate early
  - Communicate clearly about upgrade requirements
  - Take extra care in the *next* upstream merge of `op-geth` (post-Fusaka) for consensus changes
  
- **Detection:**
  - Obvious if releases aren’t shipped on time
- **Recovery Path(s):** 
  - One-off fork may be needed to fix chain split

---

### FM3: Lagging behind upstream Geth

- **Description:**  
  We decided to delay updating `op-geth` with Fusaka changes, to avoid forced Go 1.24 adoption.  
  This is lower risk since L2 does not adopt Fusaka directly. However, we may miss bug/security patches upstream.  

- **Risk Assessment:** High severity, Low Likelihood
- **Mitigations:**
  - Thoroughly review the skipped upstream merge for critical patches
  - Take extra care in the next merge post-Fusaka  
- **Detection:**
  - Immunefi report or an exploit
- **Recovery Path(s):**
  - Cherry-pick missing fixes into `op-geth`

---

### FM4: New chain deployments consume more than 16,777,216 gas

- **Description:**  
 EIP [7825](https://eips.ethereum.org/EIPS/eip-7825) introduces a
 per-transaction gas cap, which our standard deployment 
 might well exceed (especially with more dispute game types being deployed).

- **Risk Assessment:** Low severity, High Likelihood
- **Mitigations:**
  - Deploy before Fusaka goes live (only works for a short time)
  - Avoid expanding the set of standard contracts deployed
  - Tune the "padding" we apply in `op-deployer` (see https://github.com/ethereum-optimism/optimism/pull/17710)
  - Rearchitect the deployment script to break up the transaction into several smaller transactions
- **Detection:**
  - Obvious, deployment transaction reverts
- **Recovery Path(s):**
  - N/A

---


## Generic Items

This upgrade is an unnamed L2 hardfork, activated by an L1 hardfork: so the items in [fma-generic-hardfork.md](./fma-generic-hardfork.md) apply:

- Chain Halt at activation  
- Invalid absolute prestate provided.  
- Chain split across clients  

- [x] Confirm these items have been considered and updated if necessary.

---

## Audit Requirements

none

---

## Appendix

| EIP | Description | Compatibility Concerns | Components Affected |
| --- | --- | --- | --- |
| [7594](https://eips.ethereum.org/EIPS/eip-7594) | **PeerDAS (Headline feature)** | Blob Tx senders must compute and attach “cell proofs” | `op-batcher` (tx-mgr), `proyxd` |
| [7642](https://eips.ethereum.org/EIPS/eip-7642) | `eth/69` History expiry & simpler receipts | None |  |
| [7823](https://eips.ethereum.org/EIPS/eip-7823) | Set upper bounds for MODEXP | None |  |
| [7825](https://eips.ethereum.org/EIPS/eip-7825) | Tx Gas Limit Cap | May limit `superchain-ops` on L1 | `superchain-ops` |
| [7883](https://eips.ethereum.org/EIPS/eip-7883) | ModExp Gas Cost Increase | Increases cost of dispute games |  |
| [7892](https://eips.ethereum.org/EIPS/eip-7892) | **Blob Parameter Only Hardforks (Headline feature)** | Framework for blob schedule changes; affects L1 cost; new prestate implied | `op-node` / `op-program`, `kona` |
| [7917](https://eips.ethereum.org/EIPS/eip-7917) | Deterministic proposer lookahead | None (L1 PoS only) |  |
| [7918](https://eips.ethereum.org/EIPS/eip-7918) | Blob base fee bounded by execution cost | No action needed (`excessBlobGas`) |  |
| [7934](https://eips.ethereum.org/EIPS/eip-7934) | RLP Execution Block Size Limit | TBD |  |
| [7935](https://eips.ethereum.org/EIPS/eip-7935) | Default gas limit to XX0M | Backwards compatible |  |
| [7939](https://eips.ethereum.org/EIPS/eip-7939) | CLZ opcode | None |  |
| [7951](https://eips.ethereum.org/EIPS/eip-7951) | Precompile for secp256r1 support | None |  |
| [7910](https://eips.ethereum.org/EIPS/eip-7910) | `eth_config` JSON-RPC method | Non-breaking |  |

---

## Action Items

- [ ] (BLOCKING) Resolve comments and incorporate into doc  
- [ ] (BLOCKING) Add action tests for `op-node` and `kona`  
- [ ] (BLOCKING) Deploy to local multi-client devnet (op-geth + op-reth + Fusaka + BPO1/BPO2)  
- [ ] (non-BLOCKING) Deploy to public Fusaka/BPO-enabled devnet  
- [ ] (non-BLOCKING) Update `op-sepolia` & `op-mainnet` vm-runners with new absolute prestate  
- [ ] (non-BLOCKING) Review skipped upstream merge [op-geth#689](https://github.com/ethereum-optimism/op-geth/pull/689)  
