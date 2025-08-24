# [Interop Proofs]: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[Name of Failure Mode 1]](#name-of-failure-mode-1)
  - [[Name of Failure Mode 2]](#name-of-failure-mode-2)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)
  - [Appendix A: This is a Placeholder Title](#appendix-a-this-is-a-placeholder-title)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Italics are used to indicate things that need to be replaced._

|                    |                          |
|--------------------|--------------------------|
| Author             | Adrian Sutton            |
| Created at         | 2025-05-08               |
| Initial Reviewers  | Mofi Taiwo, Paul Dowman  |
| Need Approval From | _Security Reviewer Name_ |
| Status             | Draft                    |

> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team.
> - Maintain the “Status” property accordingly. An FMA document can have the following statuses:
    >

- **Draft 📝:** Doc is created but not yet ready for review.

> - **In Review 🔎:** Security is reviewing, and Engineering is iterating on the design. A checklist of action items
    will be created during this phase.
    >

- **Implementing Actions 🛫:** Security has signed off on the content of the document, including the resulting action
  items. Engineering is responsible for implementing the action items, and updating the checklist.

> - **Final 👍:** Security will transition the status of the document to Final once all action items are completed.

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the project. The FMA
    review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers the changes made to the fault proof system to support interop. These are details in the
[interop fault proofs spec](https://specs.optimism.io/interop/fault-proof.html)

### Shared DisputeGameFactory

The `DisputeGameFactory` is now shared by all chains in a dependency set. Proposals are made using super roots which
include the output root for all chains in the dependency set. A single fault dispute game is used to decide the validity
of the state of all chains.

### SuperFaultDisputeGame and SuperPermissionedDisputeGame

Two new game implementations are provided:

1. `SuperFaultDisputeGame` which replaces `FaultDisputeGame`
2. `SuperPermissionedDisputeGame` which replaces `PermissionedDisputeGame`

Both of these contracts are largely the same as the pre-interop versions they replace. The key changes include:

* Removing code to support challenging the L2 block number (now handled in the fault proof program)
* Preventing games being created with the root claim `keccak("invalid")` which is used as a marker for an always invalid
  state
* Modifying the L2BlockNumber local key to always populate the PreimageOracle with the proposal timestamp for the game,
  regardless of the claim's position in the dispute tree

### Multistage Fault Proof Program

The fault proof program (`op-program` and `kona`, though only `op-program` is currently in production) has been updated
to use separate steps to derive the chain. This ensures that the fault proof VM can execute each step in a reasonable
amount of time. In particular it ensures that a single step only needs to execute the full block from a single chain,
keeping resource usage roughly equivalent to pre-interop. There are a fixed 128 steps per timesstamp transition. The
first two steps reproduce the next block for a single chain from the L1 batch data without verifying executing messages.
Following steps are a no-op, to simplify the addition of more chains in the future. The final step verifies executing
messages across all chains and replaces any blocks found to be invalid with deposit-only blocks.

### Invalid Proposal Timestamp Handling

The L2 block number challenge is removed from the contracts and replaced by the fault proof program transitioning to
the invalid state (`keccak("invalid")`) if data is unavailable on L1 to reach the proposal timestamp. Previously,
reaching the L1 head was considered the end of derivation and simple trace extension was applied. This left ambiguity
as to whether the derivation had reached the proposal block (indicating it is valid) or had reach the L1 head (invalid).

### op-challenger Updates

`op-challenger` has been updated to handle the new game types. `op-supervisor` is now used as its primary source of
truth. The `TraceProvider` used for the top half of the game is a new implementation that follows the multi-step state
transition used by interop.

### op-dispute-mon Updates

`op-dispute-mon` has been updated to support the name game types. `op-supervisor` is now used as its primary source of
truth for proposals, using super roots instead of output roots. Monitoring is otherwise unchanged.

## Failure Modes and Recovery Paths

### FM1: Unavailability of Preimage Data

- **Description:**
  - The final consolidation step validates executing messages. To do this it must have access to the receipts from the
    block as derived from L1 batch data.
  - When that block is invalid, it will have been re-orged out of the canonical chain by honest nodes and may have been
    pruned from its database.
  - As the fault proof program derives the original optimistic block in a separate step it does not have the receipts.
- **Risk Assessment:**
  - Medium impact, low likelihood
- **Mitigations:**
  - op-geth does not currently prune data from non-canonical blocks so would always have the required data available.
  - Interop proofs action tests run using a source node that only contains canonical blocks.
  - We periodically execute op-program using inputs from op-mainnet and op-sepolia. This periodic cannon
    runner ([vm-runner]) runs on oplabs infrastructure using source nodes available to op-challenger. The vm-runner
    samples game inputs for the latest L2 safe head every 2 hours and uses cannon to execute the op-program using the
    sampled inputs. Note that this sampling does not include every game created.
- **Detection:**
  - Existing monitoring for forecast or actual invalid game resolution
- **Recovery Path(s):**
  - Follow the [Fault Proof Recovery Runbook]
  - Deploy a fixed op-challenger. Does not require governance approval or a hard fork.
  - If sufficient context is not available from the preimage hint emitted by the fault proof program, deploy an updated
    prestate with additional information in the hint. Requires governance approval but can be done without a hard fork.

### FM2: Problem in Game Design

- **Description:**
  - A problem in the design of the dispute game, in particular in the multistep super root state transition, could lead
    to games resolving incorrectly.
  - This may include valid proposals being invalidated and invalid proposals being confirmed.
- **Risk Assessment:**
  - Medium impact, low likelihood
- **Mitigations:**
  - [Suite of action tests] verifying the sequence of expected states to transition between super roots in a wide
    variety of cases.
- **Detection:**
  - Existing monitoring for forecast and actual incorrect game results.
- **Recovery Path(s):**
  - Follow the [Fault Proof Recovery Runbook]
  - A fixed design would need to be devised, implemented, approved by governance and deployed.

### FM3: Mismatch Between op-challenger and op-program

- **Description:**
  - To ensure it always wins games, op-challenger must calculate claim values that match the claim op-program considers
    correct.
  - A bug in op-challenger or op-program could cause these claims to be mismatched.
  - This could lead to games resolving incorrectly, including valid proposals being invalidated and invalid proposals
    being confirmed.
- **Risk Assessment:**
  - Medium impact, low likelihood
- **Mitigations:**
  - [Suite of action tests] verifying the honest trace used by op-challenger matches the expected claims by op-program.
  - Bottom half of the game continues to post cannon states as in pre-interop games.
- **Detection:**
  - Existing monitoring for forecast and actual incorrect game results.
  - We periodically execute op-program in [vm-runner] using inputs from op-mainnet and op-sepolia. The expected output
    is calculated using the same op-challenger trace provider as is used to calculate the correct claims when playing
    games.
- **Recovery Path(s):**
  - Follow the [Fault Proof Recovery Runbook]
  - If the bug is in op-challenger, deploy a fixed version. Does not require governance approval or a hard fork.
  - If the bug is in op-program, deploy a fix via a new prestate. Requires governance approval but can be done without a
    hard fork.

### FM4: Increased Maximum Preimage Size

- **Description:**
  - The consolidation step introduces the first requirement for op-program to read receipts from L2 chains.
  - Since L2 chains have higher gas limits it is possible to create larger receipts in a single block which may need to
    be added into the PreimageOracle.
  - The larger preimages may incur higher costs for the honest actor to populate the `PreimageOracle`.
  - The preimage can still be posted via the existing large preimage proposal process which op-challenger automatically
    utilises.
- **Risk Assessment:**
  - Low impact, low likelihood
- **Mitigations:**
  - Preimages only need to be populated into the `PreimageOracle` when the max depth is reached and `step()` is called.
    This requires posting at least 300 ETH in bonds.
- **Detection:**
  - [Existing monitoring for large preimage proposals](https://github.com/ethereum-optimism/k8s/blob/0eb3b759ecfe52ed36ece0531a559f11e699419f/grafana-cloud/terraform-rules/challenger.yaml#L94-L102)
- **Recovery Path(s):**
  - Automatically handled by op-challenger
  - Potentially increase bond sizes via new dispute game implementation deployment (requires governance approval).

### Generic items we need to take into account:

See [generic hardfork failure modes](./fma-generic-hardfork.md)
and [generic smart contract failure modes](./fma-generic-contracts.md).
Incorporate any applicable failure modes with FMA-specific mitigations and detections directly into this document.

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure
they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: document author)


## Audit Requirements

An audit has been performed for op-program and is scheduled for the overall interop system, including fault proofs.

## Appendix

### Appendix A: Additional Interop FMAs

Other interop FMAs have some overlap with the proofs system:

* [Supervisor FMA](./fma-supervisor.md) includes failure modes around unavailability or incorrect results from op-supervisor
* [Portal FMA](./fma-interop-portal.md) includes failure modes related to the contract migration

[Suite of action tests]: https://github.com/ethereum-optimism/optimism/blob/fa86f81da6bed8489508907ed0956134d029c09f/op-e2e/actions/interop/proofs_test.go

[Fault Proof Recovery Runbook]: https://www.notion.so/oplabs/Fault-Proofs-Recovery-Runbook-8dad0f1e6d4644c281b0e946c89f345f?pvs=4

[vm-runner]: https://github.com/ethereum-optimism/optimism/tree/develop/op-challenger/runner
