# opgas - the Offchain Unit of Computation: Design Doc

| Author | *Anton Evangelatov, Josh Klopfenstein* |
| --- | --- |
| Created at | *2026-01-13* |
| Initial Reviewers | *TBD* |
| Need Approval From | *TBD* |
| Status | *Draft* |

## Purpose

Allow the sequencer of OP Stack chains to influence the fee paid on a per-transaction basis, aligning fees with the *actual* performance characteristics of each transaction, rather than rely solely on the static, in-protocol EVM gas model.

## Summary

Introduce **opgas** unit of calculation, which allows the sequencer of a given OP Stack chain to influence the fee paid by users on a per-transaction basis. Users sign transactions for the worst-case execution and receive rebates if better-than-worst-case execution is possible. By assuming worst-case performance, the sequencer can not only charge the signed gas limit for adversarial transactions, such as spam, but also a lower fair price is transaction execution was better than worst-case.

## Problem Statement + Context

The EVM gas model is a static, in-protocol approximation of the real-world supply of computation. It is not based on the real-world performance or bottlenecks of the chain. This results in a few problems:
- a significant disparity in block processing times
- transactions are systematically mispriced - if we think of every transaction as consuming some amount of wall-clock time to process, we are systematically mispricing transactions: those that take a long time to process are undercharged relative to those that are executed quickly.
- performance variance - the best-case versus worst-case execution difference is approximately 10-100x (SWAG); an ETH transfer can achieve approximately 1-2 giga-gas per second (SWAG) of throughput, in contrast, certain computationally intensive operations (such as some precompiles) may only achieve 20 mega-gas per second (SWAG) — a difference of roughly 50-100x (SWAG) in real performance despite similar gas costs.

As an example, a very large percent of Base chain state has been taken over by a single protocol of questionable utility - XEN. A tiny number of users have monopolized the majority of network resources due to transaction execution mispricing - both in terms of *CPU* and *storage* resource utilization.

## Proposed Solution

The proposed solution is to introduce **opgas** notion, alongside **gas used** (EVM gas used) and **gas limit**, at the expense of putting more trust in the sequencer of a given OP Stack chain. **opgas** value is determined by the sequencer and is capped at the **gas limit** (or up to the **gas used**, namely EVM gas used, to be determined later).

We propose for an initial implementation to couple **opgas** with the **wall-clock execution time**, although alternative schemes are also possible. **wall-clock execution time** would act as a real-time oracle for the supply of computation. This proposal does not address *state growth*, although it's possible to amend the **opgas** scheme in the future and account for other resources, and not just CPU-time.

### Core Mechanism

1. **Set the block gas limit to the worst-case EVM gas-per-millisecond throughput $G$:** Calculate the gas per millisecond that the chain can safely handle assuming every transaction is maximally expensive. This becomes the block gas limit.
2. **Execute transactions and measure wall-clock time:** The sequencer runs each transaction and records how long it actually takes to execute. For a transaction that executes with gas used $g$ and time $t$, adjust its gas used proportional to the worst-case throughput $G$, capped at the signed transaction gas limit.
3. **Rebate proportional to performance:** If a transaction executes faster than the worst-case assumption, the sequencer rebates gas proportionally. A transaction that runs in 10% of the expected time gets 90% of its gas rebated.
4. **EIP-1559 continues to function:** The rebate mechanism integrates with EIP-1559 price discovery. The base fee adjusts based on actual (post-rebate) gas consumption, maintaining market-based pricing.

This gas model effectively uses a new unit of computation, which we call **opgas**, which in the happy case results in lower-than-gas-limit gas used and only falls back to metering resources in EVM gas for worst-case transactions.

$$
\text{opgas}=\min\left\{\frac{g}{t\cdot G}, 1\right\}\text{evmgas}
$$

```python
# Constant determined by experimentation. Using 100MGas/s as an example.
WORST_CASE_GAS_PER_SECOND = 100_000_000

def evmgas_to_opgas(gas_used, seconds_used, tx_gas_limit):
  worst_case_gas_used = WORST_CASE_GAS_PER_SECOND * seconds_used
  op_gas_scalar = min(gas_used / worst_case_gas_used, 1)
	op_gas_used = min(gas_used * op_gas_scalar, tx_gas_limit)
	return op_gas_used
```

Given the **core mechanism**, we get the following effects:

1. **Existing UX preserved:** Users already sign transactions specifying the maximum they're willing to pay. This proposal maintains that property — users sign for the worst case and receive rebates for better-than-worst-case execution.
2. **Guaranteed coverage:** By assuming worst-case performance, the sequencer can always charge a fair price even for adversarial transactions. If a user sends a maximally expensive transaction, they've already committed to paying for it.
3. **Second-price auction semantics:** The model retains the property that users pay the market rate, not their maximum. They sign high, pay only what's actually consumed.

### Design Principles

The presented design has been thought out based on the following principles:

- Preserve existing UX with respect to transaction signing.
- Stick with a minimal set of changes.
- EIP-1559 continues to function.

### Architecture and PoC

#### Refund accumulation

During transaction execution, `op-geth` already contains code for *Refund Accumulation*. When a storage slot is cleared (set from non-zero to zero), refunds are added to the refund counter in core/vm/gas_table.go. At the end of transaction execution, `op-geth` already contains code for *Refund Application* at `core/state_transition.go`.

#### Propagating calculated `opgas` to block header building

After transaction execution, we need to propagate the calculated `opgas` (based on *wall-clock execution time*) - this could potentially be done with a new field added to Receipt at `core/types/receipt.go`.

#### Block header modifications
An open design question is where the sequencer communicates the rebate amounts. An option is to Include a mapping of transaction indices to rebate amounts in the block header. This keeps transaction data unchanged, but requires header extensions.

#### Benchmarking: Baseline calibration
Establish a baseline for "worst-case gas per millisecond" and "best-case gas per millisecond" through benchmarking. All rebates are calculated relative to this baseline. Millisecond-level timing should suffice. Sub-millisecond precision adds complexity without proportional benefit.

#### Benchmarking: How effective the PoC is

TODO(anteva): TBD

## Impact on Developer Experience

TODO(anteva): Add notes around estimaton APIs and trust of sequencer.

## Note on Sequencer Trust

This design relies on the sequencer to set rebates honestly. Two mitigations are worth noting:

- **Temporary measure:** If gas schedules eventually become modular or perfectly aligned with performance, fee pricing can move back on-chain.
- **TEE execution:** Running the sequencer in a trusted execution environment can provide stronger guarantees that rebates are calculated honestly based on actual resource consumption.

## Security Considerations

Considering that the **opgas** unit and functionality is increasing the trust in the OP chain sequencer, theoretically it is possible to encode and program a stop-gap in an event of chain incident, where the L2 chain operator could turn off **opgas** on-demand and revert back to EVM gas used.

## Alternatives Considered

### Transaction Timeout with Reversion

Allow the sequencer to revert transactions that exceed their time budget. If a transaction consumes more wall-clock time than its gas would justify at worst-case rates, the sequencer could revert it and charge for the time consumed.

This accomplishes similar goals but grants additional power to the sequencer (arbitrary reversion capability), which may raise concerns. The rebate approach is more conservative in terms of sequencer authority.

## Risks & Uncertainties

TODO(anteva): TBD

### Different client implementations will have different results

Different client implementations may have drastically different results. This is probably preferable! The EVM gas model abstracts over the implementation, which is what we want to avoid with opgas. For UX purposes, it’s probably best to run a single implementation as the sequencer.

## Open questions

1. Should the sequencer be able to charge up to the signed transaction `gas limit`, or should the sequencer be able to charge up to the `EVM gas used`?
