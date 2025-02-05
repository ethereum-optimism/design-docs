# Upgrade 12 (L1 Pectra Defense): Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Generic Items](#generic-items)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |                      |
| ------------------- | -------------------- |
| Author              | George Knee          |
| Created at          | 5th February 2025    |
| Needs Approval From |                      |
| Other Reviewers     |                      |
| Status              | Draft                |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers Upgrade 12, which involves the following changes:

- (Consensus Layer) Ability to parse and validate blocks and receipts with EIP-7702 transactions, as well as blocks with a non-nil EIP-7685 requestsHash field. 
- (Smart Contracts) For chains using Fault Proofs, L1 contracts updates which reference an updated absolute prestate hash

Each change has its own section below with a list of Failure Modes.

Below are references for this project:

- [Github tracker](https://github.com/orgs/ethereum-optimism/projects/117/views/9)
- [Specs clarification (Logs)](https://specs.optimism.io/protocol/derivation.html#on-future-proof-transaction-log-derivation)
- [Specs clarification (L1 Retrieval)](https://specs.optimism.io/protocol/derivation.html#l1-retrieval)


## Generic Items

Although this upgrade is technically a soft fork (it does not need to be coordinated across nodes other than being applied before Pectra activates on L1) many of the items in [./fma-generic-hardfork.md](./fma-generic-hardfork.md) apply. In particular: 
- Chain Halt at activation
- Activation failure (node software)
- Invalid `DisputeGameFactory.setImplementation` execution.
- Chain split across clients

- [ ] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements


## Action Items


Additional action items are copied here from the [generic hardfork FMA](./fma-generic-hardfork.md) doc:

