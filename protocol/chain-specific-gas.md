# Chain-Specific Gas Schedules

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | @niran                                             |
| Created at         | 2025-07-03                                         |

---

# Purpose
 
This document proposes allowing rollup operators to specify their own gas schedule in OP Stack as part of each hard fork.

# Summary

Execution clients currently select the correct jump table for opcodes and their prices based on the active Ethereum execution layer fork:

```go
// NewEVMInterpreter returns a new instance of the Interpreter.
func NewEVMInterpreter(evm *EVM) *EVMInterpreter {
	var table *JumpTable
	switch {
	case evm.chainRules.IsPrague:
		table = &pragueInstructionSet
	case evm.chainRules.IsCancun:
		table = &cancunInstructionSet
  // ...
```

> _We're using the `geth` codebase for illustration. `reth` uses a different approach where the current fork is checked for each opcode rather than once for the entire jump table._

We establish a new pattern for Superchain rollups to specify their own gas schedule as part of each hard fork:

```go
// NewEVMInterpreter returns a new instance of the Interpreter.
func NewEVMInterpreter(evm *EVM) *EVMInterpreter {
	var table *JumpTable
	switch {
	case evm.chainRules.IsOptimismJovian && evm.chainRules.IsOptimismBase:
		table = &jovianInstructionSetForBase
	case evm.chainRules.IsPrague:
		table = &pragueInstructionSet
  // ...
```

Superchain rollups are identified by their chain IDs to set corresponding chain rules for the EVM:

```go
IsOptimismBase: isMerge && (c.ChainID == new(big.Int).SetUint64(8453) || c.ChainID == new(big.Int).SetUint64(84532))
```

The gas schedule customizations can be implemented by patching the default jump table for the latest fork:

```go
func newJovianInstructionSetForBase() JumpTable {
	instructionSet := newPragueInstructionSet()

	createOp := *instructionSet[CREATE]
	createOp.constantGas = 480_000
	instructionSet[CREATE] = &createOp

	create2Op := *instructionSet[CREATE2]
	create2Op.constantGas = 480_000
	instructionSet[CREATE2] = &create2Op

	callOp := *instructionSet[CALL]
	callOp.dynamicGas = gasCallJovianBase // 509_100
	instructionSet[CALL] = &callOp

	return validate(instructionSet)
}
```

Each OP Stack hard fork can potentially update the underlying gas schedule from Ethereum, like the Isthmus fork did for Pectra's gas schedule changes. By default, these changes would override any customizations made for a specific chain. To maintain gas schedule customizations, rollups must re-evaluate their customizations with each hard fork and reapply them when underlying changes occur.

Eventually, we'd like to remove chain-specific configuration from the codebase of execution clients and move them to the Superchain configs for each rollup.

```toml
[hardforks]
  canyon_time = 1704992401 # Thu 11 Jan 2024 17:00:01 UTC
  delta_time = 1708560000 # Thu 22 Feb 2024 00:00:00 UTC
  ecotone_time = 1710374401 # Thu 14 Mar 2024 00:00:01 UTC
  fjord_time = 1720627201 # Wed 10 Jul 2024 16:00:01 UTC
  granite_time = 1726070401 # Wed 11 Sep 2024 16:00:01 UTC
  holocene_time = 1736445601 # Thu 9 Jan 2025 18:00:01 UTC
  isthmus_time = 1746806401 # Fri 9 May 2025 16:00:01 UTC
  jovian_time = 1756702801 # Mon Sep 01 2025 05:00:01 UTC

  [hardforks.jovian]
  create_gas = 480_000
  create2_gas = 480_000
  call_new_account_gas = 509_100
```

The main blocker for this config-oriented approach is defining a configuration format that is flexible enough for the use cases we envision in the near term. We can simply pick a handful of parameters that we want to make configurable and commit to supporting those going forward, or we can try to shift the way the ecosystem thinks about gas cost EIPs and execution client implementations to expect all parameters to be configurable. An ecosystem-wide approach would require more work up front with the benefit of minimizing OP Stack's diffs from the standard execution clients.

This proposal recommends implementing chain-specific gas schedules directly in the execution client codebase for now, with the caveat that any customizations should be equivalent to pure parameter changes. This excludes any custom logic that cannot be expressed in a configuration file.

# Problem Statement + Context

Gas prices in Ethereum are designed to prevent strain on the network by capping the usage rate for each operation (at `GAS_LIMIT / BLOCK_TIME / OPERATION_COST`). This has worked well for Ethereum mainnet with minor adjustments to the gas schedule over time. However, rollups use significantly shorter block times and aspire to much higher gas targets: Ethereum targets 1.5 Mgas/sec today while World Chain targets 9 Mgas/sec and Base targets 25 Mgas/sec. They've begun to run into scenarios that demonstrate wildly mispriced operations, especially when inserting into the state trie.

At 25 Mgas/sec, [Base's benchmarks](https://github.com/base/benchmark) show that calculating the state root for a block full of `CALL`s to new accounts takes 5.5 seconds during a GetPayload call that is sent two seconds after geth begins processing the block. This time increases roughly linearly with the number of accounts created: at 12.5 Mgas/sec, GetPayload takes 2.2 seconds, and at 37.5 Mgas/sec, GetPayload takes 8.2 seconds. To ensure that the sequencer can consistently produce blocks within two seconds, we'd like to start increasing the base fee when the state root calculation exceeds 500ms, so we increase the gas cost of account creation by 15×. This also allows for an EIP 1559 multiplier of up to 4 before it becomes possible to create blocks that take longer than two seconds to calculate the state root.

Each time an OP Stack rollup increases its gas target, the operator should consider updating the gas schedule to restrict state root calculation time.

## Example Gas Schedule Changes

### 25 Mgas/sec with 500M Accounts

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 480,000 | 15× increase |
| `CALL` (to new account) | 25,000 | 509,100 | 15× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |

### 15 Mgas/sec with 500M Accounts

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 288,000 | 9× increase |
| `CALL` (to new account) | 25,000 | 297,000 | 9× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |

### 5 Mgas/sec with 500M Accounts

| Opcode | Standard Gas Cost | New Gas Cost | Notes |
|--------|--------------|--------------|-------|
| `CREATE` | 32,000 | 96,000 | 3× increase |
| `CALL` (to new account) | 25,000 | 93,000 | 3× of the previous total (25,000 creation + 9,000 value transfer) while leaving the 9,000 value-transfer cost unchanged |


# Alternatives Considered

* **Use a single custom gas schedule for all OP Stack chains**. Most OP Stack chains already have gas targets high enough to be worth repricing account creation, so repricing for everyone would be a reasonable option. But since the state root calculation time depends on gas targets and state sizes that change for chains over time, it's more useful to give operators a bit more flexibility.
* **Offer a set of pre-defined gas schedules**. This would allow rollups to choose from a set of common gas schedules that are optimized for different gas targets and state sizes. This would remove the need to configure specific gas cost parameters in the config file. Instead, the rollup operator would just specify the label for the gas schedule they want to use. But this approach would put the burden of maintaining these gas schedules and coordinating changes on the OP Stack team, and the code needed to implement this would be difficult to merge into upstream execution clients.
* **Directly configure gas costs in `SystemConfig`**. This is the most flexible option, but since many opcodes have customized formulas with multiple parameters, storing them all in `SystemConfig` would get messy. This would also make mistakes during forks more likely since each rollup operator would need to evaluate their customizations to make sure they make sense for each fork.
