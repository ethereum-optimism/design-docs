# L1 Congestion fee and builder reordering or throttling

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Sebastian Stammler                                 |
| Created at         | 2025-07-30                                         |
| Initial Reviewers  | George Knee, Protolambda, Niran Babalola           |
| Need Approval From | Matt Slipper                                       |
| Status             | _Draft_ <!-- _Draft / In Review / Implementing Actions / Final_ -->|

## Purpose

Purpose is to evaluate congestion-tracking-based in- and out-of-protocol solutions to the
**L1 Inclusion Information Lag** and **Limited Blob Throughput** problems, and how they relate
to DA spam and PFAs.

This is part of the overall design work for the upcoming Jovian hardfork.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

The current L1 cost model doesn't account for *limited L1 data availability (DA) throughput* and suffers from an *L1
inclusion information lag*. This can lead to network congestion and batchers overpaying during high-demand periods
without adequate economic signals to regulate transaction flow.

- **L1 Inclusion Information Lag:** users only pay L1 fees based on the (blob) base fee of the past L1 origin block,
while the batcher may pay significantly more when submitting the batch containing user transactions. This is a known
problem that has generally been treated as a low-severity issue because under normal usage patterns, **on average,**
users will pay an L1 cost that roughly matches the batcher’s costs.
- **Limited Blob Throughput:** Current L1 throughput is 6 blobs per block at target with a max of 9 blobs per block.
This is approx. `128 kB * 6 / 12 s = 64 kB/s` or `96 kB/s`, resp. The L1 cost is proportional to the L1 origin’s base
fee and blob base fee.

Two other designs that are currently being discussed to addres deficiencies in our fee market are
* [Minimum Base Fee](./minimum-base-fee.md)
* [Confgurable Calldata Gas](https://github.com/ethereum-optimism/design-docs/pull/294) (still being evaluated)

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

> [!NOTE]
> The following sections will, for simplicity, first describe the in-protocol L1 congestion fee proposal and then
later describe how the same mechanism can be used to define an out-of-protocol throttling mechanism at the builder.

### Overview

I propose adding an additional *L1 congestion fee* component to the *L1 cost* that responds dynamically to aggregated DA
usage. It is similar to the blob fee on L1 and actually aims to extend it at the L2 level. The mechanics would thus be
similar: tracking excess DA usage over a target and then setting an exponential-proportional L1 congestion fee
multiplier of the L1 cost, based on the FastLZ-based estimated DA size (not gas usage) which is already calculated since
Fjord for the L1 cost. This new congestion fee component would generate fast economic feedback to DA usage spikes and
render sustained DA spam attacks unfeasible to mount. At the same time, during normal DA usage, the congestion fee
multiplier would be 1, or very close to 1, so this new cost shouldn’t affect chains during normal usage patterns.

*How does this solution address the two main problems?*

- **L1 Inclusion Information Lag:** The congestion fee would project the increased cost for the batcher to get excess DA
usage submitted to L1. Although this is an imperfect model-based projection, it would still create the necessary
economic back-pressure against heavy DA usage. In fact, this back-pressure would occur earlier than when L1 transaction
inclusion becomes difficult due to increased batcher load, as the target would be set lower than the L1 blob throughput
target.
- **Limited Blob Throughput:** While the congestion fee wouldn’t theoretically fully prevent a chain’s DA throughput to
exceed the theoretical maximum L1 blob throughput, by nature of the exponential-proportionality it would become
exponentially more expensive to keep such excess traffic up, so any rational actor with limited resources will stop at
some point. And the right parameter choices can make sure this point is early enough, e.g. well before sequencing window
elapse.
    - It’s still important that batcher throttling is kept in place as a fail-safe, see separate section below.

### Specification

#### Overview

 The high-level idea is that we extend the current L1 cost by a factor that projects the added cost that the batcher may
 pay to get excess DA usage submitted to L1. And this is done naturally as a factor to the current L1 cost, as we will
 see shortly.

Recall the Fjord L1 fee function:

```python
l1FeeScaled = l1BaseFeeScalar*l1BaseFee*16 + l1BlobFeeScalar*l1BlobBaseFee
estimatedSizeScaled = max(minTransactionSize * 1e6, intercept + fastlzCoef*fastlzSize)
l1FeeFjord = estimatedSizeScaled * l1FeeScaled / 1e12
```

So writing the equation in a more abstract way, without scaling and ignoring the base fee for a moment (the
`l1BaseFeeScalar` is a very low number on most chains nowadays), the L1 fee is 

```python
l1_fee_fjord = l1_blob_fee_scalar * l1_blob_base_fee * estimated_size
```

The proposal is now to extend the L1 cost to 

```python
l1_fee = l1_fee_fjord * congestion_fee_multiplier
= estimated_size * l1_blob_fee_scalar * e**(l1_excess_blob_gas/BLOB_BASE_FEE_UPDATE_FRACTION) * e**(excess_da_bytes/CONGESTION_FEE_UPDATE_FRACTION)
= estimated_size * l1_blob_fee_scalar * e**(l1_excess_blob_gas/BLOB_BASE_FEE_UPDATE_FRACTION + excess_da_bytes/CONGESTION_FEE_UPDATE_FRACTION)
```

So it’s as if the chain-locally-tracked `excess_da_bytes` are (up to a factor
`BLOB_BASE_FEE_UPDATE_FRACTION/CONGESTION_FEE_UPDATE_FRACTION`) added to the L1 `excess_blob_gas`, as if projecting that
(worst-case) that’s what the batcher will pay in increased blob costs because of excess DA usage on the L2. And
`l1_excess_blob_gas` is almost the same as the number of excess bytes stored in the excess blobs (noting that blob gas
maps almost 1:1 to encoded bytes in a full blob).

 > [!NOTE]
 > In an earlier version, the congestion fee was an additive new component of the L1 cost. But this would be much
less optimal since it would always only start to rise from `1 wei` . Because of the way the L2 excess is added inside
the exponential to the L1 blob excess, it seems more natural to make this a modifying factor.

#### Congestion fee tracking and update rules

We reuse or repurpose the following header fields, definitions and functions from the
[EIP-4844](https://eips.ethereum.org/EIPS/eip-4844) specification:

- We reuse the header fields `blob_gas_used` and `excess_blob_gas` and also call them `da_bytes_used` and
  `excess_da_bytes`, respectively.
- We redefine `TARGET_BLOB_GAS_PER_BLOCK` as `TARGET_DA_BYTES_PER_BLOCK`
- We redefine `BLOB_BASE_FEE_UPDATE_FRACTION` as `CONGESTION_FEE_UPDATE_FRACTION`
- We reuse `fake_exponential` - numerically approximates the exponential function using Taylor expansion.
- We redefine `calc_excess_blob_gas` as `calc_excess_da_bytes`:
    
    ```python
    def calc_excess_da_bytes(parent: Header) -> int:
        if parent.excess_da_bytes + parent.da_bytes_used < TARGET_DA_BYTES_PER_BLOCK:
            return 0
        else:
            return parent.excess_da_bytes + parent.da_bytes_used - TARGET_DA_BYTES_PER_BLOCK
    ```
    
    - Note that for the first post-fork block, both `parent.blob_gas_used`  and `parent.excess_blob_gas` are already set to `0`  per the Ecotone specs.
- We borrow the transaction size estimation by FastLZ size from the Fjord specs:
    
    ```python
    def get_estimated_size(tx: Transaction) -> int:
    		return max(minTransactionSize, (intercept + fastlzCoef*tx.fast_lz_size)/1e6)
    ```
    
    - `tx.fast_lz_size` is the `fast_lz_size` of a transaction as it is already calculated since Fjord.
    - We already scale the size estimation back by dividing by `1e6`

We can now define the `l1_fee` function using the `fake_exponential` and the Fjord L1 fee function `l1_fee_fjord`:

```python
def l1_fee(header: Header, tx: Transaction) -> int:
    return fake_exponential(
        l1_fee_fjord(header, tx),
        header.excess_da_bytes,
        CONGESTION_FEE_UPDATE_FRACTION
    )
```

This calculation leads to a better precision than calculating `l1_fee = l1_fee_fjord * congestion_fee_multiplier` with
the `congestion_fee_multiplier` using the `fake_exponential`  only returning integer values, but is otherwise
conceptually equivalent (if the multiplier was a float instead).

Similarly to EIP-4844, at the start of a new block, its `excess_da_bytes` field is updated to
`calc_excess_da_bytes(parent)`  and each transaction’s L1 cost is calculated using `l1_cost`. The header field
`da_bytes_used` sums up all block transactions’ size estimations from `get_estimated_size(tx)`.

The fee mechanics of the L1 cost also stay the same otherwise, also keeping in line with EIP-4844 blob-fee behavior: the
full `l1_cost` is deducted from the sender balance before transaction execution and burned, and is not refunded in case
of transaction failure.

#### New SystemConfig Parameters

I suggest to make the parameters `TARGET_DA_BYTES_PER_BLOCK` and `CONGESTION_FEE_UPDATE_FRACTION` updatable via the
`SystemConfig`. An open question is whether we set default values at the fork block or whether the feature is off by
default (i.e. having a constant congestion fee multiplier of 1).

> [!NOTE]
> An alternative is to make these parameters dependent on the block gas limit and target, but otherwise not
configurable, to avoid a tragedy of the commons scenario where chain operators could be incentiviezed to raise the
target to get more cheap throughput at the rollup ecosystem's expense of higher blob usage.

#### DA Usage Approximation

It's worth noting that the `get_estimated_size`, based on `fast_lz_size`, is only an approximation of the actual DA
usage that a transaction will have in its batches. This has been analyzed in detail when the FastLZ-based
compressed-size approximation was introduced with Fjord, and it was found to almost always slightly over-estimates the
compressed size. It thus seems like an acceptable approximation of DA-usage and fine-tuning of parameters should lead to
a well-working congestion fee mechanism that properly estimates the actual data usage on L1.

#### Implementation Considerations

We can reuse the currently unused (always 0) `blob_gas_used` and `excess_blob_gas` header fields to track DA usage and
excess. This makes conceptually sense as the congestion fee is similar to the blob fee on L1 and actually aims to mirror
it at the L2 level. If there’s a worry that we may want to introduce L2 blobs in the future, new fields could also be
added. But semantically, it would be a good fit, also taking into account that blob gas really is an approximation of
the number of blob bytes (`GAS_PER_BLOB` is `2**17 = 131,072` which is the byte size of a blob, and a blob can hold
almost that amount of bytes of data).

### Parameter Selection

Remember that the parameters `TARGET_DA_BYTES_PER_BLOCK`  and `CONGESTION_FEE_UPDATE_FRACTION` are suggested to be
updatable via the `SystemConfig`, if not be defined in terms of the block gas target and limit.

#### Target DA Bytes Per Block

The `TARGET_DA_BYTES_PER_BLOCK` parameter needs to be set per chain to the target DA throughput on that chain, that is,
the maximum da-bytes per block that should be sustained for longer periods of time (on average). An ideal target is
higher than what a chain observes most of the time, but low enough that if sustained DA-spam (or just genuine DA-heavy
transactions) were to occur, the congestion fee would quickly rise and create economic back-pressure for DA-heavy
transactions.

We would probably need to write a tool analyze the distribution of DA usage of a chain over its blocks in order to
choose good values.

**Ecosystem-wide Throughput**

Since all rollups share the same L1 DA layer, ideally those targets are chosen in a coordinated fashion so that the sum
of all targets stays below the L1 blob throughput target of currently `64 kB/s`, or at least be a low multiple of it.
Additionally, non-OP Stack rollups have to be taken into account too.

Note that for this reason, the target selection suffers from the Tragedy of the Commons. Chain operators could be
incentivized to set higher targets than would be good for the community to offer more throughput more cheaply. So the
alternaive to define them in terms of the block gas target and limit is a viable alternative.

#### Update Fraction (rate of change)

The parameter `CONGESTION_FEE_UPDATE_FRACTION`controls the maximum rate of change of the congestion fee per DA byte. It
can be used to achieve an approximate maximum change rate of `e**(TARGET_DA_BYTES_PER_BLOCK /
CONGESTION_FEE_UPDATE_FRACTION)` per block. In EIP-4844, the update fraction is set so that the max change rate is set
to be `~1.125`. Further analysis and simulations are required to learn if this is a good value. Note that since L2s have
generally a shorter block time, default of 2s currently, the change rate in absolute time is quicker than on L1.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The additional resource usage of any congestion-excess-tracking-based solution is considered negligible, because the
estimated DA sizes of transactions are already calculated since Fjord and the other calculations are trivial, only
involving basic arithmetic and evaluation of the `fake_exponential`.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->
TODO (probably irrelevant)

## Further Analysis

### Limitations

#### Varying throughput target

This mechanism works well if a good throughput target can be defined for a chain. However, if a chain varies its
throughput over time, the target has to be regularly updated. And as discussed before, the ecosystem-wide throughput
should be taken into account when assigning targets to individual chains.

Ideally, we could define a dynamic mechanism for a self-adjusting throughput target. For the sake of discussion, it’s
currently set to a constant. For example, derivation could observe a moving average throughput, e.g. over a week, as a
basis for a more dynamic target. But this would have other drawbacks.

#### **Batcher throttling is still required**

Because a chain operator can only control spam on its own chain, if there starts to appear L1 blob spam for any other
reasons, e.g. a spam attack on another rollup (or set of rollups) that doesn’t have throttling or congestion fees in
place, it is still necessary to have other mitigations against spam like batcher throttling in place, which operates on
the actual observable quantity of L1 backlog, which is the ultimate variable that we want to protect with all those
mitigations.

### Relationship to Throttling

This new cost component is closely related to throttling because it’s another mechanism that influences data-throughput and block building. Batcher-induced throttling of the sequencer, as is currently used by most production chains, works by observing the backlog of data that still needs to be submitted to L1 and then throttle the sequencer following some rules. The current default rule is a dumb cut-off throttling switch that enables strong throttling if the backlog exceeds, say, 10 MB. Recently, additional throttling rules have been added, namely, linear, quadratic and an experimental PID-like throttling controller.

Throttling can activate rather late and affect the chain in other unfavorable ways. But it is still an important feature
to stay, as it acts as a fail-safe if there are other reasons why the batcher can’t get blobs submitted to L1, e.g. due
to connection errors, or because other rollups or applications cause blob contention, or because an economically
irrational actor is ready to generate spam at hefty cost. It should be viewed as a complementary feature to an
in-protocol congestion fee because it acts on the actually observed value of L1 backlogged data rather than economic
models and assumptions on rational behavior.

It’s also important to note that both, throttling and the congestion fee parameters need to be set to compatible values.
If batcher throttling parameters are too sensitive, the batcher will always induce throttling on the sequencer before
the congestion fee even has a chance to create economic back-pressure on DA-heavy transactions.

## Block Builder Congestion-based Reordering/Throttling

The congestion fee mechanism described here can already be used outside a hardfork to reorder/delay transactions during
block building, to mitigate DA-spam induced congestion at the block builder policy level.

One implementation of this could calculate the congestion fee multiplier for each transaction and then *divide* the L1
cost component of the transaction by it to get to a new virtual value of the transaction and then consider this virtual
value when ordering transactions by value for block inclusion. This would naturally start to punish heavy-DA
transactions over light-DA transactions as target-excess increases. However, one major drawback is that often there
isn't actually a queue of transactions, so just reordering may not help much in reality.

Another option is to calculate the L1 congestion component for each transaction as
`l1_cost(tx) * (1 - congestion_fee_multiplier(tx))` and then require the priority fee to cover this amount, possibly
only over some threshold so this would only target DA-heavy transactions while still acceping 0 priority fees
for DA-light transactions. Of course, this would directly invite PFAs, which we're trying to mitigate, but only
for DA-heavy transactions, where the argument could be made that until the in-protocol fees reflect the real cost of
transactions, this is an acceptable tradeoff. Note that even on L1, the comparable blob fee market sometimes observes PFAs
too because the blob base fee is so low that it can be ineffective for hours during congestion.

A third option is to use the excess calculation and a transaction's virtual congestion cost to implement some form of
filtering. This would be very similar to batcher throttling, which filters out transactions based on their DA size, so
it also comes with the same drawbacks.

In general, any builder-policy-based solutions suffers from the same problem, that an artificial contriction on blocks
may lead to gas-light blocks and thus a decreasing base fee, breaking the fee market, and introducing PFAs.


## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

### Batch inclusion lag-based fee

Instead of the modeling approach, tracking excess DA over a target, we could also directly observe the batch
inclusion lag in-protocol and use it as a basis for a congestion fee. E.g. the observable variable could be the time
lag between the newest L2 (or L1 origin) timestamp in a batch and the L1 timestamp that the batch got included. The
priority fee for that transaction could also be taken into account. A time-size-weighted average over the last batches
could also be used.

Such a congestion fee would have the benefit that it is more directly based on the variable we actually care about.
However, there would still be an information lag. A chain operator could also exploit this mechanism to drive up fees by
just delaying batch submission. However, this would worsen the UX and benign chain operators wouldn’t have an interest
in doing so to keep their chain attractive. A chain operator can also, at any point in time, just increase their profit
margin by reconfiguring the fee params in the `SystemConfig`  (which, on the other hand, would be more obvious than
batch submission delaying).

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
