# Upgrade 12 (L1 Pectra Defense): Failure Modes and Recovery Path Analysis

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
| Created at          | 5th February 2025   |
| Needs Approval From | Tom Assas           |
| Other Reviewers     | Josep Bove (shadow) |
| Status              | In Review           |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers Upgrade 12, which involves the following changes:

- (Consensus Layer) Ability to parse and validate blocks and receipts with EIP-7702 transactions, as well as blocks with a non-nil EIP-7685 requestsHash field.
- (Smart Contracts) For chains using Fault Proofs, L1 contracts updates which reference an updated absolute prestate hash.

Each change has its own section below with a list of Failure Modes.

Below are references for this project:

- [Github tracker](https://github.com/orgs/ethereum-optimism/projects/117/views/9)
- [Specs clarification (Logs)](https://specs.optimism.io/protocol/derivation.html#on-future-proof-transaction-log-derivation)
- [Specs clarification (L1 Retrieval)](https://specs.optimism.io/protocol/derivation.html#l1-retrieval)
- [OP Contracts Manager specs](https://specs.optimism.io/experimental/op-contracts-manager.html?highlight=opcm#op-contracts-manager)

## Failure Modes and Recovery Paths

### FM1: Consensus bug (client software)

- **Description:** L2 EL/CL clients are not able to parse and/or validate blocks when Pectra goes live on L1, those nodes may halt. This could happen a) if operators do not update their nodes to a suitable release or release candidate before Pectra activates on L1 or b) no such release is available in time or c) there is a bug in such releases.
- **Risk Assessment:** High severity, Low Likelihood
- **Mitigations:**
  1. We will rely on unit tests, [end-to-end tests](https://github.com/ethereum-optimism/optimism/pull/14006) and cross-client devnet acceptance tests to ensure no client halts when L1 activates Pectra. In particular, we performed [manual Kurtosis testing](https://github.com/ethereum-optimism/optimism/pull/14046) with an L1 that has Pectra activated, and sent EIP-7702 transactions to the L1. This mitigates c.
  2. Pectra will activate on Sepolia before L1 mainnet. This will provide another test (in production) of the node software changed. If the sepolia superchain continues to progress its safe chains, this will give us high confidence that the mainnet superchain will also. This mitigates b and c.
  3. We will make our software releases well ahead of time and communicate clearly to operators about the need to upgrade. See these docs https://docs.optimism.io/notices/pectra-changes. This mitigates a and b.
- **Detection:** Manual or (preferably) automated/scheduled testing.
- **Recovery Path(s)**: The affected clients would need to be patched as soon as possible and new releases cut and notices to upgrade broadcast.

### FM2: Consensus bug (proof system)

- **Description:** If any fault proof program is unable parse and/or validate blocks when Pectra goes live on L1, it may be impossible to prove correct blocks and defend against malicious challenges. See FM1 for various scenarios which trigger this failure mode.
- **Risk Assessment:** High severity, Low Likelihood
- **Mitigations:**
  1. Our [end-to-end tests](https://github.com/ethereum-optimism/optimism/pull/14006) include coverage for `op-program` (this runs in CI) as well as `kona` (this has been run manually and passes).
  2. If upstream work does not yet allow for appropriate end-to-end tests, we can patch our L1 clients in the testing environment(s) so we can still run the tests.
- **Detection:** Automatic proofs monitoring systems would alert on-call engineers quickly in this instance. We run the op-challenger (with the new absolute prestate) on our production networks already, so we this part of the system will benefit from several weeks of battle-testing.
- **Recovery Path(s)**: The affected programs would need to be patched as soon as possible and new releases cut. In the meantime, the recovery paths in the [generic FMA document](./fma-generic-hardfork.md#invalid-disputegamefactorysetimplementation-execution) all apply.

### FM3: Bug in OP Contracts Manager

- **Description:** The changes to L1 contracts which are required for this upgrade are being executed by a new path. Any bug could cause a failure of the fault proofs system (see FM2).
- **Risk Assessment:** High severity, Medium Likelihood
- **Mitigations:**
  - The superchain-ops tasks will include both manual checking (in Validations.md) and automated checking (in NestedSignFromJson.s.sol). Thus although the manner of executing the upgrade is changing, we are maintaining the
    pre-existing methods of fully validating the resulting system.
- **Detection:**
  - If a system is misconfigured despite the mitigations, we would not have a predictable method of detecting such issues.
- **Recovery Path(s)**: Any bugs would need to be patched as soon as possible. We can fallback to the old upgrade path using superchain-ops tasks with manually prepared calldata.

## Generic Items

Although this upgrade is technically a soft fork (it does not need to be coordinated across nodes other than being applied before Pectra activates on L1) many of the items in [./fma-generic-hardfork.md](./fma-generic-hardfork.md) apply. In particular:

- Chain Halt at activation
- Invalid `DisputeGameFactory.setImplementation` execution.
- Chain split across clients

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] (BLOCKING): Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [x] (BLOCKING): Action tests will be added which are run on op-node and Kona https://github.com/ethereum-optimism/optimism/issues/13967. Ideally they run against the usual mainline version of op-geth -- if this is not possible the tests can run against a patched version until op-geth is ready to support such tests.
- [x] (BLOCKING): The changes will be deployed to a local multi-client kurtosis devnet with both op-geth and op-reth running as well as Pectra activated on L1. https://github.com/ethereum-optimism/optimism/pull/14046
- [x] (BLOCKING): we will modify the batcher to cope with the change in calldata which activates with Pectra https://github.com/ethereum-optimism/optimism/releases/tag/op-batcher%2Fv1.11.0 so that it can continue to switch DA mode (between blobs and calldata) appropriately.
- [ ] (non-BLOCKING) The changes will be deployed to a devnet which targets a public, Pectra-enabled L1 devnet.
- [x] (non-BLOCKING): We will update the op-sepolia and op-mainnet vm-runners to use the new absolute prestate. The vm-runner runs the op-program in the MIPS FPVM using inputs sampled from a live chain.

Additional action items are copied here from the [generic hardfork FMA](./fma-generic-hardfork.md) doc:

- [ ] (non-BLOCKING): The superchain-ops task to upgrade any contract should check if the semantic versions and bytecodes after the upgrade are as expected.
