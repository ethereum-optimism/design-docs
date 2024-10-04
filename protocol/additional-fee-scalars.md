# Additional fee scalars

#### Metadata

Authors: @puma314, @yuwen01.
Created: September 15, 2024.

# Purpose
<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->
Today, the fee formula for the OP stack is computed as `totalFee = gasUsed * (baseFee + priorityFee) + l1Fee` where, the `l1Fee` is computed as follows ([spec](https://github.com/ethereum-optimism/specs/blob/06a2d0b8e5d08da66612d0e19aa7bc625ceb277e/specs/protocol/fjord/exec-engine.md?plain=1#L28)):

```
l1FeeScaled = l1BaseFeeScalar*l1BaseFee*16 + l1BlobFeeScalar*l1BlobBaseFee
estimatedSizeScaled = max(minTransactionSize * 1e6, intercept + fastlzCoef*fastlzSize)
l1Fee = estimatedSizeScaled * l1FeeScaled / 1e12
```

This fee formula presents challenges for variants of the OP stack that leverage alt-DA, ZK proving or a custom gas token. These chains make use of different resources and the existing fee formula is too simplistic to properly charge users for their resource consumption. 

The purpose of this design doc is to propose a simple addition to the existing fee formula (that will be a no-op for most chains, to minimize disruptions), that will allow for greater fee flexibility for the OP stack variants covered above.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

We propose the addition of 2 new rollup operator configured scalars, collectively named the Operator Fee Parameters. These scalars are named
`operatorFeeScalar` and `operatorFeeConstant`, and they factor in the fee calculation as follows:
```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed / 1e6

totalFee = operatorFee + gasUsed * (baseFee + priorityFee) + l1Fee
```

These scalars will be updated via the `SystemConfig` L1 contract. A new role, the `OperatorFeeManager`, is added 
to administrate the `operatorFeeScalar` and `operatorFeeConstant`.

A new fee vault, the `OperatorFeeVault`, is added to store the operator fee. 

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

There are several variants of the OP stack that have trouble with the existing fee formula. We cover them below:

* **Alt-DA:** For chains leveraging alt-DA, the `l1Fee` is not an accurate measure of the DA costs for a transaction, as it is priced according to Ethereum's `blobGas` costs. These costs are entirely separate from the pricing of another DA layer.
* **ZK Proving:** For OP Stack variants that want to utilize ZK proofs (for example via [OP-Succinct](https://github.com/succinctlabs/op-succinct)), the cost of ZK proving a transaction is a significant resource that is not taken into consideration in the current fee structure.
* **Custom gas token:** For chains using a custom gas token, the `l1Fee` is priced in ETH, but transaction fees are paid in another token. 


# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

We propose the addition of two new rollup operator-configured scalars, collectively referred to as the **operator fee parameters**. These scalars, the `operatorFeeScalar` and `operatorFeeConstant`, play a role in calculating fees as follows.

## Operator Fee Formula

```
operatorFee = operatorFeeConstant + operatorFeeScalar * gasUsed / 1e6

totalFee = operatorFee + gasUsed * (baseFee + priorityFee) + l1Fee
```

The `operatorFeeScalar` will be a `u32`, scaled by 1e6 in a similar fashion to the `baseFeeScalar` and `blobBaseFeeScalar`. The `operatorFeeConstant` will be a `u64`.

The `operatorFee` will be directed to a new `FeeVault`, the `OperatorFeeVault`, in a similar fashion to the way the existing `l1Fee` is being directed to the `l1FeeVault`.

## Setting the Operator Fee Scalars

These 2 new config values can be added to the [`L1 attributes`](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/ecotone/l1-attributes.md) transaction, with a very small diff to the proof.

We expose a new function in the `SystemConfig` contract for updating the `operatorFeeScalar` and `operatorFeeConstant`. This function is only callable by the `OperatorFeeManager`, a new role responsible for tuning the operator fee scalars. The `OperatorFeeManager` is administrated by
the chain governor. 

```solidity
function setOperatorFeeScalars(uint32 operatorFeeScalar, uint64 operatorFeeConstant) external onlyOperatorFeeManager;
```

This function will emit a `ConfigUpdate` log-event, with a new `UpdateType`: `UpdateType.OPERATOR_FEE_SCALARS`.

In order to ensure a smooth network upgrade process, these scalars are automatically set to zero at the start of the upgrade. 

The operator fee manager is automatically set to the chain governor at the start of the upgrade. The chain governor administers
the operator fee manager role. 

## Analysis

* **Alt-DA**: For chains using alt-DA, the `operatorFeeConstant` is useful if there is a relatively constant overhead to posting to another DA layer. 
* **ZK Proving**: For chains with ZK proving, ZKP generation costs are usually proportional to `gasUsed` with a fixed overhead per transaction, making use of both the `operatorFeeConstant` and the `operatorFeeScalar`.
* **Custom gas token**: For chains with a custom gas token, both operator fee scalars can be useful as a ratio between the cost of the gas token and ETH to balance costs vs. the fee computation.

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The L1Attributes deposited transactions includes one extra slot of calldata -- the two new scalars,
storage packed together.

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

Alternatives include [this proposal](https://github.com/ethereum-optimism/specs/issues/73) for fee abstraction. This proposal is a much higher lift, but fully generalizes the fee mechanism to be fully customizable by any team. However, it increases node resource costs because it includes additional EVM execution for each transaction in computing the fee and must be extensively benchmarked to ensure that its resource consumption is not too high.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

There are perhaps other scalars that could be desireable to add in this change that would be simple lift before we go to the final endgame of fee abstraction.