# Withdrawals Root in Block Header: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Fault proofs system failure do to inaccurate `withdrawalsRoot`](#fm1-fault-proofs-system-failure-do-to-inaccurate-withdrawalsroot)
- [Generic failure modes:](#generic-failure-modes)
- [Specific Action Items](#specific-action-items)
- [Generic Action Items](#generic-action-items)
- [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | George Knee                                        |
| Created at         | 2025-03-20                                         |
| Initial Reviewers  | Mark Tyneway                                       |
| Need Approval From | Tom Assas                                          |
| Status             | DRAFT                                              |

## Introduction

The "Withdrawals Root in Block Header" feature copies some information stored in the L2 blockchain _state_ into the block header, making it a part of the information stored by all (including non-archive) nodes. The information in question is the `L2toL1MessagePasser` account storage root, and it is stored in the previously unused `withdrawalsRoot` field of the block header. 

It allows proposals to be made and verified without the needing to bear the cost of running an archive node.

Below are references for this project:
- [Spec](https://specs.optimism.io/protocol/isthmus/exec-engine.html#l2tol1messagepasser-storage-root-in-header)
- [op-geth PR](https://github.com/ethereum-optimism/op-geth/pull/451)
- [monorepo PR](https://github.com/ethereum-optimism/optimism/pull/13962)

## Failure Modes and Recovery Paths

### FM1: Fault proofs system failure do to inaccurate `withdrawalsRoot`

- **Description:** 
  If the `withdrawalsRoot` in the block header is incorrect, critical infra used to propose, challenge and validate fault proof games may fail. This is because it has already been switched to use the `withdrawalsRoot` header field instead of querying the information from an archive node in the usual way. Output roots are returned by the op-node [`optimism_outputAtBlock`](https://docs.optimism.io/operators/node-operators/json-rpc#optimism_outputatblock) RPC method, and this behaves differently under Isthmus -- when handling a request for an output root (it no longer delegates an `eth_getProof` to op-geth and just reads the information from the block header).
  
  Triggers: 
  
  * A failed hardfork activation in the execution client.

  * If there is an execution client bug, for example it is possible the root is (incorrectly) added to the header before the state is fully committed. 

  * If we were to ever introduce non empty `withdrawals` in the block body, this might override the mechanism introduced with this feature and invalidate the interpreation of the `withdrawalsRoot` field. 

- **Risk Assessment:**

  ** Medium impact, low likelihood ** 
  
  Temporary downtime for withdrawals, but not a _consensus criticial_ bug.

  **Mitigations:**
  * We rely on e2e tests to check for consistency between the outputs returned from op-node and those constructed manually in the old way.

  * Instead of waiting for the failure mode to materialize and then writing a patch in a rush, we could add an optional config var to op-node to switch it back into the old behavior. Rolling out the patch would then be extremely fast and no software releases would be necessary.

  * A killswitch could be installed in op-geth, such that as long as isthmus is active the node should halt if the withdrawals list in the body is ever non empty.

- **Detection:** 
  Fault proof monitoring systems would detect this failure mode.

- **Recovery Path(s)**:
  Fault proof infra would nee to be pointed at a patched op-node. The patch would restore the old behaviour for generating output roots.


## Generic failure modes: 
See the [generic FMA](./fma-generic-hardfork.md):
* Chain halt at activation (there is a change to the engine API, which elevates this risk)
* Activation failure
* Invalid setImplementation execution
* Chain split (across clients)

## Specific Action Items
- [ ] (BLOCKING) e2e tests must check for consistency between output roots returned from op-node and those constructed manually in the old way
- [ ] (non-BLOCKING) op-node could be furnished with an override to make it serve output roots in the legacy fashion; this would also aid in testing (see above item)
- [ ] (non-BLOCKING) op-geth could be made to log a critical error if ever the `withdrawals` list in the block body is non empty (post Isthmus)

## Generic Action Items
- [x] (BLOCKING): We have implemented extensive unit and end-to-end testing of the activation flow: https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/upgrades/isthmus_fork_test.go
- [ ] (BLOCKING): We have implemented multi-client testing to reduce the chance of bugs (the above test could be migrated to a fault proof test where it can run on kona)
- [ ] (non-BLOCKING): We have implemented fuzz testing in a kurtosis multi-client devnet to reduce the chance of bugs
- [ ] (BLOCKING): We will be testing the activation on our devnets and testnets.

## Audit Requirements

An audit has not been deemed necessary.
