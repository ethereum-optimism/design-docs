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

Introduce SDM as a general mechanism for subjective sequencer-defined gas refunds alongside `gas used` and `gas limit`. SDM adds a new system post-execution transaction, with type `0x7D`, which contains post-execution refund data injected by the sequencer.

The first application of SDM is the **block-level warming rebates** policy: it allows transactions to benefit from already warmed accounts and storage slots from earlier in the same block by a sequencer-defined rebate, as described in [EIP-7863](https://eips.ethereum.org/EIPS/eip-7863). This is the initial go-to-market policy; future SDM policies may issue different kinds of refunds. Users still sign transactions in the usual way, while the sequencer rebates according to its own off-chain SDM policies.

## Problem Statement + Context

The EVM gas model is a static, in-protocol approximation of resource consumption. It does not capture all execution characteristics that matter operationally to an OP Stack sequencer.

The sequencer currently cannot reflect execution efficiencies and inefficiencies in fees, even when those are material.

The goal of SDM is to let the sequencer account for such effects explicitly through subjective sequencer-defined refunds, starting with a narrow and observable policy: **block-level warming rebates** - an implementation of [EIP-7863](https://eips.ethereum.org/EIPS/eip-7863).

Currently the EVM gas model does not account for the fact that transaction ordering within a block can change the effective cost of later transactions when accounts and storage slots have already been accessed. As a result, transactions that benefit from prior accesses in the same block are priced as if they were paying the full cold-access cost.

## Proposed Solution

The proposed solution is to introduce SDM as a sequencer-defined refund mechanism alongside **gas used** (EVM gas used) and **gas limit**, at the expense of putting more trust in the sequencer of a given OP Stack chain.

Under SDM generally:

- normal EVM execution remains unchanged;
- users continue to sign standard transactions with a normal gas limit;
- the sequencer computes a per-transaction refund according to one or more SDM policies;
- the sequencer includes a new system post-execution transaction, with type `0x7D`, into each block. This transaction contains all refunds for all transactions in the block;
- the refund is applied during the state transition, and the user is refunded that amount by having the `gas used` of their transaction reduced by the sequencer;
- the refund is exposed as metadata and projected onto receipts for each transaction.

For the initial policy, the sequencer computes **block-level warming rebates** off-chain and communicates them in-block.

### Core Mechanism

SDM is the **general mechanism** by which the sequencer can attach a subjective, per-transaction refund to normal transaction execution. It should be thought of as a framework for sequencer-defined refunds, not as a synonym for any one refund policy.

At a high level, SDM works as follows:

1. **Execute transactions normally:** User and deposit transactions execute under the standard EVM gas schedule.
2. **Compute policy-defined refunds off-chain:** The sequencer applies one or more SDM policies to determine whether any transaction should receive a subjective refund, and in what amount.
3. **Serialize refund metadata in-block:** At the end of block building, the sequencer emits a system post-execution transaction, with type `0x7D`, containing the refund entries.
4. **Apply refunds as SDM accounting:** The refund is applied during the state transition, reducing the effective gas charged to the user relative to canonical EVM gas used.
5. **Project refunds onto receipts:** Nodes decode the post-execution transaction and expose the corresponding per-transaction refund in RPC receipts.

This gas model uses an additional refund mechanism alongside the canonical EVM gas used, where the sequencer can account for execution-local effects that the static gas schedule does not express. In the current PoC, this is realized as a refund-only model:

$$
\text{effective gas}=\text{gas used}-\text{sequencer-defined refund}
$$

There is also a block-level cap on the **total permissible gas used before rebates**. This is the hard limit on the sum of the raw `gas used` values of transactions that the sequencer is allowed to include in a block, before subtracting any SDM rebates. In production, this limit will be set to `MAX_GAS_LIMIT` from `L1.SystemConfig`.

Given the **SDM mechanism**, we get the following effects:

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

The current PoC implements **one SDM policy**: **block-level warming** in `op-reth`. The PoC is implemented in branch `nonsense/opgas-reth-poc2-tracer`, and you can find an overview document specifically for it at: [SDM PoC 2 Review document](https://www.notion.so/oplabs/SDM-PoC-2-Review-330f153ee162801e8fc8e5e79c010e40)

Within the current PoC the refunds are not applied to `gas used`, so that blocks are replayable and don't modify consensus fields, such as the `state root` during benchmarking with real-world chain data.

In the production implementation of SDM `gas used` will include the refund and will not be equal to the canonical EVM gas used.

#### Refund accumulation for the block-level warming policy

During block execution the executor maintains persistent warming state across transactions in the block. A rebate is owed to a later transaction **only when it actually pays the cold access cost** for an account or storage slot that an **earlier** transaction in the same block already warmed — the rebate refunds a cold→warm surcharge the transaction genuinely paid but that, block-locally, was unnecessary because the client had already loaded that state.

The rebate values come directly from [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) and match the cold → warm access deltas:

- **warm account:** 2500 gas refund (cold 2600 − warm 100)
- **warm storage read (SLOAD):** 2000 gas refund (cold 2100 − warm 100)
- **warm storage write (SSTORE):** 2100 gas refund (cold 2200 − warm 100)

##### What is never rebated: intrinsically-warm and non-charged accesses

Warming an account/slot *for later transactions* and *claiming a rebate* for it are separate concerns. An address/slot is recorded as warmed by the first transaction that touches it (so a later transaction that genuinely pays a cold access for it can be rebated), but a touch the EVM did **not** charge the cold price for claims nothing. The following are therefore **never** rebated, even when an earlier transaction in the block warmed the same address or slot:

- **A transaction's own EIP-2929 intrinsically-warm set.** EIP-2929 (with [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930), [EIP-3651](https://eips.ethereum.org/EIPS/eip-3651), and [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)) pre-seeds each transaction's accessed-address / accessed-slot set, so the transaction is billed the **warm** cost (100) — never the cold cost — on its first access to any of:
  - its own **`tx.sender`**,
  - its own **`tx.to`** (or, for a contract-creation transaction, the **created-contract address**),
  - the **precompiles**,
  - the **block coinbase** (beneficiary),
  - every **access-list** address and storage slot,
  - every **EIP-7702 authority**.

  No cold surcharge is ever paid for these, so no rebate is owed for a transaction touching its *own* intrinsically-warm entries — regardless of what earlier transactions did.
- **Protocol-level state writes the transaction is not charged a cold access for.** In particular the per-transaction **fee-vault settlement writes** — crediting the L1-fee, base-fee, and operator-fee recipients — warm those vault accounts but are not user opcode accesses and incur no EIP-2929 cold-access charge. They warm the vaults for later transactions but never themselves earn a rebate.
- **Deposit transactions** warm state for later transactions but never receive rebates.

###### Examples

- **Same sender twice.** Alice sends two transactions in one block. The second is **not** rebated for touching its own `tx.sender` (Alice): EIP-2929 pre-warmed Alice for that transaction, so it paid 100, not 2600 — no surcharge, no rebate.
- **Two transactions to the same contract.** Both call router `R`. Neither is rebated for `R` *as its own `tx.to`* (intrinsically warm). But if `R`'s code reads a storage slot that the first call already warmed, the second transaction **is** rebated 2000 for that SLOAD (it genuinely paid the cold 2100); likewise it is rebated 2500 for a *cold* `BALANCE`/`CALL` to some third account `D` that the first transaction warmed.
- **Contract creation.** A `CREATE` transaction whose created-contract address was warmed by an earlier transaction is **not** rebated for that address — it is the transaction's intrinsic creation target (billed warm).
- **Fee vaults.** A plain transfer warms the fee vaults via settlement but is **not** rebated for them. A transaction whose code does `BALANCE(baseFeeVault)` after an earlier transaction warmed the vault **is** rebated 2500 — it paid the cold access.

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
