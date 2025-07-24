# Minimum Base Fee

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | @niran                                             |
| Created at         | 2025-07-01                                         |

---

# Purpose
 
This document proposes establishing a configurable minimum base fee for OP Stack chains to shorten the length of priority fee auctions.

# Summary

We introduce a new configuration value that can be modified via `SystemConfig`:

| Name             | Type    | Default | Meaning |
|------------------|---------|---------|---------|
| `minBaseFeeLog2` | `uint8` | `0`     | The log2 of the minimum base fee that can be charged for a transaction. 0 disables the minimum base fee entirely. |

These values are updated via a new function in `SystemConfig`:

```solidity
function setMinBaseFeeLog2(uint8 minBaseFeeLog2) external onlyOwner;
```

This function will emit a `ConfigUpdate` log-event, with a new `UpdateType`: `UpdateType.MINIMUM_BASE_FEE`.

Like [Holocene's dynamic EIP 1559 parameters](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/holocene/exec-engine.md#eip-1559-parameters-in-block-header), the minimum base fee is included in the block header's `extraData` field.

## Minimum Base Fee in Block Header

The `extraData` header field of each block must have the following format:

| Name             | Type               | Byte Offset |
| ---------------- | ------------------ | ----------- |
| `version`        | `u8`               | `[0, 1)`    |
| `denominator`    | `u32 (big-endian)` | `[1, 5)`    |
| `elasticity`     | `u32 (big-endian)` | `[5, 9)`    |
| `minBaseFeeLog2` | `u8`               | `[9, 10)`   |

Additionally,

- `version` must be 1 (incremented from 0)
- `denominator` must be non-zero
- there is no additional data beyond these 10 bytes

Note that `extraData` has a maximum capacity of 32 bytes (to fit in the L1 beacon-chain `extraData` data-type) and its
format may be modified/extended by future upgrades.

## Minimum Base Fee in `PayloadAttributesV3`

The [`PayloadAttributesV3`](https://github.com/ethereum-optimism/specs/blob/a773587fca6756f8468164613daa79fcee7bbbe4/specs/protocol/exec-engine.md#extended-payloadattributesv3)
type is modified to add an extra byte for `minBaseFeeLog2` to the `eip1559Params` field:

```rs
PayloadAttributesV3: {
    timestamp: QUANTITY
    prevRandao: DATA (32 bytes)
    suggestedFeeRecipient: DATA (20 bytes)
    withdrawals: array of WithdrawalV1
    parentBeaconBlockRoot: DATA (32 bytes)
    transactions: array of DATA
    noTxPool: bool
    gasLimit: QUANTITY or null
    eip1559Params: DATA (9 bytes) or null
}
```

# Problem Statement + Context

Ethereum's EIP 1559 has successfully made first-price auctions for transaction fees rare. Users used to always have to set a fee that was high enough to outbid all other transactions to be included in a block. Now they can set a maximum fee that is high enough to cover the base fee with a margin for base fee increases caused by congestion and expect to be included in a block without ever having to outbid other transactions. In some conditions, Ethereum-based chains still experience first-price auctions via priority fees, and this happens more often on OP Stack chains. During these periods, many users experience a denial of service. They submit transactions that they expect to be included in a block but they are not included for long periods of time (and may be dropped from the mempool entirely).

Priority fee auctions occur when blocks are near the gas limit or when the base fee is significantly below the market-clearing fee. Other than full blocks, the most common cause of priority fee auctions today is [batcher sequencer throttling](https://docs.optimism.io/operators/chain-operators/configuration/batcher#batcher-sequencer-throttling), which intentionally processes fewer transactions to avoid a backlog of batches that need to be posted to L1. Similarly, if the sequencer runs out of time to execute transactions for a block, the transactions that get included will be the ones with the highest priority fees.

It can also be argued that **scaled** rollups will run well below their gas target in typical conditions, which makes the "gas target" more of a "congestion threshold." Ethereum's base fees are _congestion_ fees, and scaled rollups should rarely be congested: their maximum capacity should be much higher than typical usage. A minimum base fee would be charged in the vast majority of cases, and higher fees would only be charged when the rollup needs to discourage transactions near its capacity. In this scenario, we would need to think of congestion often starting from base fees that are the lowest the chain will allow. Today, that's 1 wei, which takes far too long to adjust to the market prices that are needed to prevent first-price auctions using priority fees.

When the base fee is at or near its minimum value, it can only increase as fast as the `BASE_FEE_MAX_CHANGE_DENOMINATOR` of EIP 1559 allows. Ethereum has this value set to `8`. The maximum the base fee can increase in one block is `(ELASTICITY_MULTIPLIER - 1) / BASE_FEE_MAX_CHANGE_DENOMINATOR`, which is `(2 - 1) / 8 = 1/8` or 12.5%. That produces a doubling time of 5.9 blocks at 12 seconds each, or 71 seconds. On OP Mainnet, the denominator is `250`, which produces a maximum increase per block of 0.4% and a doubling time of 17.7 blocks at 2 seconds each, or 35be seconds. On Base, the denominator is `50`, which produces a maximum increase per block of 2% and a doubling time of 35 blocks at 2 seconds each, or 70 seconds. Base's elasticity was recently changed from `2` to `3`, which gives a maximum increase per block of 4% and a doubling time of 17.7 blocks at 2 seconds each, or 35 seconds.

The current prevailing base fee on Base today is slightly below 0.001 gwei. It takes 411 blocks (13.7 minutes) for the base fee to increase from 1 wei to 0.001 gwei if every block is completely full at the gas limit. (In practice, we see 70% full blocks in these scenarios, so the fees take even longer to increase.) If we could configure a minimum base fee of 0.0001 gwei, we could speed up the fee adjustment (and the corresponding end of the priority fee auction) to 59 blocks, or a little under two minutes. We could even eliminate the priority fee auctions that occur while the chain is adjusting from the minimum to the typical market fee price by setting the minimum to that price to begin with. [Arbitrum One sets a minimum of 0.01 gwei](https://docs.arbitrum.io/how-arbitrum-works/gas-fees#child-chain-gas-pricing). At recent ETH prices around $2500, any of these three minimum base fees (which correspond to `2**20` through `2**24`) would be much less than one cent for a USDC transfer (about 40,500 gas). As ETH prices change, rollup operators will want to adjust the minimum base fee accordingly.

| log2(minBaseFee) | Minimum Base Fee (gwei) | USDC Transfer Cost (ETH) | Cost (USD, ~$2,500/ETH) | Time Saved on OP Mainnet (minutes) |
| ---------------- | ----------------------- | ------------------------ | ----------------------- | ----------------------------------- |
| 19 | 0.000524288 | 2.12e-8 | $0.0000531 | 11.2 |
| 20 | 0.001048576 | 4.25e-8 | $0.000106 | 11.8 |
| 21 | 0.002097152 | 8.49e-8 | $0.000212 | 12.4 |
| 22 | 0.004194304 | 1.70e-7 | $0.000425 | 13.0 |
| 23 | 0.008388608 | 3.40e-7 | $0.000849 | 13.6 |
| 24 | 0.016777216 | 6.79e-7 | $0.00170 | 14.2 |
| 25 | 0.033554432 | 1.36e-6 | $0.00340 | 14.8 |
| 26 | 0.067108864 | 2.72e-6 | $0.00679 | 15.3 |
| 27 | 0.134217728 | 5.44e-6 | $0.0136 | 15.9 |

# Backwards Compatibility

We expect this change to have little impact on applications. Applications estimate the base fee component of their costs by looking at the previous block's base fee and potentially adding a margin of [12.5%](https://ethereum.org/en/developers/docs/gas/#base-fee) or [100%](https://www.blocknative.com/blog/eip-1559-fees) for base fee increases caused by congestion. 

# Alternatives Considered

* **Hard-code a minimum base fee** (like [EIP 7762](https://eips.ethereum.org/EIPS/eip-7762)). This is the simplest solution, but it doesn't give the chain operator any flexibility to adjust the minimum base fee. Establishing a Superchain-wide consensus on a minimum base fee seems difficult, especially if the value will have to wait for the next fork to be changed.
* **Link the minimum base fee to an outside price signal** (like [EIP 7918](https://eips.ethereum.org/EIPS/eip-7918)). This approach avoids choosing a magic number for the minimum base fee and has more elegant economics: the fee doesn't drop when we have a good signal that reducing the base fee wouldn't increase gas usage. The downside is that it's harder to reason about the right values to use in the formula, which makes it hard to build a consensus around. A configurable minimum base fee gives us an option that L1 doesn't have: it allows rollup operators to change the minimum base fee as needed to ensure the fee market is working properly.
* **Do nothing**. We expect to make batcher sequencer throttling unnecessary by repricing calldata correctly, but setting a minimum base fee still gives us two big wins: we can protect the fee market from long periods of first-price auctions in a generalized way, and we can make sure OP Stack is prepared for scaled rollups that persistently operate below their gas target.

# Risks & Uncertainties

* **The rollup operator can attempt to censor the chain with an extremely high minimum base fee**. The minimum base fee can be as high as 2^255, which is more than the total supply of ETH (~2^87). As a workaround, deposit transactions can force inclusion without being affected by the minimum base fee because they [pay for gas on L1 using a separate fee market](https://docs.optimism.io/stack/transactions/deposit-flow#denial-of-service-dos-prevention).
