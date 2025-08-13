# Jovian: Calldata Footprint Block Limit

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Sebastian Stammler_                                 |
| Created at         | _2025-08-12_                                         |
| Initial Reviewers  | _TBD_                 |
| Need Approval From | _TBD_                                    |
| Status             | _Draft<!--/ In Review / Implementing Actions / Final-->_ |

## Purpose

Evaluate proposal to add a calldata block limit to mitigate DA spam and prevent priority fee auctions from occurring.
This proposal is an alternative to [Custom Calldata Floor Gas](https://github.com/ethereum-optimism/design-docs/pull/294),
avoiding the need to change individual transaction gas mechanics at all.

## Summary

A calldata footprint block limit is introduced to mitigate DA spam and prevent priority fee auctions. By tracking
calldata footprint alongside gas, this approach adjusts the block gas limit to account for high calldata usage without
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

A _calldata footprint block limit_ is introduced to limit the total amount of calldata that can fit into a block. The
base fee update rules take this new total calldata footprint into account, so the base fee market isn't broken like
for policy-level throttling solutions.

The main idea is to introduce a new _calldata footprint_ resource that is tracked next to gas, and limit it by the same
block gas limit. This is inspired by the
[Multidimensional Gas Metering](https://ethresear.ch/t/a-practical-proposal-for-multidimensional-gas-metering/22668)
design, but the regular gas isn't touched, like in this design for L1 Ethereum.

Similarly to the regular gas usage of transaction calldata, we calculate a transaction’s _calldata footprint_ by taking
its calldata, calculate the transaction size estimation from
[Fjord](https://specs.optimism.io/protocol/fjord/exec-engine.html#fjord-l1-cost-fee-changes-fastlz-estimator)
and multiply it by a factor, the _calldata footprint cost_, to get to the transaction’s calldata footprint. 

```python
size_estimate = max(minTransactionSize, intercept + fastlzCoef*fastlzSize / 1e6)
calldata_footprint = size_estimate * calldata_footprint_cost
```

Now when evaluating whether a transactions still fits into a block, we take the maximum over the two resources, regular
total gas use of all transactions, and new total calldata footprint, and see if this max is still below the block gas
limit. The Multidimensional Gas Metering design introduces a new block field `gas_metered` to differentiate it from
`gas_used` as the sum total of all transaction's gas used. However, it is proposed to just repurpose a block's
`gas_used` field to hold the maximum over both resources' totals:


```python
block.gas_used = max(sum(tx.gas_used), sum(tx.calldata_footprint))
```

This also has the advantage that the base fee update mechanics just keep working, now taking into account high block
calldata footprint.

The calldata footprint cost is comparable to the usual calldata floor cost. If it's set to a higher value, say 400, you
can expect roughly a 1/10 of incompressible calldata to fit into a block compared to only limiting it by
transaction gas usage alone. If the block is mostly low-calldata transactions that use much more regular gas than calldata
footprint, this new mechanism wouldn’t have any effect. This is the case for the vast majority of blocks.

### Impact on OP Stack Chains

We did some initial analysis of the impact on Base and OP Mainnet and found that the vast majority of blocks
wouldn't be affected by the introduction of a calldata footprint limit.

```
BASE - 2025-07-22
💰 Cost Comparison Analysis (43200 blocks):
==================================================
Cost   Avg Util   Max Util   Exceeds  
--------------------------------------
160    2.21%      27.50%     0/43200  
400    5.52%      68.75%     0/43200  
800    11.04%     137.49%    13/43200 


OPM - 2025-08-01 
💰 Cost Comparison Analysis (1000 blocks):
==================================================
Cost   Avg Util   Max Util   Exceeds
------------------------------------
160    0.37%      9.83%      0/1000 
400    0.93%      24.59%     0/1000 
800    1.87%      49.17%     0/1000 
```

More analysis are added here soon.

### Calldata footprint cost parameter

It's an open question whether we should just set a constant calldata footprint cost or make it configurable via the
`SystemConfig`, like the Custom Calldata Floor Cost design proposes. A protocol constant is simpler and leads to a more
uniform experience across different OP Stack chains, so it would be my preferred option.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

Additional resource usage for block builders or verifiers is considered negligible, because the estimated DA sizes of
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
  If the calldata footprint limit was reached first during block building, the gas used sum will be smaller as the
  block's gas used field will be the total calldata footprint instead. This may impact some analytics-based services
  like block explorers and those services need to be educated about the necessary steps to adapt their services to this
  new calculation of the block gas used.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

* [Custom Calldata Floor Gas](https://github.com/ethereum-optimism/design-docs/pull/294) - introduces customization of
  the calldata floor cost. Comparable effects, but changes gas mechanics of individual transactions.
* [L1 congestion fee](https://github.com/ethereum-optimism/design-docs/pull/312) - only proposal that could partially
  mitigate the L1 inclusion lag but is more complex to implement and harder to model and forecast the impact on
  production networks. It also doesn't set a hard cap on calldata footprint throughput, but relies on market incentives
  to drive down calldata use during (modeled) congestion.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
