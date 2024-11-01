# Holocene Hardfork: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Holocene Derivation](#holocene-derivation)
  - [The batcher violates the new (stricter) ordering rules](#the-batcher-violates-the-new-stricter-ordering-rules)
    - [Description](#description)
    - [Risk Assessment](#risk-assessment)
    - [Mitigations](#mitigations)
    - [Detection](#detection)
    - [Recovery Path(s)](#recovery-paths)
  - [Derivation Deadlock (either specification or op-node bug)](#derivation-deadlock-either-specification-or-op-node-bug)
    - [Description](#description-1)
    - [Risk Assessment](#risk-assessment-1)
    - [Mitigations](#mitigations-1)
    - [Detection](#detection-1)
    - [Recovery Path(s)](#recovery-paths-1)
- [Configurable EIP-1559 Parameters via SystemConfig](#configurable-eip-1559-parameters-via-systemconfig)
  - [Risk Assessment](#risk-assessment-2)
  - [Mitigations](#mitigations-2)
- [Update to the MIPS contract](#update-to-the-mips-contract)
  - [Undetected go1.22 changes](#undetected-go122-changes)
    - [Risk Assessment](#risk-assessment-3)
    - [Mitigations](#mitigations-3)
    - [Detection](#detection-2)
  - [Divergent FPVM implementations](#divergent-fpvm-implementations)
    - [Risk Assessment](#risk-assessment-4)
    - [Mitigations](#mitigations-4)
    - [Detection](#detection-3)
- [Generic Items](#generic-items)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |             |
| ------------------- | ----------- |
| Author              | George Knee, Sebastian Stammler, Roberto Boyardo, Mofi Taiwo |
| Created at          | 2024-09-24  |
| Needs Approval From | maurelian   |
| Other Reviewers     |             |
| Status              | Draft       |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers the Holocene hardfork, which involves the following changes:

- (Consensus Layer) Holocene Derivation
- (Execution Layer & Smart Contracts) Configurable EIP-1559 Parameters via SystemConfig
- (Smart Contracts) Update to the MIPS contract

Each change has it's own section below with a list of Failure Modes.

Below are references for this project:

- [PID: Holocene hardfork upgrade](https://www.notion.so/PID-Holocene-hardfork-upgrade-00ee1ffc414a407088fdb49841771527?pvs=21)
- [Github tracker](https://github.com/orgs/ethereum-optimism/projects/84/views/6)
- [Specs](https://specs.optimism.io/protocol/holocene/derivation.html?highlight=holocene#holocene-derivation)

## Holocene Derivation

### The batcher violates the new (stricter) ordering rules

#### Description

If the batcher:

- sends transactions out of order, or
- orders frames within transactions out of order, or
- orders batches within a channel out of order

This will lead to the consensus client hitting error conditions when it loads the frames from L1. According to the spec, frames will be dropped in order to maintain a contiguous frame queue, and out-of-order batches will be dropped at a later stage in the derivation pipeline. The safe chain would halt until the next batch can be resubmitted in order.

#### Risk Assessment

low severity / low likelihood

#### Mitigations
We have identified the following opportunities for the current batcher implementation to violate the Holocene rules:
- Blocks could be prepended to the batcher's internal queue when a channel times out on chain. The blocks which come after may have already been dequeued. E.g `[0,1,2,3,4,5] + dequeue([0,1]) => [2,3,4,5] + dequeue([2,3]) => [4,5] + timeout+prepend([0,1]) = [0,1,4,5]` which has an invalid order.
- A channel could be removed from the queue out of order, if the transaction receipts are received out of order. `[A,B,C] + txConfirmed(B) = [A,C]` which has an invalid order.
- A frame could be enqueued out of order when a tx fails. Again if the receipt arrives out or order `[a1,a2] => nextFrame() => [a2] + nextFrame() => [] + txFailed(a2) => [a2] + txFailed+enqueue(a1) => [a2,a1]` which has an invalid order.

Action items described below will remove these possibilities.

#### Detection

L2 safe chain halt

#### Recovery Path(s)

We would need to get the batcher to resubmit the transaction sent out of order. No governance or hardfork needed to recover. A simple batcher restart would cause the batcher to re-batch the unsafe chain, which should lead to recovery. Moreover, we could also temporarily operate the batcher with reduced tx sending concurrency, which should avoid out of order txs.

## Configurable EIP-1559 Parameters via SystemConfig

The Holocene upgrade introduces the ability to update the EIP-1559 ELASTICITY_MULTIPLIER and BASE_FEE_MAX_CHANGE_DENOMINATOR parameters through a SystemConfig call. Previously, these parameters were fixed at 6 and 250 respectively as of the Canyon upgrade. Allowing these parameters to be configurable gives the chain operator more flexibility in adjusting important chain properties such as the gas target, and how fast base fee adjust in response to demand changes.

This change required updating both op-geth and op-node. The op-geth changes are minimal and affect only the engine API to pass in the new parameters via NewPayloadV3, and the base fee calculation to use the provided parameters instead of the hardcoded configuration once Holocene is active. The op-node changes involve handling the new ConfigUpdate event and appropriately passing this information to op-geth via the execution API.

The change also involved updates to the L1 SystemConfig contract in order to allow the chain operator to update the parameters and generate the associated ConfigUpdate event.

### Risk Assessment

medium severity / low likelihood

### Mitigations

The upgrade is designed to function even if the SystemConfig contract does not get updated. In this case, the system falls back to the Canyon hardcoded parameters, and the system will perform base fee updates exactly as prior to the upgrade. However, it is critical that the contracts upgrade does not happen _before_ the hardfork activates, because this would cause a chain halt due to op-node not recognizing the new config update type. 

We are performing multi-client testing (op-geth and op-reth) and will run a multi-client devnet, so bugs in either will be quickly detected by consensus disagreements among them.


## Update to the MIPS contract

As part of the Holocene change, the Go compiler was updated from 1.21 to 1.22. This impacts the op-program as the go1.22 runtime makes additional syscalls that are not supported by the pre-Holocene MIPS Fault Proof Virtual Machine (FPVM).
As such, Holocene includes an update to MIPS.sol that supports go1.22 programs. This change to the FPVM is very minimal; an update to the `fcntl` syscall emulation that was partially implemented by the pre-Holocene FPVM.

### Undetected go1.22 changes

If there were other changes introduced by go1.22 beyond `fcntl`, and the MIPS FPVM does not emulate them correctly, it could result in blocks that are not fault-provable. This ultimately results in fault dispute games resolving incorrectly.

#### Risk Assessment

medium severity / low likelihood

#### Mitigations

[An audit](https://gist.github.com/3docSec/068537844f2ddb204324079138f14551) of the go1.22 changes on the op-program and the FPVM was performed by [3DOC Security](https://x.com/3docSec). The audit did not find any problems related to the go1.22 changes that breaks op-program execution in the FPVM.

Furthermore, any game that resolves incorrectly is subject to the 3.5-day finality delay. This gives the Security Council ample time to detect and respond to invalid games (including blacklisting games and falling back to to permissioned games).


#### Detection

`dispute-mon` detects invalid games and forecasts those that will be resolved incorrectly.

### Divergent FPVM implementations

If the offchain FPVM behaves differently from the MIPS FPVM contract, the op-challenger will be unable to act honestly.

#### Risk Assessment

medium severity / low likelihood

#### Mitigations

The offchain FPVM, unencumbered by governance, can be updated at any time to match the behavior of the MIPS contract.

#### Detection

To reduce the risk of discrepancies, we differentially test the MIPS contract against the offchain FPVM implementation.

## Generic Items

See [./fma-generic-hardfork.md](./fma-generic-hardfork.md).

- [ ] Check this box to confirm that these items have been considered and updated if necessary.

## Audit Requirements

The changes to the MIPs contract were subject to an audit, the report is available [here](https://gist.github.com/3docSec/068537844f2ddb204324079138f14551). An audit has not been deemed necessary for the relatively simple changes to the SystemConfig contract, which has been reviewed by OPLabs and Base internal teams.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [x] (BLOCKING): Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)
- [ ] (BLOCKING): Action tests will be added which are run on op-node and Kona https://github.com/ethereum-optimism/optimism/issues/12449
- [ ] (BLOCKING): The changes will be deployed to a devnet with both geth and reth running
- [ ] (BLOCKING): we will re-architect the batcher to eliminate the cases above
- [ ] (BLOCKING): we will add functionality to the batcher to periodically detect when the safe chain should have progressed but didn't, and to recover from that
- [ ] (BLOCKING): the script(s) or tooling for the `SystemConfig` upgrade contract will exit with an error if the activation time has not yet passed.
- [ ] (non-BLOCKING): We will update the op-sepolia vm-runner to use the new FPVM. The vm-runner runs the op-program in the MIPS FPVM using inputs sampled from a live chain. Having the vm-runner run the op-program on op-sepolia for a couple days will increase confidence that the network will continue to be fault provable.

Additional action items are copied here from the [generic hardfork FMA](./fma-generic-hardfork.md) doc:

- [ ] (BLOCKING): We have implemented extensive unit and end-to-end testing of the activation flow.
- [ ] (BLOCKING): We have implemented multi-client testing to reduce the chance of bugs
- [ ] (BLOCKING): We have implemented extensive cross-client / differential testing of the new functionality.
- [ ] (BLOCKING): We will be testing the activation on our devnets and testnets.
- [ ] (BLOCKING): We have prepared datadir backups close before the upgrade, so we can use these in an emergency to rollback.
- [ ] (BLOCKING): We have updated the runbook for recovering from a hardfork activation chain halt (including rolling back contract changes), if necessary. See https://oplabs.notion.site/RB-000-How-To-Rewind-a-Network-c21f628205354dbdbed9c691b2455a7c?pvs=74.
- [ ] (non-BLOCKING): The superchain-ops task to upgrade any contract should check if the semantic versions and bytecodes after the upgrade are as expected. 
- [ ] (non-BLOCKING): We have implemented fuzz testing in a kurtosis multi-client devnet to reduce the chance of bugs

