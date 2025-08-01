<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Operator Fee Scalar Fix](#operator-fee-scalar-fix)
      - [Metadata](#metadata)
- [Summary](#summary)
- [Problem Statement + Context](#problem-statement--context)
- [Proposed Solution](#proposed-solution)
  - [Alternatives considered](#alternatives-considered)
- [Risks & Uncertainties](#risks--uncertainties)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Operator Fee Scalar Fix

#### Metadata

Authors: @yuwen01.
Created: July 16, 2025.

# Summary

This is the current operator fee formula.

```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed / 1e6
```

The `1e-6` multiplier on the `operatorFeeScalar` is not useful for operators as it results in a fee that is too low to be useful. This design doc proposes the following change to the operator fee formula.

```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed * 100
```

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

This is the current operator fee formula.

```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed / 1e6
```

Where `operatorFeeScalar` is a `u32`, and `operatorFeeConstant` is a `u64`. However, for a 2.5 million gas transaction and a `operatorFeeScalar` of `u32::MAX_VALUE`, the operator fee will be `u32::MAX_VALUE * 2.5e6 / 1e6 ~= 10 gwei`. With an ETH price of 2.5k, this is 0.0000027 USD -- a negligible fee that is not useful for operators.

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

We propose the following change to the operator fee formula.

```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed * 100
```

Where `operatorFeeScalar` and `operatorFeeConstant` remain `u32` and `u64` respectively.

Examine the following to get a sense of the range of fees this formula supports. For the dollar cost column, we assume an ETH price of 2.5k.

| Gas Used | `operatorFeeScalar` | `operatorFeeScalar * gasUsed * 100` | Dollar Cost |
|----------|-------------------|----------------------------------|-------------|
| 30,000,000 | 1 | 3 gwei | $0.0000075 |
| 30,000,000 | 2^32 | 1.29e10 gwei | $32,250.00 |
| 1,000,000 | 1 | 0.1 gwei | $0.00000025 |
| 1,000,000 | 2^32 | 4.29e8 gwei | $1,072.50 |
| 21,000 | 1 | 0.0021 gwei | $0.00000000525 |
| 21,000 | 2^32 | 9.01e6 gwei | $22.53 |

Given that the current scalar component fees are negligible, all `operatorFeeScalar` values are set to 0 upon upgrade. Then, chain operators will be responsible for adjusting them according to the new formula. The `operatorFeeConstant` will not change upon transition.

## Scope of changes

This fix will require changes to the `GasPriceOracle` contract as well as execution clients like `op-geth`, in order to reflect the new operator fee formula.

## Alternatives considered

Instead of changing the scalar value, it is also possible to address this problem by changing the `operatorFeeScalar` to a `u64`. With this solution, we would need to add a new field to the `L1Attributes` transaction, deprecate the `setOperatorFeeScalars` function in the `SystemConfig` contract, and add a new function `setJovianOperatorFeeScalars`. Due to the increased complexity, this solution is not preferred.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

If anyone is using the `operatorFeeScalar` in production, automatically setting it to 0 upon upgrade may cause issues. Since the fee is so low right now, it is likely that any fees lost will be negligible.
