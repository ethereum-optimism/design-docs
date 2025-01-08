# [Operator Fee]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Operator Fee scalars are set to incorrect values](#fm1-operator-fee-scalars-are-set-to-incorrect-values)
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

Below are references for this project:

- [Design Doc](../protocol/operator-fee.md)
- [Spec](https://github.com/ethereum-optimism/specs/pull/382)

## Failure Modes and Recovery Paths

### FM1: Operator Fee scalars are set to incorrect values

- **Description:** 
  If the operator fee scalars are incorrectly initialized or updated, there is a risk that the transcations fees will be too high. This could lead to a situation where the chain become unusable.
- **Risk Assessment:** High impact, low likelihood.
  **Mitigations:**
  Every operator fee scalars update should be carefully tested and reviewed before deployment.
- **Detection:** 
  Monitoring gas cost estimation.
- **Recovery Path(s)**:
  If the operator fee parameters are set to unreasonable values, the rollup operator should update the `operatorFeeScalar` and `operatorFeeConstant` to reasonable values as soon as possible.

### Generic items we need to take into account:

See [./fma-generic-hardfork.md](./fma-generic-hardfork.md).

- [X] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] _Action item 2 (Assignee: tag assignee)_
- [ ] _Action item 3 (Assignee: tag assignee)_
