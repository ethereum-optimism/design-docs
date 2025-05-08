# [Operator Fee]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Operator Fee scalars are set to incorrect values](#fm1-operator-fee-scalars-are-set-to-incorrect-values)
  - [FM2: Broken Fee Estimation (Wallets)](#fm2-broken-fee-estimation-wallets)
  - [FM3: Bug in Receipt Hydrating Logic](#fm3-bug-in-receipt-hydrating-logic)
  - [FM4: Database Growth Impact on Nodes](#fm4-database-growth-impact-on-nodes)
  - [FM5: EVM incorrectly charges operator fee](#fm5-evm-incorrectly-charges-operator-fee)
  - [FM6: Transaction Pool is not updated to reflect transaction fee requirements](#fm6-transaction-pool-is-not-updated-to-reflect-transaction-fee-requirements)
  - [FM7: Discrepancy between client implementations of operator causes a chain fork](#fm7-discrepancy-between-client-implementations-of-operator-causes-a-chain-fork)
  - [FM8: Generic items we need to take into account: `L1Block` badly hydrated](#fm8-generic-items-we-need-to-take-into-account-l1block-badly-hydrated)
  - [FM9: Overflow/Underflow of operator fee calculation](#fm9-overflowunderflow-of-operator-fee-calculation)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                       |                                    |
| --------------------- | ---------------------------------- |
| Author                | leruaa                             |
| Created at            | 2025-01-08                         |
| Initial Reviewers     | Mark Tyneway                       |
| Need Approval From    | Blaine Malone                      |
| Need Re-Approval From | Tom ASSAS & Michael Amadi (shadow) |
| Status                | Implementing Actions 🛫            |

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

  - [Fee E2E test](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/system/fees/fees_test.go)
  - [Operator Fee Consistency action test](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/operator_fee_test.go)

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
  The decision has been made to not store operator fee parameters in the
  receipts if their values hasn't been set. So updated database growth rate is the following:

  ```
  (12 bytes / 2 seconds) x 365 days × 24 hours × 60 minutes × 60 seconds = 189,216,000 bytes in 1 year.
  ```

  So, about 180 GB for 1 year. Therefore, we don't think the following recovery paths are necessary anymore:

  - Use archive nodes to maintain historical data.
  - Consider implementing receipt compression retroactively.

### FM5: EVM incorrectly charges operator fee

- **Description:** The operator fee is integrated directly into the EVM, and failure to charge the fee correctly can result in ETH being minted or burned on the L2 unexpectedly.
- **Risk Assessment:** High impact, low likelihood
- **Mitigations**: Several tests have been added across multiple test suites to enforce that, with a range of operator fee configurations, no ETH is minted or burned on the L2 as a result of the feature. In addition, tests have been added to account for the specific transaction types that the operator fee applies to.
  - **Cross-client Integration Tests:**
    - https://github.com/ethereum-optimism/optimism/blob/7e6825bfb4affe7bda58ba2b32cad4c6b0198734/op-e2e/actions/proofs/operator_fee_test.go#L20
      - Covers (`op-program`, `kona`, `op-geth`, `op-node`):
        - Non-deposit transaction types are correctly charged the expected operator fee.
        - Deposit transactions are **never** charged the operator fee.
        - The operator fee refund accounts for the EVM's state refund.
        - No ETH is minted or burned as a result of the operator fee being charged.
        - When the operator fee parameters are configured to `0`, the operator fee is not applied.
          - The operator fee may never apply on the Isthmus transition block.
    - https://github.com/ethereum-optimism/optimism/blob/7e6825bfb4affe7bda58ba2b32cad4c6b0198734/op-acceptance-tests/tests/isthmus/fees_test.go#L82-L83
      - Covers (`op-geth`, `op-node`, `op-reth`, and other OP EL + CL clients supported by kurtosis): For all OP Stack EL/CL pairs (running on a kurtosis devnet,) the operator fee neither mints nor burns ETH on the L2. Additionally, asserts that clients participating in the devnet stay in-sync throughout the test.
- **Recovery Path(s):**
  - Depending on the severity of impact, and if the impacted client is sequencing the network or being used in the bridge's proof, we can choose to make the bug canonical, manually re-collateralizing the bridge on L1. Or, we could pause the bridge and hardfork to destroy any new ETH or re-mint any lost ETH.

### FM6: Transaction Pool is not updated to reflect transaction fee requirements

- **Description:** If the new transaction fee formula is not properly represented in the tx pool admission function implementation, then a transaction can enter the pool without being valid for inclusion in a block, causing the transaction to get stuck in the pool.
- **Risk Assessment:** Medium impact, medium likelihood
- **Mitigations:** Update the transaction pool admission function to use the exact same fee formula as the EVM so that a transaction does not enter the pool unless it is considered valid for inclusion in a block. Testing has been added to ensure that the fee logic matches going forward.
  - **Fixes:**
    - [op-geth PR #558: core/txpool: Take total rollup cost into account (L1 + operator fee)](https://github.com/ethereum-optimism/op-geth/pull/558/files)
  - **Tests:**
    - _TODO: Update with link to @sebastemas's test of the txpool fee logic_
- **Recovery Path(s):**
  - An issue in the transaction pool logic is not consensus critical, and can be patched without a hardfork. In the event user experience is degraded, client teams can cut a new release and ask users to upgrade.

### FM7: Discrepancy between client implementations of operator causes a chain fork

- **Description:**: Since operator fee logic is executed within the L2 EVM, any differences in the two implementations' management of state can trigger a chain fork, where one client believes the state root of a given block to be **X** and another client believes the state root of the same block to be **Y**. Given that the codebase for reth and op-geth look very different, producing code to perform identical calculations down to details such as bit precision, casting and saturation modes, is difficult. Without adaquate testing, it would be difficult to claim that a chain fork would not occur in some operator fee scenarios.
- **Risk Assessment**: Medium impact, high likelihood
- **Recovery Path(s):**
  - Test that enabling operator fee and later updating the operator fee parameters does not cause a chain split while transactions without fuzzing.
    - [PR #14972: Update operator fee NAT test to verify there is no chain split](https://github.com/ethereum-optimism/optimism/pull/14972)
  - An additional test of the above _with_ fuzzing.

### FM8: Generic items we need to take into account: `L1Block` badly hydrated

- **Description:** At each hardfork, new data can be add to the `L1Block` contract, and the method called to hydrate it change (for instance
  `setL1BlockValuesEcotone` to `setL1BlockValuesIsthmus`). If there is a bug in a future method ending up to operator fee params no
  longer being updated in the `L1Block` contract, the operator fee will no longer be taken into account in transactions fee.
- **Risk Assessment:** medium severity / low likelihood
- **Mitigations:**
  The [Operator Fee Constistency](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/operator_fee_test.go) action test runs with all known hardforks activated at genesis, and checks that operator fee parameters are correctly reported to the `L1Block` contract.
- **Detection:**
  The action or E2E tests or local testing may pick up an issue.
- **Recovery Path(s):**
  - If the bug is located in op-node, a new version must be deployed.
  - If the bug is located in the `L1Block` contract, the contract must be upgraded to fix the bug.

### FM9: Overflow/Underflow of operator fee calculation

- **Description:** If the operator fee calculation is vulnerable to overflows or underflows then it could be possible to manipulate the calculation through, say, a deliberately large gasUsed in order to trigger that overflow or underflow and charge a negative fee.
- **Risk Assessment:** high severity / low likelihood
- **Mitigations:**

  - The calculation cannot overlow or underflow. See context below:

    > `gas` is a uint64, so `(gasUsed * operatorFeeScalar / 1e6) + operatorFeeConstant` can be at most `(u64.max * u32.max / 1e6) + u64.max ~= 7.924660923989131e+22`, which is an int of bit length 77 and fits comfortably within a uint256 variable allocation.

- **Recovery Paths(s):**
  - A bug within the OP EVM would critical and would require an emergency upgrade of the sequencer and bridge.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

**Initial review**

- [ ] (NON-BLOCKING): Coordinate with wallet providers to update their fee estimation logic (assignee: TBD).
- [x] (NON-BLOCKING): Implement automated monitoring on database growth rate (assignee: TBD).

**Testing**

- [x] (BLOCKING): **NAT tests** with Kurtosis in this [PR](https://github.com/ethereum-optimism/optimism/pull/15109)
- [x] (BLOCKING): **_Differential Fuzzing_** with `op-reth`/`op-geth` to avoid any chain-split on the operator fee component (with the refund) -> [PR](https://github.com/ethereum-optimism/optimism/pull/15109).
- [ ] (NON-BLOCKING): A force inclusion transaction from L1 with operator fee and check the that balance are accounted correctly (assignee: @teddyknox)

**Monitoring:**

- [ ] (NON-BLOCKING): **Monitoring** a conservation of balance invariant. If the invariant is broken it should immediately raise alerts (assignee: @clabby @Ethnical)
  - [**PR:** feat(monitorism): Add ETH conservation invariant monitor](https://github.com/ethereum-optimism/monitorism/pull/135) under review.

## Audit Requirements

An audit has not been deemed necessary on **L1** for the relatively simple changes to the SystemConfig contract, which has been reviewed by Base internal security team.

Indeed, the only addition is the `setOperatorFeeScalars` function, which is a setter function that updates the operator fee parameters and trigger an event. This function is callable by the SystemConfig owner only.

An audit has not been deemed necessary right now of this feature.
For more context, the feature is about to be used by a low amount of Chain Operator in the near future.  
Following a conversation with @clabby, @tynes, @teddyknox and @Ethnical the decision was made not to block the release of Isthmus on an audit of the `op-geth` logic touched.
However, to make an audit in parallel of the implementation of the feature in case in near feature of an wide adoption of this feature.
To recap most of the current chains would not have this feature enabled.

Additionally, we are performing multi-client testing (op-geth and op-reth) and will run a multi-client devnet, so bugs in either will be quickly detected by consensus disagreements among them. So, we don't think the operator fee feature falls into the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) Existential + Safety category.
