# [Project Name]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [Incorrect Linux/MIPS emulation](#incorrect-linuxmips-emulation)
  - [Unimplemented syscalls or opcodes needed by `op-program`](#unimplemented-syscalls-or-opcodes-needed-by-op-program)
  - [Insufficient memory in the program](#insufficient-memory-in-the-program)
  - [Failure to run correct VM based on prestate input](#failure-to-run-correct-vm-based-on-prestate-input)
  - [Mismatch between on-chain and off-chain execution](#mismatch-between-on-chain-and-off-chain-execution)
  - [Livelocks in the fault proof](#livelocks-in-the-fault-proof)
  - [Execution traces too long for the fault proof](#execution-traces-too-long-for-the-fault-proof)
  - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

*Italics are used to indicate things that need to be replaced.*

| | |
|--------|--------------|
| Author | *Author Name* |
| Created at | *2024-10-09* |
| Initial Reviewers | *Reviewer Name 1, Reviewer Name 2* |
| Need Approval From | *Security Reviewer Name* |
| Status | Draft |

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


## Failure Modes and Recovery Paths

### Incorrect Linux/MIPS emulation

- **Description:** An incorrectly implemented FPVM could result in an invalid fault proof. This can be caused by bugs in the thread scheduler, incorrect emulation of MIPS64 instructions, and so on.
- **Risk Assessment:** High severity, Low likelihood.
- **Mitigations:** Comprehensive testing. This includes full test coverage of every supported MIPS instruction, threading semantics, and verifying op-program execution on live chain data.
- **Detection:** op-dispute-mon forecasts and alerts on undesirable game resolutions.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Unimplemented syscalls or opcodes needed by `op-program`

- **Description:** We only aim to implement syscalls and opcodes that are required by `op-program` so there are some unimplemented. The risk is that there is some previously untested code path that uses an opcode or syscall that we haven't implemented and this code path ends up being exercised by an input condition some time in the future.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** We periodically use Cannon to execute the op-program using inputs from op-mainnet and op-sepolia. This periodic cannon runner (vm-runner) runs on oplabs infrastructure.
- **Detection:** Alerting is setup to notify the proofs team whenever the vm-runner fails to complete a cannon run.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Insufficient memory in the program

- **Description:** The op-program may run out of memory, causing it to crash.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** The 64-bit address space virtually eliminates memory exhaustion risks. Go's concurrent garbage collector automatically manages memory through scheduled background goroutines.
- **Detection:** op-dispute-mon forecasts and alerts on undesirable game resolutions that would result due to a program crash.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Failure to run correct VM based on prestate input

- **Description:** The off-chain Cannon [attempts to run the correct VM version based on the prestate input](https://github.com/ethereum-optimism/design-docs/blob/0034943e42b8ab5f9dd9ded2ef2b6b55359c922c/cannon-state-versioning.md). If it doesn't work correctly the on-chain steps would not match.
- **Risk Assessment:** Medium severity, low likelihood.
- **Mitigations:** Multicannon mitigates this issue by embedding a variety of cannon STFs into a single binary. This shifts the concern of ensuring the correct VM selection to multicannon. We also run multicannon on oplabs infra via the vm-runner, to assert the multicannon binary was built correctly.
- **Detection:** This can be detected by manual review. Failing that, it would only be detected when malicious activity occurs and an honest op-challenger fails to generate a fault proof.
- **Recovery Path(s)**: Fix the op-challenger multicannon configuration.

### Mismatch between on-chain and off-chain execution

- **Description:** There could be bugs in the implementation of either the Solidity or Go versions that make them incompatible with each other.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** Diffeerential testing asserts identical on-chain and off-chain execution.
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: Depends on the specifics. If the onchain VM implementation is "more correct", then fixing this can be done solely offchain. Otherwise, a governance vote will be needed. As usual, the [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f) provides the best guidance on this.

### Livelocks in the fault proof

- **Description:** A livelocked execution prevents an honest challenger from generating a fault proof.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:** Manual review of the op-program and a quick review of Go runtime internals. The op-program uses 3 threads, and only one of those threads is used by the mutator main function. This makes livelocks very unlikely.
- **Detection:** This would manifest as an execution that runs forever. Eventually, but well before the dispute period ends, op-dispute-mon will indicate that a game is forecasted to resolve incorrectly.
- **Recovery Path(s)**: See [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f).

### Execution traces too long for the fault proof

- **Description:** It's possible that introducing multi-threading/gc greatly increases the execution time of the op-program.
- **Risk Assessment:** Medium severity, low likelihood.
- **Mitigations:** Based on vm-runner executions of 64-bit Cannon and 32-bit single-threaded Cannon, the 64-bit VM executes the op-program much faster than the 32-bit VM. However, we can always use CPUs with better single-core performance to mitigate.
- **Detection:** op-dispute-mon provides an early forecast and triggers an alert if the op-challenger stops interacting with a game.
- **Recovery Path(s)**: This can be mitigated by migrating the op-challenger to a more powerful CPU. [Fault Proof Recovery](https://www.notion.so/oplabs/RB-000-Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f) provides guidance on when it'll be appropriate to do so.


### Generic items we need to take into account:
See [./fma-generic-hardfork.md](./fma-generic-hardfork.md). 

- [ ] Check this box to confirm that these items have been considered and updated if necessary.


## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Third-party audit the offchain and onchain VM implementation and specification (Assignee: @inphi)
