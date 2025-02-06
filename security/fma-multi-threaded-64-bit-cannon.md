# Multithreaded Cannon: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [Incorrect Linux/MIPS emulation](#incorrect-linuxmips-emulation)
  - [Unimplemented syscalls or opcodes needed by `op-program`](#unimplemented-syscalls-or-opcodes-needed-by-op-program)
  - [Insufficient memory in the program](#insufficient-memory-in-the-program)
  - [Compromised Control Flow in the program](#compromised-control-flow-in-the-program)
  - [Failure to run correct VM based on prestate input](#failure-to-run-correct-vm-based-on-prestate-input)
  - [Mismatch between on-chain and off-chain execution](#mismatch-between-on-chain-and-off-chain-execution)
  - [Livelocks in the fault proof](#livelocks-in-the-fault-proof)
  - [Execution traces too long for the fault proof](#execution-traces-too-long-for-the-fault-proof)
  - [Invalid `DisputeGameFactory.setImplementation` execution](#invalid-disputegamefactorysetimplementation-execution)
- [Action Items](#action-items)
- [Audit requirements](#audit-requirements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

| | |
|--------|--------------|
| Author | Paul Dowman, Mofi Taiwo |
| Created at | *2024-10-09* |
| Initial Reviewers | Matt Solomon |
| Need Approval From | Tom Assas |
| Status | In Review |

> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team. 
> - Maintain the “Status” property accordingly. An FMA document can have the following statuses:
>   - **Draft 📝:** Doc is created but not yet ready for review.
>   - **In Review 🔎:** Security is reviewing, and Engineering is iterating on the design. A checklist of action items will be created during this phase.
>   - **Implementing Actions 🛫:** Security has signed off on the content of the document, including the resulting action items. Engineering is responsible for implementing the the action items, and updating the checklist.
>   - **Final 👍:** Security will transition the status of the document to Final once all action items are completed.

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the  project. The FMA review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers the conversion of the [Cannon Fault Proof VM](https://docs.optimism.io/stack/protocol/fault-proofs/cannon) to support multi-threading and 64-bit architecture. These changes increase addressable memory and support better memory management by unlocking garbage collection in the op-program. 

The multi-threaded Fault Proof VM is specified [here](https://github.com/ethereum-optimism/specs/blob/3abc17a68727e22c31a7a113be935943f717ee63/specs/experimental/cannon-fault-proof-vm-mt.md). The current version of the MT-Cannon contract can be found [here](https://github.com/ethereum-optimism/optimism/blob/68f77aaa317b9184cbbcd1526bc57bce1722906b/packages/contracts-bedrock/src/cannon/MIPS64.sol).

## Failure Modes and Recovery Paths

### Incorrect Linux/MIPS emulation

- **Description:** An incorrectly implemented FPVM could result in an invalid fault proof. This can be caused by bugs in the thread scheduler, incorrect emulation of MIPS64 instructions, and so on.
- **Risk Assessment:** High severity, Low likelihood.
- **Mitigations:** Comprehensive testing. This includes full test coverage of every supported MIPS instruction, threading semantics, and verifying op-program execution on live chain data.
  This includes [unit and fuzz](https://github.com/ethereum-optimism/optimism/tree/eabf70498f68f321f5de003f1d443d3e3c8100b8/cannon/mipsevm/tests) testing of MIPS instructions and Linux syscalls. It also includes [testing](https://github.com/ethereum-optimism/optimism/blob/eabf70498f68f321f5de003f1d443d3e3c8100b8/cannon/mipsevm/multithreaded/state_test.go) of multithreaded specific functionality.
- **Detection:** [op-dispute-mon](https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon#readme) forecasts and alerts on undesirable game resolutions.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Unimplemented syscalls or opcodes needed by `op-program`

- **Description:** We only aim to implement syscalls and opcodes that are required by `op-program` so there are some unimplemented. It's not feasible with the time and resources available to implement an entirely general purpose VM and Linux, and this would result in a lot of additional risk due to the added complexity and unused code paths. The risk is that there is some previously untested code path that uses an opcode or syscall that we haven't implemented and this code path ends up being exercised by an input condition some time in the future.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** We periodically use Cannon to execute the op-program using inputs from op-mainnet and op-sepolia. This periodic cannon runner ([vm-runner](https://github.com/ethereum-optimism/optimism/tree/develop/op-challenger/runner)) runs on oplabs infrastructure. The vm-runner samples game inputs for the latest L2 safe head every 2 hours and uses cannon to execute the op-program using the sampled inputs. Note that this sampling does not include every game created.
Furthermore, we [sanitize](https://github.com/ethereum-optimism/optimism/blob/eabf70498f68f321f5de003f1d443d3e3c8100b8/cannon/Makefile#L51) the op-program [in CI](https://github.com/ethereum-optimism/optimism/blob/eabf70498f68f321f5de003f1d443d3e3c8100b8/.circleci/config.yml#L928C1-L929C111) for unsupported opcodes.
- **Detection:** [Alerting](https://github.com/ethereum-optimism/k8s/blob/master/grafana-cloud/rules/challenger.yaml#L90-L110) is setup to notify the proofs team whenever the vm-runner fails to complete a cannon run. And the CI check provides an early warning against unsupported opcodes.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Insufficient memory in the program

- **Description:** The op-program may run out of memory, causing it to crash.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** The 64-bit address space virtually eliminates memory exhaustion risks. Go's concurrent garbage collector automatically manages memory through scheduled background goroutines.
- **Detection:** [op-dispute-mon](https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon#readme) forecasts and alerts on undesirable game resolutions that would result due to a program crash.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Compromised Control Flow in the program

- **Description:** This could theoretically occur when the op-program runs out of memory in a way that lets the attacker reuse code to subvert execution.
- **Risk Assessment:** High severity, low likelihood.
  - Low likelihood: This requires an attacker to craft inputs that not only induce high memory usage, but also corrupt or spray the heap in a way that either produces invalid fault proofs or prevents valid fault proofs from being generated.
- **Mitigations:** As with [Insufficient memory in the program](#insufficient-memory-in-the-program), the 64-bit address space effectively prevents this from occurring. Furthermore, the Go runtime checks memory allocations against heap corruption. However, such memory protections may not hold due to bugs in the Go runtime. Using the `unsafe` package in go can lead to unexpected behavior and we don't use it in op-program.
- **Detection:** [op-dispute-mon](https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon#readme) forecasts and alerts on undesirable game resolutions that would result due to honest claims being disputed at the bottom of the game tree.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Unknown Absolute Prestate

This absolute prestate is essentially a hash of the op-program memory loaded by Cannon, including its code.
As such, the absolute prestate (or simple prestate) encodes the op-program. The MT-Cannon VM loads the op-program differently from single threaded Cannon.
This warrants a new absolute prestate _hash_ to properly configure the `FaultDisputeGame` implementations for the MT-Cannon VM.

- **Description:** This is when there is not a known prestate (preimage) for a given absolute prestate hash in the dispute game implementations.
- **Risk Assessment:** High severity, Low likelihood.
- **Mitigations:** Every absolute prestate is built off of an op-program release tag. The prestate is [build is reproducible](https://github.com/ethereum-optimism/optimism/blob/68f77aaa317b9184cbbcd1526bc57bce1722906b/op-program/Dockerfile.repro) such that the same prestate is emitted regardless of the environment.
Furthermore, governance and Guardian signers will be instructed to reproduce the prestate build themselves and check that the prestate hash matches the op-program release that will be referenced in the MT-Cannon governance post.
Lastly, there exists a [CI check in the monorepo](https://github.com/ethereum-optimism/optimism/blob/68f77aaa317b9184cbbcd1526bc57bce1722906b/.circleci/config.yml#L1575) asserting that every op-program github release tag maps to the expected absolute prestate hash. This ensures that op-program releases stay reproducible.
- **Detection:** The vm-runner is configured to use the on-chain absolute prestate for cannon executions. Any failures in the vm-runner triggers an alert to the proofs team.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Failure to run correct VM based on absolute prestate input

- **Description:** The off-chain version of Cannon [attempts to run the correct VM version based on the absolute prestate input](https://github.com/ethereum-optimism/design-docs/blob/0034943e42b8ab5f9dd9ded2ef2b6b55359c922c/cannon-state-versioning.md). If it doesn't work correctly the on-chain steps would not match.
- **Risk Assessment:** Medium severity, low likelihood.
- **Mitigations:** Multicannon mitigates this issue by embedding a variety of cannon STFs into a single binary. This shifts the concern of ensuring the correct VM selection to multicannon. We also run multicannon on oplabs infra via the vm-runner, to assert the multicannon binary was built correctly.
- **Detection:** This can be detected by manual review. Failing that, it would only be detected when malicious activity occurs and an honest op-challenger fails to generate a fault proof.
- **Recovery Path(s)**: Fix the op-challenger multicannon configuration.

### Mismatch between on-chain and off-chain execution

- **Description:** There could be bugs in the implementation of either the Solidity or Go versions that make them incompatible with each other.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** [Diffeerential testing](https://github.com/ethereum-optimism/optimism/tree/eabf70498f68f321f5de003f1d443d3e3c8100b8/cannon/mipsevm/tests) asserts identical on-chain and off-chain execution.
- **Detection:** An op-challenger fails to fault prove an invalid claim using a witness generated offchain.
- **Recovery Path(s)**: Depends on the specifics. If the off-chain version is changed to match the onchain VM implementation, then fixing this can be done solely offchain. In the opposite case (changing the onchain version to match the off-chain version) a governance vote will be needed. As usual, the [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f) provides the best guidance on this.

### Livelocks in the fault proof

- **Description:** A livelocked execution prevents an honest challenger from generating a fault proof.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** Manual review of the op-program and a quick review of Go runtime internals. The op-program uses 3 threads, and only one of those threads is used by the mutator main function. This makes livelocks very unlikely. We looked into the livelock problem and there are possible solutions but they're deferred for future work as the risk of a livelock is considered too low to be addressed immediately.
- **Detection:** This would manifest as an execution that runs forever. Eventually, but well before the dispute period ends, op-dispute-mon will indicate that a game is forecasted to resolve incorrectly.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Execution traces too long for the fault proof

- **Description:** It's possible that introducing multi-threading/gc greatly increases the execution time of the op-program.
- **Risk Assessment:** Medium severity, low likelihood.
- **Mitigations:** Based on [vm-runner executions of 64-bit Cannon and 32-bit single-threaded Cannon](https://optimistic.grafana.net/goto/IlD3Tu4Hg?orgId=1), the 64-bit VM executes the op-program much faster than the 32-bit VM. As such, we do not expect worse performance over the existing VM, which was already found to be adequate. However, we can always use CPUs with better single-core performance to mitigate.
- **Detection:** op-dispute-mon provides an early forecast (see [this alerting rule](https://github.com/ethereum-optimism/k8s/blob/6416658dd9602f83c6be093c5c57af45877258a3/grafana-cloud/rules/dispute-mon.yaml#L154)) and triggers an alert if the op-challenger stops interacting with a game.
- **Recovery Path(s)**: This can be mitigated by migrating the op-challenger to a more powerful CPU. [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f) provides guidance on when it'll be appropriate to do so.

### Invalid `DisputeGameFactory.setImplementation` execution

The `DisputeGameFactory.setImplementation` will be used to upgrade both the `CANNON` and `PERMISSIONED_CANNON` game types. Note that this means this upgrade will not require changing the `respectedGameType` in the `OptimismPortal` as `CANNON` games will continue to be used to finalize outputs, albeit with a different VM.
As such, there will be a brief moment where there are two sets of `CANNON` games that using singlethreaded and multithreaded VMs.

- Description: This occurs when either the call to the DisputeGameFactory could not be made due to grossly unfavorable base fees on L1, an invalidly approved safe nonce, or a successful execution to a misconfigured dispute game implementation.
- **Risk Assessment:** Low severity, low likelihood.
  - Low Likelihood: The low likelihood is a result of tenderly simulation testing of safe transactions, code review of the upgrade playbook, and manual review of the dispute game implementations (which are deployed on mainnet and specified in the governance proposal so they may be reviewed).
  - Low severity: Fault Proofs continues to use the existing single-threaded FPVM. This carries a reputational risk, but it doesn't diminish the security of the system. Withdrawals will continue to work against outputs secured by the single-threaded FPVM.
- **Mitigations:** No immediate action is needed other than to retry the safe transaction. This may require another signing ceremony. Note that the op-challenger does not need to be rolled back, as multicannon is backwards compatible with older FPVM state transition functions.
- **Detection:** An un-executed safe transaction is easily detectable.
- **Recovery Path(s)**: Reschedule upgrade, possibly releasing new binary though without immediate urgency.


## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Third-party audit the offchain and onchain VM implementation and specification (Assignee: @inphi)
- [ ] [Add a healthcheck for the vm-runner (Assignee: @pauldowman)](https://github.com/ethereum-optimism/k8s/pull/5424)

## Audit requirements

An audit of the multithreaded VM is not required per the [OP Labs Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864).
A failure in the new Cannon VM and thus dispute games is mitigated by an airgap in finalized withdrawals. Furthermore, there's a window whereby the Security Council can override the results of invalid games.
Nonetheless, we will be auditing the new VM.
