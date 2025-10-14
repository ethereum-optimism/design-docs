<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Generic items we need to take into account for any hardfork:](#generic-items-we-need-to-take-into-account-for-any-hardfork)
  - [Chain Halt at Activation](#chain-halt-at-activation)
  - [Activation Failure without chain halt: L1 transactions](#activation-failure-without-chain-halt-l1-transactions)
  - [Activation Failure without chain halt: Node software](#activation-failure-without-chain-halt-node-software)
  - [Invalid `DisputeGameFactory.setImplementation` execution](#invalid-disputegamefactorysetimplementation-execution)
- [Chain split (across clients)](#chain-split-across-clients)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### Generic items we need to take into account for any hardfork:

#### Chain Halt at Activation

- **Description:** Hard forks carry the risk of bugs that cause chain halts at activation time. This could be due to L1 contract changes not applying properly, and/or a configuration or implementation bug in the node software or protocol specification.

- **Risk Assessment:** Medium severity as no funds are at risk, though it is a full liveness failure that would be messy to recover from and require releasing and distributing an updated configuration and/or release. Likelihood is low, especially if we use the mechanics for dynamically upgrading L2 contracts established in the Ecotone hardfork: namely, inserting appropriate deposit transactions in the activation block.

- **Mitigations:** 

- [ ] ACTION ITEM (BLOCKING): We have implemented extensive unit and end-to-end testing of the activation flow.
- [ ] ACTION ITEM (BLOCKING): We have implemented multi-client testing to reduce the chance of bugs
- [ ] ACTION ITEM (non-BLOCKING): We have implemented fuzz testing in a kurtosis multi-client devnet to reduce the chance of bugs
- [ ] ACTION ITEM (BLOCKING): We will be testing the activation on our devnets and testnets.

- **Detection:** Detection is straightforward as the chain will stop producing blocks. On OP Mainnet, P1 alarms are triggered and on-call engineers are paged if the unsafe head does not increase for 1 minute or if the safe head does not increase for 15 minutes. Moreover, the chain will be closely monitored during activation.

- **Recovery Path(s)**: Would not require a vote or hardfork, but we’d likely have to coordinate a chain config update that pushed back the date of the upgrade, and allowed node operators to rollback any bad blocks. Estimated sequencer downtime is 30 min in a worst-case scenario where we have to reset the chain back to a block before the activation and disable the hardfork activation. Additional steps would be required from infra providers to get back to the healthy chain. They would need to restart their op-node and op-geth with activation override command line flags/env var.

    - [ ] ACTION ITEM (BLOCKING): We have prepared datadir backups close before the upgrade, so we can use these in an emergency to rollback.

    - [ ] ACTION ITEM (BLOCKING): We have updated the runbook for recovering from a hardfork activation chain halt (including rolling back contract changes), if necessary. See https://oplabs.notion.site/RB-000-How-To-Rewind-a-Network-c21f628205354dbdbed9c691b2455a7c?pvs=74.

#### Activation Failure without chain halt: L1 transactions

- **Description:** Any contract upgrades which were scheduled may fail.

- **Risk Assessment:** Low severity.

- **Mitigations:** Same as above.

- **Detection:** We can read the state of L1 after the activation time and compare it with expectations

- [ ] ACTION ITEM (non-BLOCKING): The superchain-ops task to upgrade any contract should check if the semantic versions and bytecodes after the upgrade are as expected. 

- **Recovery Path(s)**:  
Since there is no chain halt, we can just live with it and fix it in an upcoming upgrade.

#### Activation Failure without chain halt: Node software

- **Description:** The upgrade may be misconfigured in the node software (e.g. superchain-registry dependency was not updated) and fail to take place.

- **Risk Assessment:** Low severity if there were no L1 contract changes or if there were L1 contract changes but they also failed to apply, since the chain would continue on as if the upgrade never happened. 

- **Mitigations:** As above

- **Detection:**  Node software startup logs should indicate whether the hardfork has activated properly.

- **Recovery Path(s)**: Reschedule the upgrade, releasing a new binary (without immediate urgency). 


#### Invalid `DisputeGameFactory.setImplementation` execution

- **Description:** This occurs when either the call to the `DisputeGameFactory` could not be made due to grossly unfavorable base fees on L1, an invalidly approved safe nonce, or a successful execution to a misconfigured dispute game implementation.

- **Risk Assessment:**
    - Low likelihood. The low likelihood is a result of tenderly simulation testing of safe transactions, code review of the upgrade playbook, and manual review of the dispute game implementations (which are deployed on mainnet and specified in the governance proposal so they may be reviewed).
    - High impact as Fault proofs remains incompatible with the new hardfork which would brick new bridge withdrawals. If we did nothing, i.e. ignored dispute-mon alerts and did not apply the FP recovery runbook, then an attacker would be able to steal the bonds of “honest” challengers that have upgraded their op-program by using the non-upgraded fork chain to generate a fault proof. In this scenario, the attacker would be unable to steal funds from the bridge so long as there exists an honest challenger that is configured to generate pre-upgrade execution traces. That said, the actual impact would be equivalent to [FP recovery](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21) as it would trigger the runbook, where we would have the opportunity to use overrides to recover the system.

- **Mitigations:** There are two cases:
    - If the safe transaction couldn’t be successfully executed, then: Revert `OptimismPortal` to use the permissioned fault dispute game as its respected game type. Depending on how much time till the upgrade, a superchain pause would be needed in order to gather necessary the signatures to change the `OptimismPortal`.
    - If the game implementation was misconfigured and this was detected prior to fork activation, then do the above. If detection occurs post-fork activation, then follow the [Fault Proofs Recovery Runbook](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21) (includes the [Superchain Pause Incident Response Runbook](https://docs.google.com/document/d/1ooKn16PYXl_iKkv9sK4CemzgqoiQkzUX-ufPzfY9L20/edit?pli=1&tab=t.0).).

- **Detection:** An un-executed safe transaction is easily detectable. In the case of a misconfigured game implementation, the op-dispute-mon will alert proofs-squad and security on any attempt to exploit this misconfiguration.

- **Recovery Path(s)**: Reschedule the upgrade, releasing a new binary (without immediate urgency).

### Chain split (across clients)

- **Description:** Differences in implementation across clients (e.g. `op-geth` versus `op-reth`) cause a chain split due to different effective consensus rules.
- **Risk Assessment:** medium severity / medium likelihood
- **Mitigations:**
- [ ] ACTION ITEM (BLOCKING): We have implemented extensive cross-client / differential testing of the new functionality for both scenarios a) when chains activate the fork at genesis and b) when they activate it after genesis. This is part of our alphanet / betanet [devnet process](https://devnets.optimism.io/).
- **Detection:** Replicas of one kind of client will diverge from the sequencer. If the bug made it to production, replica healthcheck alerts would fire as nodes diverged.
- **Recovery Path(s)**: Divergent clients would need to be patched to resolve any discrepancies with the canonical chain (which should ideally be the one described by the specification, but could be the e.g. op-node/op-geth chain if that is considered the reference implementation).

### Chain split driven by L1 chain split

- **Description:** Particularly for "L1 fork defense" upgrades (e.g. Pectra, Fusaka), in addition for L2 nodes being updated in time and being well configured, any L1 nodes which the L2 is connected to must _also_ be updated and well-configured. Failure to do so can result in different L2 nodes talking to divergent L1 nodes. Because L2 blocks always reference L1 blocks, this can cause a chain split on L2 in addition.
- **Risk Assessment:** medium severity / medium likelihood
- **Mitigations:**
- [ ] ACTION ITEM (BLOCKING): We have been very explicit in our notice documentation for node operators about the need to upgrade. 
- [ ] ACTION ITEM (BLOCKING): We have published a public runbook about what to do when this failure mode is detected.
- **Detection:** Replicas will diverge from reference endpoints, healthcheck alerts would fire.
- **Recovery Path(s)**: Shut down the batcher until the version and configuration discrepancies are fixed and the sequencer has performed any re-orgs. Then restart the batcher. 
