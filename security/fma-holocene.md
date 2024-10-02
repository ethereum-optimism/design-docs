# Holocene Hardfork: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Holocene Derivation](#holocene-derivation)
  - [The batcher violates the new (stricter) ordering rules](#the-batcher-violates-the-new-stricter-ordering-rules)
    - [Description](#description)
    - [Risk Assessment](#risk-assessment)
    - [Mitigations](#mitigations)
    - [Detection](#detection)
    - [Recovery Path(s)](#recovery-paths)
  - [Derivation Deadlock (either specification or op-node bug)](#derivation-deadlock-either-specification-or-op-node-bug)
    - [Description](#description-1)
    - [Risk Assessment](#risk-assessment-1)
    - [Mitigations](#mitigations-1)
    - [Detection](#detection-1)
    - [Recovery Path(s):](#recovery-paths)
- [Deterministic Standard L2 Genesis](#deterministic-standard-l2-genesis)
- [Configurable EIP-1559 Parameters via SystemConfig](#configurable-eip-1559-parameters-via-systemconfig)
- [L2ToL1MessagePasser Storage Root in Header](#l2tol1messagepasser-storage-root-in-header)
- [Update to the MIPS contract](#update-to-the-mips-contract)
- [Generic Items](#generic-items)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

| | |
|--------|--------------|
| Author | George Knee |
| Created at | 2024-09-24 |
| Needs Approval From | maurelian |
| Other Reviewers |   |
| Status | Draft |


## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers the Holocene hardfork, which involves the following changes: 
- (Consensus Layer) Holocene Derivation
- (Execution Layer & Smart Contracts) Configurable EIP-1559 Parameters via SystemConfig
- (Smart Contracts) Update to the MIPS contract

Each change has it's own section below with a list of Failure Modes.

Below are references for this project:

- [PID: Holocene hardfork upgrade](https://www.notion.so/PID-Holocene-hardfork-upgrade-00ee1ffc414a407088fdb49841771527?pvs=21)
- [Github tracker](https://github.com/orgs/ethereum-optimism/projects/84/views/6)
- [Specs](https://specs.optimism.io/protocol/holocene/derivation.html?highlight=holocene#holocene-derivation)


##  Holocene Derivation

### The batcher violates the new (stricter) ordering rules

#### Description

If the batcher:
  - sends transactions out of order, or
  - orders frames within transactions out of order, or
  - orders batches within a channel out of order

This will lead to the consensus client hitting error conditions when it loads the frames from L1. According to the spec, frames will be dropped in order to maintain a contiguous frame queue, and out-of-order batches will be dropped at a later stage in the derivation pipeline. The safe chain would halt until the next batch can be resubmitted in order.
    
#### Risk Assessment
 low severity / low likelihood
#### Mitigations
The batcher implementation could:
  - leverage nonce management to avoid sending transactions out of order in the first place (see [spec](https://specs.optimism.io/protocol/holocene/derivation.html?highlight=holocene#batcher-hardening))
  - be hardened so detect the chain halt and resubmit the frames which were dropped (see [spec](https://specs.optimism.io/protocol/holocene/derivation.html?highlight=holocene#batcher-hardening)) either:
    - as a part of normal operation
    - as a part of its startup behavior (i.e. to be triggered by a restart)
    - as a part of an emergency mode triggered by an admin API, this could allow for manual intervention
  - continually check for contiguity in its internal state, and panic or reset if this is violated (possibly then triggering some recovery behavior)
#### Detection
L2 safe chain halt
#### Recovery Path(s)
We would need to get the batcher to resubmit the transaction sent out of order. No governance or hardfork needed to recover. A simple batcher restart would cause the batcher to re-batch the unsafe chain, which should lead to recovery. Moreover, we could also temporarily operate the batcher with reduced tx sending concurrency, which should avoid out of order txs.

### Derivation Deadlock (either specification or op-node bug)

#### Description
It is possible that either i) the Holocene Derivation Specification or ii) the consensus client implementation has overlooked a corner case such that derivation simply halts when that corner case arises.
#### Risk Assessment
** high severity / medium likelihood
#### Mitigations 
Multi client testing should help surface any consensus client bugs, possibly including fuzzing of some description.
We should backport as much insight as possible from the implementation (which is code) to the specs (which is just words). Here's an example of doing just that [holocene: fix frame queue specs of derivation](https://github.com/ethereum-optimism/specs/pull/386).
#### Detection
L2 safe chain halt
#### Recovery Path(s):
We would need to fix the the implementation via a hardfork. As a hotfix, we would probably also make an emergency release which migrates back to the old DP at the block where derivation is stuck. We might already implement something like this as a contingency, e.g. adding a holocene_deactivation_time that would then move back to old derivation if this time is set. We could then instruct node operators to set this flag to some value, providing a quick recovery path.

##  Deterministic Standard L2 Genesis
##  Configurable EIP-1559 Parameters via SystemConfig
##  L2ToL1MessagePasser Storage Root in Header
##  Update to the MIPS contract
##  Generic Items
See [./fma-generic-hardfork.md](./fma-generic-hardfork.md). 

- [ ] Check this box to confirm that these items have been considered and updated if necessary.


## Audit Requirements

This may require a mini audit due to updates to the MIPS contract.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)

