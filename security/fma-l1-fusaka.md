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
| Created at          | 2nd October  2025   |
| Needs Approval From |                     |
| Other Reviewers     |                     |
| Status              | DRAFT               |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers Upgrade 16b, which involves the following changes:

- (Consensus Layer) Ability to compute an accurate reflection of the L1 Blob Base Fee on L2, even when L1 activates Blob Parameter Only forks.
- (Smart Contracts) For chains using Fault Proofs, L1 contracts updates which reference an updated absolute prestate hash.

Each change has its own section below with a list of Failure Modes.

Below are references for this project:

- [Github tracker](https://github.com/ethereum-optimism/optimism/issues/17471)
- [Specs clarification](https://github.com/ethereum-optimism/specs/pull/790)
- [OP Contracts Manager specs](https://specs.optimism.io/experimental/op-contracts-manager.html?highlight=opcm#op-contracts-manager)
- [Optimism Docs on Pectra (including L1 activation times)](https://oplabs.notion.site/fusaka-upgrade-readiness-doc)

## Failure Modes and Recovery Paths

### FM1: Consensus bug (client software)

- op-geth cherry pick

### FM2: Consensus bug (proof system)




## Generic Items

Although this upgrade is technically a soft fork (it does not need to be coordinated across nodes other than being applied before Pectra activates on L1) many of the items in [./fma-generic-hardfork.md](./fma-generic-hardfork.md) apply. In particular:

- Chain Halt at activation.
- Invalid `DisputeGameFactory.setImplementation` execution.
- Chain split across clients.

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements

## Appendix

The following table shows how each EIP from Pectra will affect OPStack chains:

| EIP      | Description                                                | Impact on L2 Consensus Rules                                                                                                                  |
| -------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| EIP-7594 | PeerDAS                 |TODO.                                                      |
                                                                        |

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] (BLOCKING): Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] (BLOCKING): Action tests will be added which are run on op-node and Kona https://github.com/ethereum-optimism/optimism/issues/13967. Ideally they run against the usual mainline version of op-geth -- if this is not possible the tests can run against a patched version until op-geth is ready to support such tests.
- [ ] (BLOCKING): The changes will be deployed to a local multi-client devnet with both op-geth and op-reth running as well as Pectra activated on L1. https://github.com/ethereum-optimism/optimism/pull/14046
- [ ] (non-BLOCKING): The changes will be deployed to a devnet which targets a public, Fusaka- and BPO1,2-enabled L1 [devnet]([url](https://devnets.optimism.io/balrog.html))
- [ ] (non-BLOCKING): We will update the op-sepolia and op-mainnet vm-runners to use the new absolute prestate. The vm-runner runs the op-program in the MIPS FPVM using inputs sampled from a live chain.





