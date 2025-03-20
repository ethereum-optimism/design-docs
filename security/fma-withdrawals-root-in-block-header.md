# [Withdrawals Root in Block Header]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Operator Fee scalars are set to incorrect values](#fm1-operator-fee-scalars-are-set-to-incorrect-values)
  - [FM2: Broken Fee Estimation (Wallets)](#fm2-broken-fee-estimation-wallets)
  - [FM3: Bug in Receipt Hydrating Logic](#fm3-bug-in-receipt-hydrating-logic)
  - [FM4: Database Growth Impact on Nodes](#fm4-database-growth-impact-on-nodes)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | George Knee                                        |
| Created at         | 2025-03-20                                         |
| Initial Reviewers  | Mark Tyneway                                       |
| Need Approval From | Tom Assas                                          |
| Status             | DRAFT                                              |

## Introduction

The "Withdrawals Root in Block Header" feature copies some information stored in the L2 blockchain _state_ into the block header, making it a part of the information stored by full (non-archive) nodes. The information in question is the `L2toL1MessagePasser` account storage root, and it is stored in the previously unused `withdrawalsRoot` field of the block header. 

It allows proposals to be made and verified without the needing to bear the cost of running an archive node.

Below are references for this project:
- [Spec](https://specs.optimism.io/protocol/isthmus/exec-engine.html#l2tol1messagepasser-storage-root-in-header)

## Failure Modes and Recovery Paths

### FM1: Fault proofs system failure do to inaccurate `withdrawalsRoot`

- **Description:** 
  Even if all clients agree and there is no chain split, if the `withdrawalsRoot` is incorrect, critical infra used to propose, challenge and validate fault proof games may fail when it is switched to use the `withdrawalsRoot` header field instead of querying the information from an archive node in the usual way.  Because the information is present both in state and in the block header, there is the opportunity for an "off by one" error to arise, if there is confusion  about whether the root applies to the state before or after the block is applied. If there are any problems with activation, this would also cause the same issue.
- **Risk Assessment:**

  **Mitigations:**
  Fault proofs infrastructure (i.e. `op-challenger`) should be run against the newly populated `withdrawalsRoot` field in a testing environment, to confirm it still functions and agrees with a legacy-configured system.

- **Detection:** 
  Fault proof monitoring systems would detect this failure mode.

- **Recovery Path(s)**:
  Fault proof infrastructure would need to be patched. The pre-signed pause may need to be invoked. 


### Generic items we need to take into account: 
See the [generic FMA](./fma-generic-hardfork.md):
* Chain halt at activation
* Activation failure 
* Invalid setImplementation execution
* Chain split (across clients)

## Action Items
- [ ]   Fault proofs infrastructure (i.e. `op-challenger`) should be run against the newly populated `withdrawalsRoot` field in a testing environment, to confirm it still functions and agrees with a legacy-configured system.

## Audit Requirements

An audit has not been deemed necessary.
