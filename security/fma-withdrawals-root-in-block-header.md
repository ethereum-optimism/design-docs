# Withdrawals Root in Block Header: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Withdrawals downtime do to inaccurate "withdrawalsRoot"](#fm1-withdrawals-downtime-do-to-inaccurate-withdrawalsroot-in-the-block-header)
  - [FM2: Failure of p2p network due to bug in new topic/message serde logic](#fm2-failure-of-p2p-network-due-to-bug-in-new-topicmessage-serde-logic)
- [Generic failure modes:](#generic-failure-modes)
- [Specific Action Items](#specific-action-items)
- [Generic Action Items](#generic-action-items)
- [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                      |
| ------------------ | ------------------------------------ |
| Author             | George Knee                          |
| Created at         | 2025-03-20                           |
| Initial Reviewers  | Mark Tyneway                         |
| Need Approval From | Tom Assas, Michael Amadi (Shadowing) |
| Status             | Implementing Actions 🛫              |

## Introduction

The "Withdrawals Root in Block Header" feature copies some information stored in the L2 blockchain _state_ into the block header, making it a part of the history (information stored by all, including non-archive, nodes). The information in question is the `L2toL1MessagePasser` account storage root, and it is stored in the previously unused `withdrawalsRoot` field of the block header.

It allows proposals to be made and verified without the needing to bear the cost of running an archive node.

Below are references for this project:

- [Spec](https://specs.optimism.io/protocol/isthmus/exec-engine.html#l2tol1messagepasser-storage-root-in-header)
- [op-geth PR](https://github.com/ethereum-optimism/op-geth/pull/451)
- [monorepo PR](https://github.com/ethereum-optimism/optimism/pull/13962)

## Failure Modes and Recovery Paths

### FM1: Withdrawals downtime do to inaccurate `withdrawalsRoot` in the block header

- **Description:**
  If the `withdrawalsRoot` in the block header is incorrect, critical infra used to enable withdrawals may fail. Namely, output proposals and challenges would be incorrect, affecting chains with permissioned and chains with permissionless proofs. This is because these components will, with the activation of the Isthmus fork, use the `withdrawalsRoot` header field instead of querying the information from an archive node in the usual way. Output roots are returned by the op-node [`optimism_outputAtBlock`](https://docs.optimism.io/operators/node-operators/json-rpc#optimism_outputatblock) RPC method, and this behaves differently under Isthmus -- when handling a request for an output root (it no longer delegates an `eth_getProof` to op-geth and just reads the information from the block header).

  Triggers:

  - A failed hardfork activation in the execution client.

  - If there is an execution client bug, for example it is possible the root is (incorrectly) added to the header before the state is fully committed.

  - If we were to ever introduce non empty `withdrawals` in the block body, this might override the mechanism introduced with this feature and invalidate the interpreation of the `withdrawalsRoot` field.

- **Risk Assessment:**

  **High impact, low likelihood**

  Temporary downtime for withdrawals, or loss of funds if not remediated in time to challenge a malicious proposal.

  **Mitigations:**

  - We rely on e2e tests to check for consistency between the outputs returned from op-node and those constructed manually in the old way.

  - Instead of waiting for the failure mode to materialize and then writing a patch in a rush, we could add an optional config var to op-node to switch it back into the old behavior. Rolling out the fix would not then require any software releases.

  - op-geth could be modified to log a critical error (triggering an alert) if the withdrawals list in the body is ever non empty.

- **Detection:**
  Fault proof monitoring systems may not detect this failure mode immediately, until an actor running patched software made a proposal or challenge.

- **Recovery Path(s)**:
  Fault proof infra would nee to be pointed at a patched op-node. The patch would restore the old behaviour for generating output roots.

### FM2: Failure of p2p network due to bug in new topic/message serde logic

- **Description:**
  Because this feature introduces a new p2p gossip topic and message serialization format, a bug can mean the failure of p2p gossip for any chain with Isthmus active. This would cause an unsafe chain halt on affected nodes (but the safe chain would still progress).

- **Risk Assessment:**

  **High impact, low likelihood**

  **Mitigations:**
  We rely on [end-to-end testing](https://github.com/ethereum-optimism/optimism/blob/9249efc6343208f69283290fc9c5c8f6e7b243f8/op-service/eth/ssz_test.go#L251) (including fuzzing) to catch any bugs in this code path. We could run extended fuzzing campaigns.

- **Detection:**
  Continuous integration, or Kurtosis and/or devnet testing would catch this. Failing that, the bug makes it to production, our alerting infrastructure would notify us.

- **Recovery Path(s)**:
  The bug would need to be patched and new op-node release cut and rolled out.

## Generic failure modes:

See the [generic FMA](./fma-generic-hardfork.md):

- Chain halt at activation (there is a change to the engine API, which elevates this risk)
- Activation failure
- Invalid setImplementation execution
- Chain split (across clients)

## Specific Action Items

- [x] (BLOCKING) e2e tests must check for consistency between output roots returned from op-node and those constructed manually in the old way https://github.com/ethereum-optimism/optimism/blob/6a436fe9ac9acb215b0f4b9f87ccd3832f4d6b72/op-e2e/actions/upgrades/isthmus_fork_test.go#L286-L301
- [x] (non-BLOCKING) op-node could be furnished with an override to make it serve output roots in the legacy fashion; this would also aid in testing (see above item). It would even allow us to run the two systems side by side for a time before fully switching over. https://github.com/ethereum-optimism/optimism/pull/15150
- [ ] (non-BLOCKING) op-geth could be made to log a critical error triggering an alert if ever the `withdrawals` list in the block body is non empty (post Isthmus)

## Generic Action Items

- [x] (BLOCKING): We have implemented extensive unit and end-to-end testing of the activation flow: https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/upgrades/isthmus_fork_test.go
- [x] (BLOCKING): We have implemented multi-client testing with kurtosis and/or devnets to reduce the chance of bugs. This should be in the form of an acceptance tests which target all client types in the network https://github.com/ethereum-optimism/optimism/pull/15102
- [x] (BLOCKING) We should ensure that our usual suite of alerts applies to devnets and are routed to protocol engineers signing off on the devnet completion.
- [x] (BLOCKING): Run fuzzing on the v4 gossip p2p more than 10s (assignee: @Ethnical @geoknee) https://github.com/ethereum-optimism/optimism/pull/15068
- [x] (BLOCKING): We tested the activation on our [devnets](https://devnets.optimism.io/interop-rc-alpha.html).
- [ ] (non-BLOCKING): Creating a monitoring that differential testing from the merkle tree inclusion computation and the block.header request `withdrawalRoot` (assignee: @Ethnical). Tracking -> [Monitoring Security-Issue](https://github.com/ethereum-optimism/security-pod/issues/252)
- [ ] (non-BLOCKING): We have implemented fuzz testing in a kurtosis multi-client devnet to reduce the chance of bugs

## Audit Requirements

An audit has not been deemed necessary.
