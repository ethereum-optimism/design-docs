# [Operator Fee]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Operator Fee scalars are set to incorrect values](#fm1-operator-fee-scalars-are-set-to-incorrect-values)
  - [FM2: Broken Fee Estimation (Wallets)](#fm2-broken-fee-estimation-wallets)
  - [FM3: Bug in Receipt Hydrating Logic](#fm3-bug-in-receipt-hydrating-logic)
  - [FM4: Database Growth Impact on Nodes](#fm4-database-growth-impact-on-nodes)
  - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | leruaa                                             |
| Created at         | 2025-01-08                                         |
| Initial Reviewers  |                                                    |
| Need Approval From | maurelian                                          |
| Status             | Draft                                              |

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
  If the operator fee scalars are incorrectly initialized or updated, there is a risk that the transcations fees will be too high. This could lead to a situation where the chain become unusable.
- **Risk Assessment:**
  High impact, low likelihood.
  **Mitigations:**
  Every update to the operator fee scalars should be carefully tested and reviewed before deployment.
- **Detection:** 
  Monitoring gas cost estimation.
- **Recovery Path(s)**:
  If the operator fee parameters are set to unreasonable values, the rollup operator should update the `operatorFeeScalar` and `operatorFeeConstant` to reasonable values as soon as possible.

### FM2: Broken Fee Estimation (Wallets)

- **Description:** 
  If wallets fail to update their fee estimation logic, users will no longer be shown the accurate costs of a transaction.
- **Risk Assessment:**
  Medium impact, medium likelihood.
  **Mitigations:**
  Coordinate with wallet providers to update their fee estimation logic. This includes MetaMask, Coinbase Wallet, and others.
- **Detection:** 
  Confirm that wallets are using the correct fee estimation logic post-launch. This can be done manually on chains that have added an operator fee.
- **Recovery Path(s)**:
  Notify wallets of the new fee structure and ask them to update their fee estimation logic if the operator fee is enabled.

### FM3: Bug in Receipt Hydrating Logic

- **Description:**
  If there is a bug in the receipt hydrating logic, the operator fee may not be correctly reflected in transaction receipts, leading to incorrect fee reporting and potential accounting issues.
- **Risk Assessment:**
  Medium impact, low likelihood.
- **Mitigations:**
  Extensive testing of receipt hydration with various transaction types and fee configurations. Ensure backwards compatibility with existing receipt formats.
- **Detection:**
  Monitor transaction receipts and compare reported fees with expected calculations. Watch for discrepancies in accounting systems.
- **Recovery Path(s):**
  Deploy fix for receipt hydration logic. Historical receipts will remain incorrect but can be recalculated using on-chain data if needed.

### FM4: Database Growth Impact on Nodes

- **Description:**
  The addition of operator fee fields increases the size of transaction receipts, leading to faster database growth. This could accelerate the need for solutions like EIP-4444 or other history expiry mechanisms.
- **Risk Assessment:**
  Medium impact, high likelihood.
- **Mitigations:**
  - Implement history expiry solutions like EIP-4444 when available.
- **Detection:**
  - Monitor database growth rate compared to pre operator fee baseline.
  - Track disk usage metrics across internal nodes.
- **Recovery Path(s):**
  - Use archive nodes to maintain historical data.
  - Consider implementing receipt compression retroactively if needed.


### Generic items we need to take into account:

See [./fma-generic-hardfork.md](./fma-generic-hardfork.md).

- [X] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] _Action item 2 (Assignee: tag assignee)_
- [ ] _Action item 3 (Assignee: tag assignee)_
