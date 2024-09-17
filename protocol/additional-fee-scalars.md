# Additional fee scalars

#### Metadata

Authors: @puma314.
Created: September 15, 2014.

# Purpose
<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->
Today, the fee formula for the OP stack is computed as `fee = gas_used * (base_fee + priority_fee) + l1Fee` where, the `l1Fee` is computed as follows ([spec](https://github.com/ethereum-optimism/specs/blob/06a2d0b8e5d08da66612d0e19aa7bc625ceb277e/specs/protocol/fjord/exec-engine.md?plain=1#L28)):

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

We propose the addition of 2 new rollup operator configured scalars (that can be updated via the `SystemConfig`) named `gas_used_scalar` and `constant_scalar` that factor in the fee calculation as follows:
```
fee = constant_scalar + gas_used * (base_fee + priority_fee + gas_used_scalar) + l1Fee
```

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

We propose the addition of 2 new rollup operator configured scalars (that can be updated via the `SystemConfig`) named `gas_used_scalar` and `constant_scalar` that factor in the fee calculation as follows:
```
fee = gas_used * (base_fee + priority_fee) + l1Fee + gas_used_scalar * gas_used + constant_scalar
```

These scalars should be `u256`, so as to be on the same order of magnitude as the other terms in the fee calculation. These 2 new config values can be added to the [`L1 attributes`](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/ecotone/l1-attributes.md) transaction, with a very small diff to the proof.

The chain operator will have control over these values (similar to `baseFeeScalar` and `blobBaseFeeScalar`) and these values will be set and updated in the exact same way as the existing `scalar` attributes.

**Analysis**

* **Alt-DA**: For chains using alt-DA, the `constant_scalar` is useful if there is a relatively constant overhead to posting to another DA layer. 
* **ZK Proving**: For chains with ZK proving, ZKP generation costs are usually proportional to `gas_used` with a fixed overhead per transaction, making use of both the `constant_scalar` and the `gas_used_scalar`.
* **Custom gas token**: For chains with a custom gas token, both scalars can be useful as a ratio between the cost of the gas token and ETH to balance costs vs. the fee computation.


## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

There are basically no additional resources used by this proposal. 

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

Alternatives include [this proposal](https://github.com/ethereum-optimism/specs/issues/73) for fee abstraction. This proposal is a much higher lift, but fully generalizes the fee mechanism to be fully customizable by any team. However, it increases node resource costs because it includes additional EVM execution for each transaction in computing the fee and must be extensively benchmarked to ensure that its resource consumption is not too high.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

There are perhaps other scalars that could be desireable to add in this change that would be simple lift before we go to the final endgame of fee abstraction.