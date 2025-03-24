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

|                    |              |
| ------------------ | ------------ |
| Author             | George Knee  |
| Created at         | 2025-03-20   |
| Initial Reviewers  | Mark Tyneway |
| Need Approval From | Tom Assas    |
| Status             | DRAFT        |

## Introduction

The "Withdrawals Root in Block Header" feature copies some information stored in the L2 blockchain _state_ into the block header, making it a part of the history (information stored by all, including non-archive, nodes). The information in question is the `L2toL1MessagePasser` account storage root, and it is stored in the previously unused `withdrawalsRoot` field of the block header.

It allows proposals to be made and verified without the needing to bear the cost of running an archive node.

Below are references for this project:

- [Spec](https://specs.optimism.io/protocol/isthmus/exec-engine.html#l2tol1messagepasser-storage-root-in-header)
- [op-geth PR](https://github.com/ethereum-optimism/op-geth/pull/451)
- [monorepo PR](https://github.com/ethereum-optimism/optimism/pull/13962)

## Failure Modes and Recovery Paths

### FM1: Withdrawals downtime do to inaccurate `withdrawalsRoot`

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

### FM2: Chain split from genesis due to future changes to genesis state tooling

- **Description:**
  The genesis state is, with the activation of Isthmus at genesis, now part of the genesis block. Therefore changes to the tooling which generates the genesis block from the genesis state can cause a chain split if there is a bug introduced in the future or even if the genesis state is changed intentionally but in an uncoordinated manner.

- **Risk Assessment:**

  **High impact, low likelihood**

  **Mitigations:**

  A new e2e test could be introduced, to recompute the genesis block hash for a few select chains and compare to the existing (snapshot) block hashes stored in the superchain registry.

- **Detection:**
  Existing or newly added e2e tests would hopefully catch this. If the bug made it to production, replica healthcheck alerts would fire as nodes diverged.

- **Recovery Path(s)**:
  The changes to the genesis state would need to be reverted and rescheduled into a hardfork if still desired. Chains which were sequenced with the modified genesis sate logic may need to be repaired with a special hardfork.

### FM3: Failure of p2p network due to bug in new topic/message serde logic

- **Description:**
  Because this feature introduces a new p2p gossip topic and message serialization format, a bug can mean the failure of p2p gossip for any chain with Isthmus active. This would cause an unsafe chain halt.

- **Risk Assessment:**

  **High impact, low likelihood**

  **Mitigations:**
  We rely on end-to-end testing (including fuzzing) to catch any bugs in this code path. We could run extended fuzzing campaigns.

- **Detection:**
  Continuous integration, or Kurtosis and/or devnet testing should catch this. Failing that, the bug makes it to production, our alerting infrastructure would notify us.

- **Recovery Path(s)**:
  The bug would need to be patched and new op-node release cut and rolled out.

## Generic failure modes:

See the [generic FMA](./fma-generic-hardfork.md):

- Chain halt at activation (there is a change to the engine API, which elevates this risk)
- Activation failure
- Invalid setImplementation execution
- Chain split (across clients)

## Specific Action Items

- [ ] (BLOCKING) e2e tests must check for consistency between output roots returned from op-node and those constructed manually in the old way
- [ ] (non-BLOCKING) op-node could be furnished with an override to make it serve output roots in the legacy fashion; this would also aid in testing (see above item). It would even allow us to run the two systems side by side for a time before fully switching over.
- [ ] (non-BLOCKING) op-geth could be made to log a critical error triggering an alert if ever the `withdrawals` list in the block body is non empty (post Isthmus)

## Generic Action Items

- [x] (BLOCKING): We have implemented extensive unit and end-to-end testing of the activation flow: https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/upgrades/isthmus_fork_test.go
- [ ] (BLOCKING): We have implemented multi-client testing to reduce the chance of bugs (the above test could be migrated to a fault proof test where it can run on kona)
- [ ] (non-BLOCKING): We have implemented fuzz testing in a kurtosis multi-client devnet to reduce the chance of bugs
- [ ] (BLOCKING): We will be testing the activation on our devnets and testnets.

## Audit Requirements

An audit has not been deemed necessary.
