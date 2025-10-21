# [Operator Fee Fix]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Sudden change in magnitude of fees on chains already using the feature.](#fm1-sudden-change-in-magnitude-of-fees-on-chains-already-using-the-feature)
  - [FM2: Broken Fee Estimation (Wallets)](#fm2-broken-fee-estimation-wallets)
  - [FM3: Discrepancy between client implementations of operator causes a chain fork](#fm3-discrepancy-between-client-implementations-of-operator-causes-a-chain-fork)
  - [FM4: Generic items we need to take into account: `L1Block` badly hydrated](#fm4-generic-items-we-need-to-take-into-account-l1block-badly-hydrated)
  - [FM5: Overflow/Underflow of operator fee calculation](#fm5-overflowunderflow-of-operator-fee-calculation)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                       |                                    |
| --------------------- | ---------------------------------- |
| Author                | geoknee                            |
| Created at            | 15th October 2025                  |
| Initial Reviewers     |                                    |
| Need Approval From    | Matt Solomon                       |
| Need Re-Approval From |                                    |
| Status                | Final                              |

## Introduction

This document covers the Operator Fee Fix feature slated for the Jovian hardfork.

When the fix activates, execution clients switch to a different formula for computing the operator fee from the `scalar` and `constant` (configured via the `SystemConfig` contract as before).
The `GasPriceOracle` predeploy is also updated to reflect the new formula.

Below are references for this project:

- [Design Doc](../protocol/operator-fee-scalar-fix.md)
- [Spec](https://github.com/ethereum-optimism/specs/pull/764)

## Failure Modes and Recovery Paths

### FM1: Sudden change in magnitude of fees on chains already using the feature.

- **Description:**
  Most chains will not set a nonzero operator fee scalar and be unaffected by the consensus change. Any chain which _has_ set a nonzero scalar will be charging only very low fees before the fix activates.
  After activation, the operator fee component would rise, in the worst case up to `10**8` their current value. This is highly undesirable and could cause a denial of service to users until the scalar can be fixed to a lower value.
- **Risk Assessment:**
  Medium impact, low likelihood.
  **Mitigations:**
  We will communicate explicitly that any chain currently using a nonzero `operatorFeeScalar` needs to adjust it _prior_ to the activation of the Jovian fork on their chain.
- **Detection:**
  Block explorers, and operator metrics would reveal the hike in fees and drop in traffic.

- **Recovery Path(s)**:
  If, under the new formula, the operator fee parameters are set to unreasonable values, the rollup operator should update the `operatorFeeScalar` and `operatorFeeConstant` to reasonable values as soon as possible.

  With cast it can be done like this: `cast send <l1_system_config_address> "setOperatorFeeScalars(uint32, uint64)" <operator_fee_scalar> <operator_fee_constant>`

### FM2: Broken Fee Estimation (Wallets)

- **Description:**
  If wallets fail to update their fee estimation logic, users will no longer be shown the accurate costs of a transaction.
- **Risk Assessment:**
  This failure mode can only happen on chains that enable the operator fee feature. Wallets may have updated to the old fee formula, or not at all, and therby be out of date.
  Medium impact, medium likelihood.
  **Mitigations:**
  Coordinate with wallet providers to update their fee estimation logic. This includes MetaMask, Coinbase Wallet, and others.
- **Detection:**
  Using a given wallet, compare the estimated transaction cost with the actual transaction cost, and check if the difference relates to the operator fee, using the formula.
- **Recovery Path(s)**:
  Notify wallets of the new fee structure and ask them to update their fee estimation logic if the operator fee is enabled.


### FM3: Discrepancy between client implementations of operator causes a chain fork

- **Description:**: Since operator fee logic is executed within the L2 EVM, any differences in the two implementations' management of state can trigger a chain fork, where one client believes the state root of a given block to be **X** and another client believes the state root of the same block to be **Y**. Given that the codebase for reth and op-geth look very different, producing code to perform identical calculations down to details such as bit precision, casting and saturation modes, is difficult. Without adaquate testing, it would be difficult to claim that a chain fork would not occur in some operator fee scenarios.
- **Risk Assessment**: Medium impact, high likelihood
- **Mitigation:**
    - Ensure acceptance tests assert no chain fork.

### FM4: Generic items we need to take into account: `L1Block` badly hydrated

- **Description:** At each hardfork, new data can be add to the `L1Block` contract, and the method called to hydrate it change (for instance
  `setL1BlockValuesIsthmus` to `setL1BlockValuesJovian`). If there is a bug in a future method ending up to operator fee params no
  longer being updated in the `L1Block` contract, the operator fee will no longer be taken into account in transactions fee.
- **Risk Assessment:** medium severity / low likelihood
- **Mitigations:**
  The [Operator Fee Constistency](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/operator_fee_test.go) action test runs with all known hardforks activated at genesis, and checks that operator fee parameters are correctly reported to the `L1Block` contract. This has been updated to cover the new formula under Jovian.
- **Detection:**
  The action or E2E tests or local testing may pick up an issue.
- **Recovery Path(s):**
  - If the bug is located in op-node, a new version must be deployed.
  - If the bug is located in the `L1Block` contract, the contract must be upgraded to fix the bug.

### FM5: Overflow/Underflow of operator fee calculation

- **Description:** If the operator fee calculation is vulnerable to overflows or underflows then it could be possible to manipulate the calculation through, say, a deliberately large gasUsed in order to trigger that overflow or underflow and charge a negative fee.
- **Risk Assessment:** high severity / low likelihood
- **Mitigations:**

  - The calculation cannot overlow or underflow. See context below:

    > `gas` is a uint64 in most execution client implementations (and further bounded by the block gas limit), so `(gasUsed * operatorFeeScalar * 100) + operatorFeeConstant` can be at most `(u64.max * u32.max * 100) + u64.max ~= 7.924660923989131e+33`, which is an int of bit length 103 and fits comfortably within a uint256 variable allocation.

- **Recovery Paths(s):**
  - A bug within the OP EVM would critical and would require an emergency upgrade of the sequencer and bridge.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

**Initial review**

- [ ] (NON-BLOCKING): Coordinate with wallet providers to update their fee estimation logic (assignee: TBD).
- [ ] (BLOCKING): Ensure release notice states "any chain currently using a nonzero `operatorFeeScalar` needs to adjust it _prior_ to the activation of the Jovian fork on their chain."

**Testing**

- [ ] (BLOCKING): **Acceptance tests** Ensure they check for chain splits.
- [ ] (BLOCKING): **_Differential Fuzzing_** Check that we still have coverage for `op-reth`/`op-geth` to avoid any chain-split on the operator fee component (with the refund) -> [PR](https://github.com/ethereum-optimism/optimism/pull/15109).
- [ ] (NON-BLOCKING): A force inclusion transaction from L1 with operator fee and check the that balance are accounted correctly


## Audit Requirements

An audit will cover the changes to the L2 contracts and the network upgrade transactions which are derived as a part of the Jovian upgrade.
