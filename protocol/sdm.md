# Sequencer-Defined Metering (SDM) - an offchain and out-of-protocol mechanism for gas refunds by the sequencer

| Author             | _Anton Evangelatov, Josh Klopfenstein_ |
| ------------------ | -------------------------------------- |
| Created at         | _2026-01-13_                           |
| Initial Reviewers  | _TBD_                                  |
| Need Approval From | _TBD_                                  |
| Status             | _Draft_                                |

## Purpose

Allow the sequencer of OP Stack chains to influence the fee paid on a per-transaction basis, aligning fees with the actual performance characteristics of each transaction, rather than rely solely on the static, in-protocol EVM gas model.

## Summary

Introduce a subjective sequencer-defined gas refund alongside `gas used` and `gas limit`. SDM is introducing a new system post-execution transaction, with type `0x7D`, which contains post-execution data injected by the sequencer. The motivation for SDM and the first application of SDM is the **block-level warming rebates** policy: it allows transactions to benefit from already warmed accounts and storage slots from earlier in the same block by a sequencer-defined rebate, as described in [EIP-7863](https://eips.ethereum.org/EIPS/eip-7863). This is the initial go-to-market policy; future SDM policies may issue different kinds of refunds. Users still sign transactions in the usual way for the worst case, while the sequencer may charge less when execution benefits from block-local access reuse.

## Problem Statement + Context

The EVM gas model is a static, in-protocol approximation of resource consumption. It does not capture all execution characteristics that matter operationally to an OP Stack sequencer. In particular, it does not account for the fact that transaction ordering within a block can change the effective cost of later transactions when accounts and storage slots have already been accessed.

This creates a few problems:

- transactions that benefit from prior accesses in the same block are priced as if they were paying the full cold-access cost;
- the sequencer cannot reflect block-local execution efficiencies in fees, even when those efficiencies are material;
- repeated access patterns within a block are not surfaced in a way that can be inspected or analyzed later.

The goal of SDM is to let the sequencer account for such effects explicitly, starting with a narrow and observable policy: **block-level warming rebates** - an implementation of [EIP-7863](https://eips.ethereum.org/EIPS/eip-7863).

## Proposed Solution

The proposed solution is to introduce a sequencer-defined refund alongside **gas used** (EVM gas used) and **gas limit**, at the expense of putting more trust in the sequencer of a given OP Stack chain. In the initial policy, the sequencer computes block-warming rebates off-chain and communicates them in-block.

Under this policy:

- normal EVM execution remains unchanged;
- users continue to sign standard transactions with a normal gas limit;
- the sequencer computes a per-transaction rebate when a transaction benefits from accounts or storage slots already warmed earlier in the block;
- the sequencer includes a new system post-execution transaction, with type `0x7D`, into each block. This transaction contains all rebates for all transactions in the block;
- the rebate is applied during the state transition, and the user is refunded that rebate by having the `gas used` of their transaction reduced by the sequencer;
- the rebate is exposed as metadata and projected onto receipts for each transaction.

### Core Mechanism

1. **Track warming across the block:** During block execution, the sequencer tracks which accounts and storage slots have already been warmed by earlier transactions.
2. **Compute per-transaction rebate:** For each later non-deposit transaction, if an accessed account or storage slot was already warmed earlier in the block, the sequencer accumulates a rebate.
3. **Serialize the rebate data in-block:** At the end of block building, the sequencer emits a system post-execution transaction, with type `0x7D`, containing the refund entries.
4. **Project rebates onto receipts:** Nodes decode the post-execution transaction and expose the corresponding per-transaction rebate in RPC receipts.

This gas model uses an additional refund mechanism alongside the canonical EVM gas used, where the sequencer can account for execution-local effects that the static gas schedule does not express. In the current PoC, this is realized as a refund-only model:

$$
\text{effective gas}=\text{gas used}-\text{sequencer-defined refund}
$$

There is also a block-level cap on the **total permissible gas used before rebates**. This is the hard limit on the sum of the raw `gas used` values of transactions that the sequencer is allowed to include in a block, before subtracting any SDM rebates. In production, this limit will be set to `MAX_GAS_LIMIT` from `L1.SystemConfig`.

Given the **core mechanism**, we get the following effects:

1. **Existing UX preserved:** Users still sign standard transactions specifying the maximum they are willing to pay.
2. **Minimal execution changes:** The EVM gas schedule remains unchanged; SDM is carried as additional sequencer-produced accounting.
3. **Inspectable accounting:** The rebate data is explicit in-block and can be projected onto receipts and replayed later.

### Design Principles

The presented design has been thought out based on the following principles:

- Preserve existing UX with respect to transaction signing.
- Stick with a minimal set of changes.
- Keep the first policy narrow and inspectable.
- Make the sequencer’s accounting externally replayable.

### Architecture and PoC

The current PoC implements SDM as **block-level warming** in `op-reth`. The PoC is implemented in branch `nonsense/opgas-reth-poc2-tracer`, and you can find an overview document specifically for it at: [SDM PoC 2 Review document](https://www.notion.so/oplabs/SDM-PoC-2-Review-330f153ee162801e8fc8e5e79c010e40)

Within the current PoC the refunds are not applied to `gas used`, so that blocks are replayable and don't modify consensus fields, such as the `state root` during benchmarking with real-world chain data.

In the production implementation of SDM `gas used` will include the refund and will not be equal to the canonical EVM gas used.

#### Refund accumulation

During block execution, the executor maintains persistent warming state across transactions in the block. When a later user transaction accesses state that was already warmed earlier in the block, a rebate is accumulated.

The current PoC uses rebate values that come directly from [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929):

- **warm account:** 2500 gas refund
- **warm storage read:** 2000 gas refund
- **warm storage write:** 2100 gas refund

Deposits may warm state for later transactions, but do not themselves receive rebates.

These values match the cold -> warm access deltas used by tx-level warming.

#### Propagating calculated refunds to block assembly

After execution, per-transaction rebate entries are accumulated as `(tx_index, gas_refund)` pairs. During payload building, these entries are encoded into a system OP transaction of type `0x7D`, namely `TxPostExec`.

The post-execution transaction payload is RLP encoded and currently contains:

- a `nonce` to ensure uniqueness,
- a `version`, and
- a list of refund `entries`, where each entry contains:
  - `index`: the original transaction index in the block
  - `gas_refund`: the sequencer-computed rebate

#### Receipt / RPC projection

Nodes inspect the block for the system post-execution transaction, decode its payload, and project the refund for each transaction into the RPC receipt as **`opGasRefund`**.

#### Replay and inspection

The PoC also includes replay and inspection tooling that can:

- counterfactually replay a historical block with SDM accounting,
- synthesize the expected SDM payload,
- compare replay results against the embedded payload and receipt-level refund projection,
- attribute each refund to the earlier transaction that first warmed the account or storage slot.

#### Benchmarking: How effective the PoC is

Benchmarking document for PoC 2 can be found at: [SDM Benchmarking for PoC 2](https://www.notion.so/oplabs/SDM-Benchmarking-for-PoC-2-Block-Level-Warming-330f153ee16280d4b2effdf0b3b53eb7)

#### Integration with Flashblocks

A concrete design option for integration of SDM with Flashblocks is to preserve a single final post-execution transaction in the sealed block.

The core idea is:

- `op-rbuilder` computes and streams **refund-aware prefixes** on every flashblock
- each prefix includes the **current candidate `TxSDM`** for the whole prefix
- `rollup-boost` no longer blindly concatenates all transactions, but it treats `TxPostExec` as a **reserved mutable slot** and always keeps only the **latest** `TxPostExec`

This results in:

- exactly one `TxPostExec` in the final block
- per-prefix cumulative fields (`gas_used`, `state_root`, `receipts_root`, `block_hash`) already computed correctly by the builder
- no need for `rollup-boost` to execute or recompute roots

This is a viable design, but it is **not append-only** in the current strict sense. It introduces a new rule:

> All user transactions are append-only across prefixes, but the reserved `TxPostExec` position is mutable and may be replaced by later prefixes.

## Note on Sequencer Trust

The current PoC makes the SDM accounting explicit in-block and externally replayable.

## Security Considerations

Considering that the functionality to issue refunds by the sequencer increases trust in the OP chain sequencer, it should be possible to encode and program a stop-gap such that, in the event of a chain incident, the L2 chain operator could turn off the refund mechanism on-demand and revert back to plain EVM gas accounting.

## Alternative Policies Considered

### Wall-clock execution time

An alternative SDM policy is to price transactions based on wall-clock execution time. This would let the sequencer align fees more directly with observed execution cost, rather than with specific execution effects such as block-level warming.

This may ultimately address a broader class of mispricing issues, but it also introduces more design complexity around measurement, calibration, reproducibility, and trust. The current proposal starts with block-warming because it is narrower, easier to inspect, and easier to replay.
