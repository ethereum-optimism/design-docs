# [Project Name]: Failure Modes and Recovery Path Analysis

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

*Italics are used to indicate things that need to be replaced.*

| | |
|--------|--------------|
| Author | Mark Tyneway |
| Created at | *2024-10-04* |
| Initial Reviewers | *Matt Solomon, Maurelian* |
| Need Approval From | *Matt Solomon* |
| Status | *Draft* |

## Introduction

This document covers the Standard L2 Genesis Feature. This feature exists to solve multiple problems:

- Enable releases of predeploys based on hardforks
- Enable simple upgrades of predeploys by removing network specific configuration from them and initializers
- Make a block history integrity check as simple as observing the genesis state root
- Simplifies management of the L2 Proxy Admin

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/97)
- [Tracking Ticket](https://github.com/ethereum-optimism/optimism/issues/12302)
- [Prototype Implementation](https://github.com/ethereum-optimism/optimism/pull/12057)

## Failure Modes and Recovery Paths

***Use one sub-header per failure mode, so the full set of failure modes is easily scannable from the table of contents.***

### Predeploys Lose Access to Chain Specific Configuration

- **Description:** On the upgrade block, a system deposit will perform a migration of network specific config from the current contracts that it is held in into the `L1Block` contract. For new chains, this network specific config is set via deposit transactions.
- **Risk Assessment:** High severity, low likelihood. This would result in withdrawals through the `L2CrossDomainMessenger` and the `L2StandardBridge` to be broken.
- **Mitigations:** Thorough unit and end to end tests should catch this. We have sufficient converage in `op-e2e` and the existing contracts unit tests to catch any of these issues. Spinning up a new chain end to end is important to make sure that the network specific config is deposited correctly into the chain.
- **Detection:** Simply observing that the chain specific config is accessible via the `L1Block` contract after the upgrade can give us assurance that the migration worked as expected.
- **Recovery Path(s)**: It is most likely that this could be fixed with a contract upgrade issued through the multisig. It is unlikely that a hardfork would be required to fix this issue.

### Network Specific Config is Not Initialized as First Transactions

- **Description:** It is theoretically possible that if the starting L1 block is not set correctly, then the network specific config will not be included as the first transactions in the block or it will be skipped. The `SystemConfig.startBlock` now becomes semantics of the protocol itself, as it is defined as the first block in which derivation should begin.
- **Risk Assessment:** High risk, low likelihood.
- **Mitigations:** We should ensure that whatever builds the rollup config uses this value. Ideally, this value is made optional on the rollup config and the value is simply fetched from the `SystemConfig`. This allows for backwards compatibility with older OP Stack chains as setting it can be an optional override. There is a world where the `startBlock` is filled in after the fact correctly on all `SystemConfig` contracts and we can delete it completely from the rollup config. We should also be sure to clearly note that this is an invariant as part of the protocol specs.
- **Detection:** This is only an issue for new chains, we could have "post checks" as part of the deployment system that ensure that the network specific config was set properly in the chain.
- **Recovery Path(s)**: The design works great in theory, it is just up to the tooling/client software to start derivation at the correct height. If a chain misses the events, the chain operator will need to reinitialize the `SystemConfig` for them to be passed through.

### Loss of Bridge Funds

- **Description:** It is technically possible that the bridge funds are lost due to a bug in the storage migration. There is a diff in the bridge contracts as part of this project, so it is important to thoroughly review and test the diff.
- **Risk Assessment:** High severity, low likelihood
- **Mitigations:** Thorough testing can prevent this issue from happening.
- **Detection:** We now have better process for handling contract upgrades that can catch bugs that previously have gone uncaught. The storage layout lock files will help a lot here.
- **Recovery Path(s)**: There is no coming back from lost bridge funds.

### [Name of Failure Mode 2]

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: *How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?*

### Generic items we need to take into account:
See [./fma-generic-hardfork.md](./fma-generic-hardfork.md). 

- [ ] Check this box to confirm that these items have been considered and updated if necessary.


## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ] Resolve all comments on this document and incorporate them into the document itself (Assignee: Mark Tyneway)

## Audit Requirements

*Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning.*
