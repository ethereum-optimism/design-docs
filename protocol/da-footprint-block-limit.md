# Jovian: DA Footprint Block Limit

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Sebastian Stammler_                                 |
| Created at         | _2025-08-12_                                         |
| Initial Reviewers  | _TBD_                 |
| Need Approval From | _TBD_                                    |
| Status             | _Draft<!--/ In Review / Implementing Actions / Final-->_ |

## Purpose

Evaluate proposal to add a DA footprint block limit to mitigate DA spam and prevent priority fee auctions from occurring.
This proposal is an alternative to [Custom Calldata Floor Gas](https://github.com/ethereum-optimism/design-docs/pull/294),
avoiding the need to change individual transaction gas mechanics at all.

## Summary

A DA footprint block limit is introduced to mitigate DA spam and prevent priority fee auctions. By tracking DA footprint
of transactions alongside gas, this approach adjusts the block gas limit to account for high estimated DA usage without
altering individual transaction gas mechanics. Preliminary analysis shows minimal impact on most blocks on production
networks like Base or OP Mainnet.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

The current L1 cost model doesn't account for *limited L1 data availability (DA) throughput*. 
This can lead to network congestion for batchers if the throughput on L2s surpasses that of L1 blob space, or if blob
space is congested for other reasons.

Current L1 throughput is 6 blobs per block at target with a max of 9 blobs per block.
This is approx. `128 kB * 6 / 12 s = 64 kB/s` or `96 kB/s`, resp. The L1 cost is proportional to the L1 origin’s base
fee and blob base fee. But Most OP Stack chains have gas targets high enough to accept far more than 128kb of calldata
per (2s) block. So the calldata floor cost doesn't nearly limit calldata throughput enough to prevent.

The current measure to counter high DA throughput that's being deployed on productio networks is batcher throttling.
When a network is throttled at the builder policy-level, however, its base fee can dive and lead to priority fee
auctions and other unwanted user experience issues.

Also see the design doc [Custom Calldata Floor Gas](https://github.com/ethereum-optimism/design-docs/pull/294) for a
more detailed analysis of the same problem and context and a similar solution.

## Proposed Solution

A _DA footprint block limit_ is introduced to limit the total amount of estimated compressed transaction data that can
fit into a block. The base fee update rules take this new total DA footprint into account, so the base fee market isn't
broken like for policy-level throttling solutions.

The main idea is to introduce a new _DA footprint_ resource that is tracked next to gas, and limit it by the same
block gas limit. This is inspired by the
[Multidimensional Gas Metering](https://ethresear.ch/t/a-practical-proposal-for-multidimensional-gas-metering/22668)
design, but the regular gas isn't touched, like in this design for L1 Ethereum.

Similarly to the regular gas usage of transaction calldata, we calculate a transaction’s _DA footprint_ by taking
its RLP-encoding, calculate the compressed transaction DA usage estimation from
[Fjord](https://specs.optimism.io/protocol/fjord/exec-engine.html#fjord-l1-cost-fee-changes-fastlz-estimator)
and multiply it by a factor, the _DA footprint gas scalar_, to get to the transaction’s DA footprint. 

```python
da_usage_estimate = max(minTransactionSize, intercept + fastlzCoef*fastlzSize / 1e6)
da_footprint = da_usage_estimate * da_footprint_gas_scalar
```

Here, `fastlzSize` is the length of the FastLZ-compressed RLP-encoding of a transaction.

Now when evaluating whether a transactions still fits into a block, we take the maximum over the two resources, regular
total gas use of all transactions, and new total DA footprint, and see if this max is still below the block gas limit.
The Multidimensional Gas Metering design introduces a new block field `gas_metered` to differentiate it from `gas_used`
as the sum total of all transaction's gas used. However, it is proposed to just repurpose a block's `gas_used` field to
hold the maximum over both resources' totals:


```python
block.gas_used = max(sum(tx.gas_used), sum(tx.da_footprint if tx.type != DepositTxType))
```

A block's total DA footprint only takes non-Deposit transactions into account, because Deposit transactions aren't part
of batches, so they don't contribute to a block's DA usage.

This definition has the advantage that the base fee update mechanics just keep working, now taking into account high
total block DA footprint.

The DA footprint cost is comparable to the calldata floor cost. If it's set to a higher value, say 400, you
can expect _roughly_ a 1/10 of incompressible calldata to fit into a block compared to only limiting it by
transaction gas usage alone. If the block is mostly low-calldata transactions that use much more regular gas than DA
footprint, this new mechanism wouldn’t have any effect. This is the case for the vast majority of blocks.

### DA footprint gas scalar

I propose to make the DA footprint gas scalar parameter configurable via the `SystemConfig`. The next Ethereum hardfork
[Fusaka](https://eips.ethereum.org/EIPS/eip-7607) introduces
[Blob Parameter Only Hardforks (EIP-7892)](https://eips.ethereum.org/EIPS/eip-7892) and it is hard to estimate how
quickly blob parameters and the number and throughput of OP Stack chains will evolve over the next months, so it is hard
to set a well-informed constant now that will keep being a good constant for the foreseeable future.

However, I propose to set a DA footprint gas scalar value of `800` as a sane default that should have minimal impact on
existing and future chains, while protecting the OP Stack ecosystem from DA spam attacks because throughput of random
data would be reduced by roughly a factor of 20 compared to what is currently allowed by a block's floor gas limit
alone. See also the next section for an analysis of the impact of different gas scalars on various OP Stack chains.

### Impact on OP Stack Chains

We back-tested the estimated impact of a DA footprint block limit on Base, OP Mainnet, WorldChain, Unichain, Ink, Soneium,
Mode with various gas scalars and found that the vast majority of blocks wouldn't be affected, even with a high gas scalar
value of 800. The full analysis can be found in our
[op-analytics](https://github.com/ethereum-optimism/op-analytics/tree/main/notebooks/adhoc/jovian_analysis) repository.

For each chain, the analysis took the following approach. A random 1% of blocks were sampled from a recent 120 day
period (4/29 - 8/26). Then for each block, the DA footprint was estimated by calculating the FastLZ-based DA usage
estimation for each transaction, and then taking the total sum. This was then compared to the block's gas limit and gas
usage to see if the block would have been impacted by a DA footprint limit. If the DA footprint was below the block's gas
usage, then there wouldn't have been any impact, since the new block gas usage would be defined as the max over the
total transaction gas usages and DA footprints. If it went over the gas usage, but not gas limit, then the impact would
only have been that the new block gas usage would have been set higher (so the base fee may have potentially increased
faster), but the block still wouldn't have been limited. Only if the DA footprint also went over the gas limit, would it
have limited that block's transactions to a smaller DA footprint. For those blocks, we also looked at the
distribution of the excess DA footprint as an indicator of how much those blocks would have been limited.

The following tables show, for each analyzed chain, the resulting statistics.
* *Scalar*: DA footprint gas scalar value.
* *Effective Limit*: The DA usage limit that the given gas scalar would imply (`block_gas_limit / da_footprint_gas_scalar`), in estimated compressed bytes.
* *Exceeding Limit*: Number and percent of blocks for wich the total DA footprint exceeds the gas limit. Those blocks
  would have been limited to include fewer transactions.
* *Exc. Gas Usage*: Number and percent of blocks for wich the total DA footprint exceeds the total gas usage. For those blocks, the `block.gas_used`
  field would equal the total block DA footprint instead of the total transaction gas used.
* *Avg. Utilization*: Average utilization of DA footprint resource.
* *Max Utilization*: Maximum utilization of DA footprint resource.

#### OP Mainnet

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/op_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `51,840`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
| -------|-----------------|-----------------|----------------|------------------|-----------------|
| 160    | 250,000         | 0               | 0              | 1.1%             | 44.8%           |
| 400    | 100,000         | 1 (0.00%)       | 3 (0.01%)      | 2.8%             | 112.1%          |
| 600    |  66,666         | 2 (0.00%)       | 17 (0.03%)     | 4.1%             | 168.2%          |
| 800    |  50,000         | 10 (0.02%)      | 47 (0.09%)     | 5.5%             | 224.2%          |

![OP Mainnet (random) DA size estimation distribution](./assets/da-footprint-block-limit/OPM-random-DA-distribution.png)

**[Top 1 Percentile Calldata Usage](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/op_top_percentile_2025-04-29_2025-08-26.html):**
_Sample Size:_ `52,165`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|----------------|------------------|-----------------|
| 160    | 250,000         | 0               | 0              | 1.1%             | 40.0%           |
| 400    | 100,000         | 0               | 1              | 2.7%             | 99.9%           |
| 600    |  66,666         | 4 (0.01%)       | 8 (0.02%)      | 4.1%             | 149.9%          |
| 800    |  50,000         | 17 (0.03%)      | 23 (0.04%)     | 5.5%             | 199.8%          |


#### Base

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/base_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `51,840`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|----------------|------------------|-----------------|
| 160    | 937,500         | 0               | 0              | 2.3%             | 40.7%           |
| 400    | 375,000         | 1 (0.00%)       | 8 (0.0%)       | 5.8%             | 101.8%          |
| 600    | 250,000         | 11 (0.02%)      | 204 (0.4%)     | 8.7%             | 152.8%          |
| 800    | 187,500         | 44 (0.08%)      | 339 (0.7%)     | 11.6%            | 203.7%          |

![Base (random) DA size estimation distribution](./assets/da-footprint-block-limit/Base-random-DA-distribution.png)


**[Top 1 Percentile Calldata Usage](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/base_top_percentile_2025-04-29_2025-08-26.html):**
_Sample Size:_ `52,318`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage  | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|-----------------|------------------|-----------------|
| 160    | 937,500         | 0               | 0               | 4.4%             | 39.2%           |
| 400    | 375,000         | 0               | 21 (0.0%)       | 11.1%            | 98.0%           |
| 600    | 250,000         | 107 (0.20%)     | 399 (0.8%)      | 16.6%            | 147.0%          |
| 800    | 187,500         | 516 (0.99%)     | 1451 (2.8%)     | 22.1%            | 196.1%          |

#### Ink

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/ink_random_2025-04-29_2025-08-26.html):**
_Sample size:_ `103,680`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|----------------|------------------|-----------------|
| 160 | 187,500 | 0 | 0 (0.0%) | 0.3% | 52.7% |
| 400 | 75,000 | 2 (0.00%) | 1480 (1.4%) | 0.8% | 131.9% |
| 600 | 50,000 | 2 (0.00%) | 3534 (3.4%) | 1.2% | 197.8% |
| 800 | 37,500 | 2 (0.00%) | 4203 (4.1%) | 1.7% | 263.7% |

#### Unichain

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/unichain_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `103,679`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage   | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|------------------|------------------|-----------------|
| 160    | 187,500         | 0               | 0                | 0.8%             | 20.5%           |
| 400    |  75,000         | 0               | 407 (0.4%)       | 2.0%             | 51.2%           |
| 600    |  50,000         | 0               | 623 (0.6%)       | 3.0%             | 76.8%           |
| 800    |  37,500         | 1 (0.00%)       | 1111 (1.1%)      | 4.1%             | 102.4%          |


#### Soneium

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/soneium_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `51,840`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|----------------|------------------|-----------------|
| 160 | 250,000 | 0 | 0 (0.0%) | 1.0% | 14.7% |
| 400 | 100,000 | 0 | 20 (0.0%) | 2.5% | 36.8% |
| 600 | 66,666 | 0 | 700 (1.4%) | 3.8% | 55.2% |
| 800 | 50,000 | 0 | 2360 (4.6%) | 5.0% | 73.6% |

#### Mode

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/mode_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `51,840`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage   | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|------------------|------------------|-----------------|
| 160    | 187,500         | 0               | 0                | 0.3%             | 3.1%            |
| 400    |  75,000         | 0               | 50 (0.1%)        | 0.7%             | 7.7%            |
| 600    |  50,000         | 0               | 412 (0.8%)       | 1.0%             | 11.6%           |
| 800    |  37,500         | 0               | 433 (0.8%)       | 1.3%             | 15.5%           |


#### WorldChain

**[Random Sampling](https://htmlpreview.github.io/?https://github.com/ethereum-optimism/op-analytics/blob/main/notebooks/adhoc/jovian_analysis/notebooks/saved_output_html/worldchain_random_2025-04-29_2025-08-26.html):**
_Sample Size:_ `51,840`

| Scalar | Effective Limit | Exceeding Limit | Exc. Gas Usage | Avg. Utilization | Max Utilization |
|--------|-----------------|-----------------|----------------|------------------|-----------------|
| 160    | 225,000         | 0               | 0              | 1.5%             | 19.6%           |
| 400    |  90,000         | 0               | 10 (0.0%)      | 3.8%             | 49.1%           |
| 600    |  60,000         | 0               | 63 (0.1%)      | 5.7%             | 73.7%           |
| 800    |  45,000         | 0               | 78 (0.2%)      | 7.7%             | 98.2%           |

#### Conclusion

It can be seen that, for all chains, for the vast majority of blocks, there would have been no impact since the DA footprint was smaller
than even the gas usage. This is good, because it means that a DA footprint would only impact a small minority of
blocks, and therefore be nearly imperceptible to most users. But since the limit is in place for every block, it would
still protect against worst-case high-throughput incompressible DA spam, which is the objective.

It can furthermore be seen that even a smaller fraction of blocks would have been limited by a DA footprint limit,
showing that under normal usage scenarios (no incompressible DA spam), a DA footprint limit has a neglegible impact on
chains.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

Additional resource usage for block builders or verifiers is considered negligible, because the estimated DA usage of
transactions are already calculated since Fjord and the other calculations are trivial, only involving basic arithmetic.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

* No impact expected on users or tooling simulating individual transactions. In particular, no impact on wallets
  expected.
* The invariant is broken that the sum total of all transactions' _gas used_ is the block's _gas used_.
  If a block's total DA footprint is larger than the total transaction gas usage, the block's gas used field will be set
  to the larger total DA footprint instead. This may impact some analytics-based services like block explorers and those
  services need to be educated about the necessary steps to adapt their services to this new calculation of the block gas
  used.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

* [Custom Calldata Floor Gas](https://github.com/ethereum-optimism/design-docs/pull/294) - introduces customization of
  the calldata floor cost. Comparable effects, but changes gas mechanics of individual transactions.
* [L1 congestion fee](https://github.com/ethereum-optimism/design-docs/pull/312) - only proposal that could partially
  mitigate the L1 inclusion lag but is more complex to implement and harder to model and forecast the impact on
  production networks. It also doesn't set a hard cap on DA footprint throughput, but relies on market incentives
  to drive down calldata use during (modeled) congestion.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
