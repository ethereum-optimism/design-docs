# Multiple Gas Schedules

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | @niran                                             |
| Created at         | 2025-07-03                                         |

---

# Purpose
 
This document proposes offering multiple gas schedules in OP Stack for rollup operators to choose from at runtime via `SystemConfig`.

# Summary

We introduce a new configuration value that can be modified via `SystemConfig`:

| Name             | Type    | Default  | Meaning |
|------------------|---------|----------|---------|
| `gasScheduleId` | `bytes2` | `0x0000` | The ID of the gas schedule to use |

This value is updated via a new function in `SystemConfig`:

```solidity
function setGasScheduleId(bytes2 gasScheduleId) external onlyOwner;
```

This function will emit a `ConfigUpdate` log-event, with a new `UpdateType`: `UpdateType.GAS_SCHEDULE_ID`.

Gas schedule ID `0x0000` represents the default gas schedule inherited from Ethereum. Each additional gas schedule is identified by a unique ID determined by given the gas schedule a human-readable label, calculating the keccak256 hash of the label, and taking the first two bytes of the hash.

Each OP Stack hard fork can update each gas schedule. Typical forks will just inherit any repricings that have occurred on Ethereum. However, when broad repricings like [EIP 7904](https://eips.ethereum.org/EIPS/eip-7904) occur, the corresponding OP Stack hard fork will need to update each gas schedule to match the new gas costs.

The initial alternative gas schedule is called `target-25Mgps-accounts-500M` and has the ID `0x3e3a`. Its gas prices are optimized for a chain with a gas target equivalent to 25M gas per second and 500M accounts in its state. Such a chain should have opcodes priced such that the base fee increases when state trie insertions take longer than half of a block time to process. In total, three gas schedules would be added to start with:

| Label                         | ID       | Gas target (Mgas/sec) | Accounts (M) |
|-------------------------------|----------|-----------------------|--------------|
| `target-25Mgps-accounts-500M` | `0x3e3a` |                    25 |          500 |
| `target-15Mgps-accounts-500M` | `0x688a` |                    15 |          500 |
| `target-5Mgps-accounts-500M`  | `0x70fd` |                     5 |          500 |

## Gas Cost Changes in `target-25Mgps-accounts-500M`

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 480,000 | 15× increase |
| `CALL` (to new account) | 25,000 | 509,100 | 15× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |

## Gas Cost Changes in `target-15Mgps-accounts-500M`

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 288,000 | 9× increase |
| `CALL` (to new account) | 25,000 | 297,000 | 9× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |

## Gas Cost Changes in `target-5Mgps-accounts-500M`

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 96,000 | 3× increase |
| `CALL` (to new account) | 25,000 | 93,000 | 3× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |

# Problem Statement + Context

Gas prices in Ethereum are designed to prevent strain on the network by capping the usage rate for each operation (at `GAS_LIMIT / BLOCK_TIME / OPERATION_COST`). This has worked well for Ethereum mainnet with minor adjustments to the gas schedule over time. However, rollups use significantly shorter block times and aspire to much higher gas targets: Ethereum targets 1.5 Mgas/sec today while World Chain targets 9 Mgas/sec and Base targets 25 Mgas/sec. They've begun to run into scenarios that demonstrate wildly mispriced operations, especially when inserting into the state trie.

At 25 Mgas/sec, [Base's benchmarks](https://github.com/base/benchmark) show that calculating the state root for a block full of `CALL`s to new accounts takes 5.5 seconds during a GetPayload call that is sent two seconds after geth begins processing the block. This time increases roughly linearly with the number of accounts created: at 12.5 Mgas/sec, GetPayload takes 2.2 seconds, and at 37.5 Mgas/sec, GetPayload takes 8.2 seconds. To ensure that the sequencer can consistently produce blocks within two seconds, we'd like to start increasing the base fee when the state root calculation exceeds 500ms, so we increase the gas cost of account creation by 15×. This also allows for an EIP 1559 multiplier of up to 4 before it becomes possible to create blocks that take longer than two seconds to calculate the state root.

Each time an OP Stack rollup increases its gas target, the operator should consider updating the gas schedule to restrict state root calculation time.

# Alternatives Considered

* **Use a single custom gas schedule for all OP Stack chains**. Most OP Stack chains already have gas targets high enough to be worth repricing account creation, so repricing for everyone would be a reasonable option. But since the state root calculation time depends on gas targets and state sizes that change for chains over time, it's more useful to give operators a bit more flexibility.
* **Directly configure gas costs in `SystemConfig`**. This is the most flexible option, but since many opcodes have customized formulas with multiple parameters, storing them all in `SystemConfig` would get messy. This would also make mistakes during forks more likely since each rollup operator would need to evaluate their customizations to make sure they make sense for each fork.
