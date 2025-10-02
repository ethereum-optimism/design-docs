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
| Created at          | 2nd October  2025   |
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

### FM1: op-batcher release not rolled out in time

- **Description:**  There is a chance we don't ship the changes we need to op-batcher,  before the forks activate on L1. Also a problem if node or chain operators fail to update in time, even if our releases are available. The impact would be a safe head stall and sequencer window expiry / reorg if the L1 rejects the batcher's blob transactions. This can happen even in the patched batcher, if a config flag is not flipped,

- **Risk Assessment:** High severity, Medium Likelihood
- **Mitigations:**
  - Get a basic release candidate ready as early as possible
  - Communicate widely and clearly about the need to update and any live ops that need to happen.
  - Check with L1 nodes on already forked networks: do they accept legacy batcher blob transactions?
  
- **Detection:**
  - Usual alerts and dashboards
- **Recovery Path(s)**: 
    - Even an old batcher can switch to calldata as a good mitigation

### FM2: op-node or kona node release not rolled out in time

- **Description:**  There is a chance we don't ship the changes we need to op-node or kona node, before the forks activate on L1. For example, the code we need to patch op-node exists in go-ethereum, but is only released late on in the development cycle (close to when we need to ship the patch).  Also a problem if node or chain operators fail to update in time, even if our releases are available. The impact would be an incorrect blob base fee, causing L2 users to be excessively overcharged, even leading to an effective denial of service due to high fees on L2. Moreover, there is a likelihood of a chain split if nodes are updated in an uncoordinated way. We have seen this failure mode [play out on Sepolia](https://www.notion.so/oplabs/OP-Sepolia-L1-Cost-Blob-Schedule-Bug-1adf153ee1628009b429ee10828fb300#1adf153ee1628009b429ee10828fb300) and it required a hardfork to fix. 

Note that, with the current blob parameter schedule, it is actually only BPO1 which will cause the problem, not Fusaka itself.


- **Risk Assessment:** High severity, Medium Likelihood
- **Mitigations:**
  - Get a basic release candidate ready as early as possible
  - Communicate widely and clearly about the need to update and any live ops that need to happen.
  - Take extra care when doing the _next_ upstream merge of op-geth, even after the Fusaka project is done. Are there any consensus changes we need to worry about?
  
- **Detection:**
  - We will know easily if we have made the releases on time.
- **Recovery Path(s)**: 
  - We may need a one-off fork to fix any chain split.



### FM3: Unintended / incorrect behaviour by lagging behind upstream Geth
- **Description:**  We made a decision to delay updating op-geth with the full changes from go-ethereum's official Fusaka release for testnets, and we will continue to wait to do that even when they make the official Fusaka release for mainnet. The reason for this was to delay the adoption of Go 1.24, which those releases would force us into, but which cost us operational time and effort. This is not as risky as it first appears, since we are not adopting Fusaka on L2. In fact it could be argued to be the less risky approach, since we make a much more targetted and tightly-scoped change to our existing state transition rules by going this route. Still, there is a chance we don't pull in an important bug fix or security patch.


- **Risk Assessment:** High severity, Low Likelihood
- **Mitigations:**
  - Do a thorough review of the upstream merge we are **not** doing, to see if there are any critical fixes we are missing out on.
  - Take extra care when doing the _next_ upstream merge of op-geth, even after the Fusaka project is done. Are there any consensus changes we need to worry about?
  
- **Detection:**
  - We may get an immunifi report or suffer an attack of some kind.
- **Recovery Path(s)**: 
  - If the above review reveals an important patch, we can cherry pick it into op-geth. 




## Generic Items

Although this upgrade is technically a soft fork (it does not need to be coordinated across nodes other than being applied before Pectra activates on L1) many of the items in [./fma-generic-hardfork.md](./fma-generic-hardfork.md) apply. In particular:

- Chain Halt at activation.
- Invalid `DisputeGameFactory.setImplementation` execution.
- Chain split across clients.

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements

## Appendix

The following table shows how each EIP from Fusaka (see [meta EIP](https://eips.ethereum.org/EIPS/eip-7607)) will affect OPStack chains:

| EIP | Description | Compatibility Concerns | Components Affected |
| --- | --- | --- | --- |
| [7594](https://eips.ethereum.org/EIPS/eip-7594)  | **PeerDAS (Headline feature)** | Blob Transaction senders must compute and attach “cell proofs”| `op-batcher` (tx-mgr), `proyxd` |
| [7642](https://eips.ethereum.org/EIPS/eip-7642) | `eth/69` History expiry and simpler receipts | None |  |
| [7823](https://eips.ethereum.org/EIPS/eip-7823) | Set upper bounds for MODEXP | None |  |
| [7825](https://eips.ethereum.org/EIPS/eip-7825) | Transaction Gas Limit Cap | May limit superchain-ops (contract upgrade transactions on L1) and require them to be broken up. | `superchain-ops` |
| [7883](https://eips.ethereum.org/EIPS/eip-7883) | ModExp Gas Cost Increase |  | |
| [7892](https://eips.ethereum.org/EIPS/eip-7892) | **Blob Parameter Only Hardforks (Headline feature)** | A framework to enable changing the blob schedule on L1 with less overhead / more frequently. Any such change will affect the L1 Cost, and a new prestate is implied when the schedule is updated.  | `op-node` / `op-program` and `kona` |
| [7917](https://eips.ethereum.org/EIPS/eip-7917) | Deterministic proposer lookahead | None, only affects consensus layer on L1 (PoS) |  |
| [7918](https://eips.ethereum.org/EIPS/eip-7918) | Blob base fee bounded by execution cost   |  No action is necessary, since the change happens inside the `excessBlobGas` function.  |  |
| [7934](https://eips.ethereum.org/EIPS/eip-7934) | RLP Execution Block Size Limit |  |  |
| [7935](https://eips.ethereum.org/EIPS/eip-7935) | Set default gas limit to XX0M | Actual limit not set yet. It will not decrease though, so should be backwards compatible. Doesn’t strictly require a hardfork. |  |
| [7939](https://eips.ethereum.org/EIPS/eip-7939) | Count Leading Zeros (CLZ) opcode | None |  |
| [7951](https://eips.ethereum.org/EIPS/eip-7951) | Precompile for secp256r1 support | None |  |
| [7910](https://eips.ethereum.org/EIPS/eip-7910) | eth_config JSON-RPC method | Non breaking change. | |


## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] (BLOCKING): Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] (BLOCKING): Action tests will be added which are run on op-node and Kona  
- [ ] (BLOCKING): The changes will be deployed to a local multi-client devnet with both op-geth and op-reth running as well as Fusaka and BPO1 and BPO2 activated on L1. 
- [ ] (non-BLOCKING): The changes will be deployed to a devnet which targets a Fusaka- and BPO1,2-enabled L1 
- [ ] (non-BLOCKING): We will update the op-sepolia and op-mainnet vm-runners to use the new absolute prestate. The vm-runner runs the op-program in the MIPS FPVM using inputs sampled from a live chain.
- [ ] (non-BLOCKING) Do a review of the upstream merge which we are **not yet adopting** https://github.com/ethereum-optimism/op-geth/pull/689. 




