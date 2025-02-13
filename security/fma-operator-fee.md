# [Operator Fee]: Failure Modes and Recovery Path Analysis

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
| Author             | leruaa                                             |
| Created at         | 2025-01-08                                         |
| Initial Reviewers  |  Mark Tyneway                                      |
| Need Approval From | Blaine Malone                                      |
| Status             | Implementing Actions                                          |

## Introduction

This document covers initial deployment of Operator Fee.

The OperatorFee is a new transaction fee that allows new OP chain variants to account for their unique cost structure. For example, the existing fee structure isn't friendly to chains using alt-DA's, since the l1fee only accounts for Ethereum's blobGasFee instead of an alt-DA's fee.

Also, For OP Stack variants that want to utilize ZK proofs, the cost of ZK proving a transaction is a significant resource that is not taken into consideration in the current fee structure.

Below are references for this project:

- [Design Doc](../protocol/operator-fee.md)
- [Spec](https://github.com/ethereum-optimism/specs/pull/382)

## Failure Modes and Recovery Paths

### FM1: Operator Fee scalars are set to incorrect values

- **Description:** 
  If the operator fee scalars are incorrectly initialized or updated, there is a risk that the transactions fees will be too high. This could lead to a situation where the chain becomes unusable.
- **Risk Assessment:**
  High impact, low likelihood.
  **Mitigations:**
  Before setting or updating the operator fee params, the operator should carefully read the [corresponding specs](https://specs.optimism.io/protocol/isthmus/exec-engine.html#operator-fee) and simulate the impact of operator fee on the whole transaction cost.
- **Detection:** 
  By default, the operator fee parameters are set to 0 and the feature is disabled. There are [E2E tests](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/system/fees/fees_test.go) that ensure there is no impact on the transaction cost when the operator fee is disabled.

  On chains that enable operator fee, the operator should monitor the transaction cost and ensure that the operator fee is not too high.
- **Recovery Path(s)**:
  If the operator fee parameters are set to unreasonable values, the rollup operator should update the `operatorFeeScalar` and `operatorFeeConstant` to reasonable values as soon as possible.

  With cast it can be done like this: `cast send <l1_system_config_address> "setOperatorFeeScalars(uint32, uint64)" <operator_fee_scalar> <operator_fee_constant>`

### FM2: Broken Fee Estimation (Wallets)

- **Description:** 
  If wallets fail to update their fee estimation logic, users will no longer be shown the accurate costs of a transaction.
- **Risk Assessment:**
  This failure mode can only happen on chains that enable the operator fee feature.
  Medium impact, medium likelihood.
  **Mitigations:**
  Coordinate with wallet providers to update their fee estimation logic. This includes MetaMask, Coinbase Wallet, and others.
- **Detection:** 
  Using a given wallet, compare the estimated transaction cost with the actual transaction cost, and check if the difference relates to the operator fee, using the formula.
- **Recovery Path(s)**:
  Notify wallets of the new fee structure and ask them to update their fee estimation logic if the operator fee is enabled.

### FM3: Bug in Receipt Hydrating Logic

- **Description:**
  If there is a bug in the receipt hydrating logic, the operator fee may not be correctly reflected in transaction receipts, leading to incorrect fee reporting and potential accounting issues.
- **Risk Assessment:**
  Medium impact, low likelihood.
- **Mitigations:**
  Action and E2E tests covering the receipt hydration logic has been added.

  * [Fee E2E test](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/system/fees/fees_test.go)
  * [Operator Fee Consistency action test](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/operator_fee_test.go)

- **Detection:**
  The action or E2E tests or local testing may pick up an issue.
- **Recovery Path(s):**
  Deploy fix for receipt hydration logic. Historical receipts will remain incorrect but can be recalculated using on-chain data if needed. The fix would be on the execution layer and will not require a hardfork.

### FM4: Database Growth Impact on Nodes

- **Description:**
  The addition of operator fee fields increases the size of transaction receipts, leading to faster database growth. This could accelerate the need for solutions like EIP-4444 or other history expiry mechanisms.
- **Risk Assessment:**
  Medium impact, high likelihood.
- **Mitigations:**
  Implement history expiry solutions like EIP-4444 when available.
- **Detection:**
  - Monitor database growth rate compared to pre operator fee baseline.
  - Track disk usage metrics across internal nodes.

  The following simulation can give a view of the impact of operator fee on the node storage: Operator fee adds 12 bytes per L2 block (4 for operator fee scalar, 8 for operator fee constant). It also add 12 bytes
  per transaction (in the transaction receipt) So, with the arbitrary number of 20 txs per block we have:

  ```
  (12 bytes + 240 bytes / 2 seconds) x 365 days × 24 hours × 60 minutes × 60 seconds = 3,973,536,000 bytes in 1 year.
  ```

  So, about 3,7 GB for 1 year.
- **Recovery Path(s):**
  - Use archive nodes to maintain historical data.
  - Consider implementing receipt compression retroactively if needed.

### Generic items we need to take into account: `L1Block` badly hydrated

- **Description:** At each hardfork, new data can be add to the `L1Block` contract, and the method called to hydrate it change (for instance
    `setL1BlockValuesEcotone` to `setL1BlockValuesIsthmus`). If there is a bug in a future method ending up to operator fee params no
    longer being updated in the `L1Block` contract, the operator fee will no longer be taken into account in transactions fee.
- **Risk Assessment:** medium severity / low likelihood
- **Mitigations:** 
    The [Operator Fee Constistency](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/operator_fee_test.go ) action test runs with all known hardforks activated at genesis, and checks that operator fee parameters are correctly reported to the `L1Block` contract.
- **Detection:** 
    The action or E2E tests or local testing may pick up an issue.
- **Recovery Path(s):**
    - If the bug is located in op-node, a new version must be deployed.
    - If the bug is located in the `L1Block` contract, the contract must be upgraded to fix the bug.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Coordinate with wallet providers to update their fee estimation logic
- [ ] Implement automated monitoring on dabase growth rate