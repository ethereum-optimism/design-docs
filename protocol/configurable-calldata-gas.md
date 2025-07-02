# Configurable Calldata Gas Costs

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | @niran                                             |
| Created at         | 2025-06-30                                         |


---

# Purpose
 
This document proposes making calldata costs configurable by the chain operator at runtime, not just during hard forks.

---

# Summary

[EIP 7623](https://eips.ethereum.org/EIPS/eip-7623) uses `STANDARD_TOKEN_COST` and `TOTAL_COST_FLOOR_PER_TOKEN` parameters to calculate the floor cost of each transaction. This proposal makes these parameters configurable via `SystemConfig` instead of hard-coded in the client.

We introduce **two new configuration values** that can be modified via the rollup's `SystemConfig` contract:

| Name | Type | Default | Meaning |
|------|------|-------------------|---------|
| `eip7623StandardTokenCost` | `uint32` | `4` | The ratio of the cost of a non-zero byte in `tx.data` to the cost of a zero byte in `tx.data` |
| `eip7623TotalCostFloorPerToken` | `uint32` | `10` | The cost of a zero byte in `tx.data` when calculating the total cost floor |

These values are updated via a single new function in `SystemConfig`:

```solidity
function setEIP7623Params(uint32 standardTokenCost, uint32 totalCostFloorPerToken) external onlyOwner;
```

This function will emit a `ConfigUpdate` log-event, with a new `UpdateType`: `UpdateType.EIP7623_PARAMS`.

At the next fork, op-geth stops reading the compile-time constants in `FloorDataGas` and instead calls a `FloorDataGasFunc` built from the values stored in two dedicated slots of the `L1Block` contract. The values are included in the state of the rollup via the [L1 block attributes](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/isthmus/l1-attributes.md) transaction that updates the `L1Block` contract's storage.

---

# Problem Statement & Context

In Ethereum, calldata costs are determined within the calculation for intrinsic gas, which requires a hard fork to change. Recent EIPs have avoided changing the cost of calldata directly by setting a floor for the transaction's cost instead. Pectra's [EIP 7623](https://eips.ethereum.org/EIPS/eip-7623) set a floor of 10 gas for zero bytes and 40 gas for nonzero bytes. [EIP 7976](https://eips.ethereum.org/EIPS/eip-7976) proposes to increase the floor cost further based partially on L1's plans to increase the gas target. These gas cost floors primarily affect calldata-heavy transactions, leaving execution-heavy transactions unaffected.

Rollups have inherited Ethereum's intrinsic gas formula, but rollups have different constraints for the calldata that they process, particularly the L1's data availability capacity, the rollup's block time, and the rollup's gas limit. These constraints change over time, so the cost of calldata must change accordingly to use L1's data availability capacity without exceeding it. At a minimum, rollups will want to adjust their calldata costs with every L1 hard fork that changes blob capacity, which coincides well with OP Stack forks. But any time a rollup wants to change its gas limit (as `SystemConfig` already allows), it will also want to adjust its calldata costs to account for the new gas limit.

Since Pectra increased L1's data availability capacity to a target of 6 128kb blobs every 12 seconds, it can handle 64 kb/s of throughput. A single rollup with a two-second block time can only accept 128kb of batches per block. At 40 gas per byte (as of Pectra's [EIP 7623](https://eips.ethereum.org/EIPS/eip-7623)), it would cost 5.12M gas per block for a rollup's calldata to use enough blobs to hit L1's blob target. Most OP Stack chains have gas targets high enough to accept far more than 128kb of calldata per block! This allows blocks to exceed the rollup's constraints without putting any pressure on the base fee to price out this congestion.

| Chain | Gas target | Underpricing factor |
|-------|------------|---------------------|
| Base | 50M | ≈ 9.8× |
| OP Mainnet | 20M | ≈ 3.9× |
| Ink  | 15M | ≈ 2.9× |
| Unichain | 15M | ≈ 2.9× |
| Mode | 15M | ≈ 2.9× | 

Because so many OP Stack rollups have underpriced calldata, these chains rely on [batcher sequencer throttling](https://docs.optimism.io/operators/chain-operators/configuration/batcher#batcher-sequencer-throttling) to prevent large backlogs of batches from accumulating. The downside of this approach is that it "breaks the fee market." Users must outbid each other's priority fees in a first-price auction instead of expecting all transactions willing to pay the current base fee to be included in blocks. Many applications do not expect to actively participate in priority fee auctions because EIP 1559 has been so successful at eliminating them, so they experience these periods as a denial of service. (The base fee also plummets to zero during these periods, so even when the throttle is no longer binding, it takes time for the priority fee auction to end.)

Pricing calldata correctly should make batcher sequencer throttling unnecessary and eliminate the priority fee auctions we see today when calldata is congested.

---

# Alternatives Considered

* **Continue hard-coding gas costs**. Updating gas costs once or twice per year on a similar cadence to L1 hard forks is likely frequent enough for most chains, but it doesn't address differences in gas targets between OP Stack chains without each chain maintaining a persistent fork of the client.
* **Full fee-abstraction** (see [`specs#73`](https://github.com/ethereum-optimism/specs/issues/73)). Only addresses fees, but we need to manipulate gas consumption for the fee market to function properly.
* **Expose a single configuration value**. We don't foresee `eip7623StandardTokenCost` ever being modified from its default value of `4`, but it's included in the proposal for completeness.
* **Change the actual calldata gas cost, not just the floor**. This diverges from how the L1 community has been addressing calldata costs and impacts more transactions than are necessary to solve the problem.
* **Adopt multidimensional metering** (see the [Ethereum Magicians thread](https://ethresear.ch/t/a-practical-proposal-for-multidimensional-gas-metering/22668)). This is likely the proper medium-term solution, but it will take time for the proposal to stabilize and to be adopted on L1.
* **Generalized configurable gas schedule**. There are more resources that are mispriced on rollups besides calldata. Extending this proposal to cover them would likely require dynamic arrays in `SystemConfig` and `L1Block`, so the added complexity doesn't seem worth it yet. Calldata would likely still be a special case since there's no corresponding opcode to price.

---

# Risks & Uncertainties

* As with all updates to `SystemConfig` and `L1Block`, the storage slots must be carefully allocated to ensure forward compatibility.
* Future Ethereum EIPs may make this configuration obsolete.
* Without care, it might be possible to set values so high that the system transaction cannot be executed and the chain will halt.
* L1's Glamsterdam fork might include [EIP 7904](https://eips.ethereum.org/EIPS/eip-7904), which aims to reprice the entire gas schedule. The OP Stack fork that includes Glamsterdam would need to consider how rollups with customized calldata costs would transition to a new value in a coordinated way.
