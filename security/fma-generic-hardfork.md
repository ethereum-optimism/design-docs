<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Generic items we need to take into account for any hardfork:](#generic-items-we-need-to-take-into-account-for-any-hardfork)
  - [Chain Halt at Activation](#chain-halt-at-activation)
  - [Incomplete Activation](#incomplete-activation)
  - [Hard fork Failure to Activate](#hard-fork-failure-to-activate)
  - [Invalid `DisputeGameFactory.setImplementation` execution](#invalid-disputegamefactorysetimplementation-execution)
- [Chain split (across clients)](#chain-split-across-clients)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### Generic items we need to take into account for any hardfork:

#### Chain Halt at Activation

- **Description:** Most hard forks have risk of bugs that cause chain halts at activation time, and this hard fork is no exception.

- **Risk Assessment:** Medium severity as no funds are at risk, though it is a full liveness failure that would be messy to recover from and require releasing and distributing an updated configuration and/or release. Likelihood is low, especially as we are using the mechanics for dynamically upgrading L2 contracts established in the Ecotone hardfork: namely, inserting appropriate deposit transactions in the activation block.

- **Mitigations:** We have implemented extensive unit and end-to-end testing of the activation flow.  We will be testing the activation as well on our devnets and testnets.

- **Detection:** Detection is straightforward as the chain will stop producing blocks.

- **Recovery Path(s)**: Would not require a vote or hardfork, but we’d likely have to coordinate a chain config update that pushed back the date of the upgrade, and allowed node operators to rollback any bad blocks.

    - We should also prepare datadir backups close before the upgrade, so we can use these in an emergency to rollback.

    - Estimated sequencer downtime is 30 min in a worst-case scenario where we have to reset the chain back to a block before the activation and disable the hardfork activation. Additional steps would be required from infra providers to get back to the healthy chain. They would either need to restart their op-node and op-geth with activation override command line flags/env var, or their images to an emergency release with activation disabled.

#### Incomplete Activation

- **Description:** The upgrade may take place but some steps may fail leaving the upgrade in a partial state. For example the contracts could get upgraded but changes to the consensus or execution layer were not applied, or the rest of the functionality upgrades but the contract deployments fail.

- **Risk Assessment:** Medium severity — funds should not be at risk, but chain could halt and recovery unfortunately could be very messy.

- **Mitigations:** End to end testing, and making sure the upgrade works properly and at the right time on our devnets and testnet. The end to end testing consists of all the tests in the `op-e2e` folder that were added as part of the implementation PRs.

- **Detection:** Requires we check: (1) updated contracts contain the new expected bytecode, (2) transactions submitted after the hardfork are behaving as expected and (3) invoking the new contracts gives the proper results.

- **Recovery Path(s)**: Recovery would likely involve a new node release and/or tools allowing for appropriate rollback depending on the precise nature of the failures.

    - There are two approaches, depending on failure

        - if it’s minor, we just live with it and fix it in an upcoming upgrade → no downtime

        - if it’s critical, we can initiate the same rollback as described above → same downtime as above

#### Hard fork Failure to Activate

- **Description:** The upgrade may be misconfigured and fail to take place.

- **Risk Assessment:** Low severity — the chain would continue on as if the upgrade never happened. We could recover easily by rescheduling the upgrade.

- **Mitigations:** End to end testing, and making sure the upgrade works properly and at the right time on our devnets and testnest. The end to end testing consists of all the tests in the `op-e2e` folder that were added as part of the implementation PRs.

- **Detection:** We can easily see if the contracts were upgraded as expected by reading the semantic versions. Node software startup logs should indicate whether the hardfork has activated properly.

- **Recovery Path(s)**: Reschedule upgrade, possibly releasing new binary though without immediate urgency.

#### Invalid `DisputeGameFactory.setImplementation` execution

- **Description:** This occurs when either the call to the `DisputeGameFactory` could not be made due to grossly unfavorable base fees on L1, an invalidly approved safe nonce, or a successful execution to a misconfigured dispute game implementation.

- **Risk Assessment:**
    - Low likelihood. The low likelihood is a result of tenderly simulation testing of safe transactions, code review of the upgrade playbook, and manual review of the dispute game implementations (which are deployed on mainnet and specified in the governance proposal so they may be reviewed).
    - High impact as Fault proofs remains incompatible with the new hardfork which would brick new bridge withdrawals. If we did nothing, i.e. ignored dispute-mon alerts and did not apply the FP recovery runbook, then an attacker would be able to steal the bonds of “honest” challengers that have upgraded their op-program by using the non-upgraded fork chain to generate a fault proof. In this scenario, the attacker would be unable to steal funds from the bridge so long as there exists an honest challenger that is configured to generate pre-upgrade execution traces. That said, the actual impact would be equivalent to [FP recovery](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21) as it would trigger the runbook, where we would have the opportunity to use overrides to recover the system.

- **Mitigations:** There are two cases:
    - If the safe transaction couldn’t be successfully executed, then: Revert `OptimismPortal` to use the permissioned fault dispute game as its respected game type. Depending on how much time till the upgrade, a superchain pause would be needed in order to gather necessary the signatures to change the `OptimismPortal`.
    - If the game implementation was misconfigured and this was detected prior to Fjord activation, then do the above. If detection occurs post-Fjord, then follow the [Fault Proofs Recovery Runbook](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21).

- **Detection:** An un-executed safe transaction is easily detectable. In the case of a misconfigured game implementation, the op-dispute-mon will alert proofs-squad and security on any attempt to exploit this misconfiguration.

- **Recovery Path(s)**: Reschedule upgrade, possibly releasing new binary though without immediate urgency.

### Chain split (across clients)

- **Description:** Differences in implementation across clients (e.g. `op-geth` versus `op-reth`) cause a chain split due to different effective consensus rules.
- **Risk Assessment:** medium severity / medium likelihood
- **Mitigations:** 
Multi-client testing
- **Detection:** Replicas of one kind of client will diverge from the sequencer
- **Recovery Path(s)**: Most likely we would have the op-node/op-geth chain be the canonical one as this is the reference implementation.Other clients would need to be patched to resolve any discrepancies.
