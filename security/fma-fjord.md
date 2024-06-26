<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Fjord Hardfork FMA (Failure Modes and Recovery Path Analysis)](#fjord-hardfork-fma-failure-modes-and-recovery-path-analysis)
  - [Introduction](#introduction)
  - [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
    - [Bugs introduced in the L2 contract upgrades](#bugs-introduced-in-the-l2-contract-upgrades)
    - [Incorrect L1 Data Fee computation](#incorrect-l1-data-fee-computation)
    - [Failures with RIP-7212 precompile](#failures-with-rip-7212-precompile)
    - [Brotli Channel Compression](#brotli-channel-compression)
    - [Max Sequencer Drift update](#max-sequencer-drift-update)
    - [10x of channel size constants](#10x-of-channel-size-constants)
    - [Brotli/FastLZ size mismatch exploit](#brotlifastlz-size-mismatch-exploit)
    - [P256VERIFY is too expensive for Fault Proofs](#p256verify-is-too-expensive-for-fault-proofs)
    - [Generic items we need to take into account:](#generic-items-we-need-to-take-into-account)
      - [Chain Halt at Activation](#chain-halt-at-activation)
      - [Incomplete Activation](#incomplete-activation)
      - [Hard fork Failure to Activate](#hard-fork-failure-to-activate)
      - [Invalid `DisputeGameFactory.setImplementation` execution](#invalid-disputegamefactorysetimplementation-execution)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Fjord Hardfork FMA (Failure Modes and Recovery Path Analysis)

| | |
|--------|--------------|
| Author | Roberto Bayardo, Sebastian Stammler |
| Created at | 2024-04-19 |
| Needs Approval From | Matt Solomon |
| Other Reviewers | Sebastian Stammler, Dragan Zurzin |
| Status | Final |

## Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers the Fjord superchain upgrade, on a mainnet level, which involves four changes:

- enables a precompile for the secp256r1 elliptic curve ([RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md))
- a more accurate L1 data cost function that estimates the batch-compressed transaction size using a [FastLZ](https://code.google.com/archive/p/fastlz/) based linear regression.
- Introduces Brotli as new channel compression algo
- Increase max-sequencer-drift to 30 minutes
- 10x the max rlp bytes per channel and max channel bank size
    - Impl: https://github.com/ethereum-optimism/optimism/pull/10357

Below are references for this project:

- The Fjord upgrade will be fully documented in the monorepo [specifications](https://github.com/ethereum-optimism/specs) once finalized.
- The fork specification describing the RIP-7212 changes are [here](https://github.com/ethereum-optimism/specs/blob/main/specs/fjord/overview.md).
- FastLZ related spec changes can be found [here](https://github.com/ethereum-optimism/specs/pull/25).
- [Impl](https://github.com/ethereum-optimism/specs/pull/146) for max-sequencer-drift spec update.
- Brotli channel compression
    - Specs: https://github.com/ethereum-optimism/specs/pull/168
    - Impl: https://github.com/ethereum-optimism/optimism/pull/10358
- Check script confirming correct activation of EL changes: https://github.com/ethereum-optimism/optimism/pull/10578

## Failure Modes and Recovery Paths

### Bugs introduced in the L2 contract upgrades

Fjord introduces changes to the `GasPriceOracle` precompile contract on the L2. As with any contract change, there is the potential for new bugs to be introduced. The contract changes introduced for Fjord are contained primarily in this PR: https://github.com/ethereum-optimism/optimism/pull/9618 (main PR with upgrade txs) and the GPO change in its separate PR https://github.com/ethereum-optimism/optimism/pull/10534

- `GasPriceOracle.sol`:

    - Adds one new member variable, `isFjord`, a boolean value, and a setter `setFjord()` to flip it to true. The setter will revert unless the sender is the `DEPOSITOR_ACCOUNT` from the L1Block contract.

    - Updates `getL1Fee()` such that if `isFjord` is true, then the updated Fjord cost function is invoked (`_getL1FeeFjord`) instead of previous ones.

    - adds a convenience function `getL1FeeUpperBound` which is not consensus-critical. Its purpose is to allow applications to get an upper bound on the l1 data fee for any transaction with very low gas cost, as it does not require the entire transaction to be provided as calldata.

- **Risk Assessment:** Low. These changes are similar but far more limited than the related changes made in the previous Ecotone hardfork. In this upgrade, while the function itself is changed, the data required to compute the function is the same as before. Ecotone additionally changed the L1 block attributes, and there are no such changes in Fjord. If there are bugs in the new cost function implementation, then external wallets may inaccurately report expected fee values to users. The Solidity implementation however is not actually used by the system itself for determining fees during consensus; the Solidity cost function is mostly a convenience function for external tooling.

- **Mitigations:** The new functions are unit and fuzz tested within Solidity. Additionally the solidity implementation of FastLZLen is differentially tested against the Go implementation and a reference C implementation. Tests can be found [here](https://github.com/ethereum-optimism/optimism/pull/9618/files#diff-92afca4468688532309fa0b00ba9c48e03158c272aab774307d088372cf11e63R74), they consist of 3 fuzz tests.

- **Detection:** Most chains have monitoring in place for tracking L1 data fees and recovery, and would expect alerts to fire if say we are significantly underpricing.  For example, the our implementation team has set up DataDog alerts on various metrics including batcher/proposer funding levels, which are replenished from sequencer fee revenue. Detecting overpricing might require manual inspection of transaction fees after the hard fork.

- **Recovery Path(s)**:  Contracts are upgradable, and if there are any bugs, an emergency deployment of a fixed contract could be scheduled. This will require a multi-sig ceremony for each chain and could take some time.


### Incorrect L1 Data Fee computation


- **Description:** Fjord updates the cost function for computing the L1 data fee of each transaction. If there are bugs in this implementation it could result in inappropriate L1 data fees being charged, either too high (costing users significant fees) or too low (causing revenue loss for chain operators).  Implementation of this change is primarily in [this PR](https://github.com/ethereum-optimism/op-geth/pull/249).

- **Risk Assessment:** Low. The new cost function introduces no new inputs, but instead implements a more sophisticated handling and transformation of existing inputs, in particular the bytes in the transaction data. Instead of naively counting zero and non-zero bytes to estimate compressibility, the Fjord function uses an actual but high-performance compression algorithm, FastLZLen, to estimate compressibility of the transaction. The output of this function is fed into a linear regression, with hard-coded coefficients tuned to minimize estimation error across various slices of chain data (details can be found in this [compression analysis repo](https://github.com/roberto-bayardo/compression-analysis)).

- **Mitigations:** Unit and end-to-end testing, extensive evaluation of the cost function showing far better robustness in minimizing estimation error compared to what has been used pre-Fjord.

- **Detection:** Dashboards would quickly reveal if we are charging too little and alerts should fire if we are no longer recovering revenue from L1 data posting. L1 fee parameters are tuned to maintain a 5% margin above expected cost. If we are charging too much then we’ll observe revenue recovery well above this margin, and could even here reports from users/dApp operators. Chains such as Base have alerts set to fire If we are charging too little and revenue recovery goes below 0.

- **Recovery Path(s)**: Fjord preserves the ability to adjust fee recovery margin via fee scalar and blob fee scalars should we be under or over charging

### Failures with RIP-7212 precompile

- **Description:** The 7212 precompile may not get activated properly at the point of the hardfork, or it may be activated properly but there may be bugs in the implementation.

- **Risk Assessment:** Low.  If 7212 fails to deploy, since there are no existing uses of it, there will be no immediate impact. The impact is in prohibiting future potential uses of this predeploy. If the issue is a bug in the implementation, then users could end up trying to use it anyway and get incorrect results.  The implementation however is the reference implementation that has been been vetted by the broader community, so this outcome is unlikely.

- **Mitigations:** Precompiles activate new code paths and do not require dynamic deployment of new contract code, which implies the risk is lower than a predeploy update.  Mitigations mostly involve e2e testing, and vetting activation in our dev & testnets.  The 7212 implementation itself that we are using has been extensively tested and vetted not only by us but also the broader community.

- **Detection:** A simple transaction that attempts to invoke the precompile will quickly reveal whether it was deployed successfully. If so, we can then re-run the test-suites against it to make sure it still is working as intended.

- **Recovery Path(s)**: Because of the low impact, if the precompile fails to activate, the appropriate recovery path would be to attempt to redeploy it in a subsequent hard-fork. If the deployment is buggy, we could discourage its use until we update it in a subsequent hardfork. We could even block transactions that invoke it at the sequencer level if we want to be extra cautious. (Users could still invoke it via forced inclusion, but this would require they first understand why it’s being blocked in the first place.)

### Brotli Channel Compression

- **Description:** Channel compression currently uses the zlib algorithm, which is fast and works well for small data compression. With the advent of multi-blob batches, we found that Brotli-10 offers significant improvements in compression ratios (10-20% in some cases), motivating its inclusion in the Fjord upgrade. One risk is the brotli compression doesn’t activate to become a valid compression algo and we continue to use zlib after the hardfork.

    - Implementation: https://github.com/ethereum-optimism/optimism/pull/10358

    - Additional E2E test: https://github.com/ethereum-optimism/optimism/pull/10531

- **Risk Assessment:** Low; batch format itself isn’t changing, other than a new version type indicating which compression algorithm was used to create the transaction. Upon activation, chain derivation will be able to interpret the new versions and apply brotli decompression instead of zlib in response.

- **Mitigations:** We extended e2e tests to exercise all variations of compression (zlib / brotli-10) and batch type (span-batch vs. singular), and will confirm brotli-compression activates safely on testnets.  We will also allow zlib-versioned batches to continue to be handled appropriately during chain derivation even after the Fjord hardfork.

- **Detection:** Inspecting batches posted after Fjord will reveal if the new version indicator is being set correctly.

- **Recovery Path(s)**: Because we’ll continue to allow chain derivation to interpret zlib-compressed batches, an upgraded release (not involving a hardfork) could revert to previous zlib-compression behavior if there are any issues detected with using Brotli.

### Max Sequencer Drift update

- **Description:** Max sequencer drift change might introduce unexpected consequences. It may either fail to activate, or use a wrong value. Implementation of this change can be found in [this PR](https://github.com/ethereum-optimism/optimism/pull/10465); it involves simply hardcoding max-sequencer-drift to the desired value and using that value after the upgrade instead of the previous chain-configured parameter. It is raised from the default parameter value of 600 seconds = 10 min to a constant value of 1800 seconds = 30 min.

    - If it fails to activate, nothing happens and the old chain parameter (10 min for almost all chains) will continue to be used.

    - If a wrong unintended value is used post-activation, it would have the impact that L2 blocks would accept or reject blocks that shouldn’t be.

- **Risk Assessment:** Very Low, as the change simply amounts getting the value from a hard-coded constant instead of the previous parameter.

- **Mitigations:** Activation of the new max sequencer drift is by L1 origin timestamp, so L2 blocks of the same epoch will consistently have the same max sequencer drift around the activation time. Activation by L1 origin is straight forward, as simply the first set of L2 blocks to use an L1 origin with timestamp ≥ the Fjord timestamp will be subject to the new max sequencer drift rules. New tests were introduced to test the correct activation of this change in the sequencer’s block creation code as well as the derivation pipeline.

- **Detection:** This wouldn’t be obviously detectable unless we experienced an L1 outage and blocks would get rejected before the intended 30 minute sequencer drift.  We could force this condition on our testnets to confirm it’s working as intended.

- **Recovery Path(s)**: If the failure is a simple failure to activate, it’s simple enough to perform the change again in the next hardfork. If the failure is a wrong value, then the recovery depends on the longest actual sequencer drift since Fjord activation.

    - If there weren’t any L1 outages, thus having maintained a low sequencer drift, then on such chains this could be fixed by a hotfix release.

    - If there was an actual L1 outage that lead to unintended rejected or accepted blocks, then the current faulty max sequencer drift rule could probably still be maintained for a while and then fixed in a quick follow-up fork.

### 10x of channel size constants

- **Description:** Two constants related to reading channels from L1 in the derivation pipeline were increased by a factor of 10x, the `MAX_RLP_BYTES_PER_CHANNEL` and `MAX_CHANNEL_BANK_SIZE`. This change is important for chains that run with higher gas limit configurations that would allow single blocks to become larger than the current limit of 10MB for the max rlp size. Specifically if the gas limit is greater than 40 Million, transactions with all zeros in the calldata could create a 10MB block which would exceed the existing limit.
    The only conceivable failure mode is that this change doesn’t properly activate. This would be inconsequential to Ethereum chains, and only affect Alt-DA chains.

- **Risk Assessment:** Very low likelihood of failure and impact. The implementation simply changes to the new constants on the Fjord activation timestamp. The impact to Ethereum chains would be none, as they don’t need a larger max rlp or channel bank size. The impact for Alt-DA chains would be that in theory there could be blocks that don’t fit into a single channel, which would invalidate such blocks and halt the chain.

- **Mitigations:** A new `ChainSpec` was added to encapsulate the constants changes for this upgrade and future changes of constants. Extensive unit tests of this new abstraction were added.

For more detail, feel free to review: https://github.com/ethereum-optimism/optimism/pull/10357

- **Detection:** It wouldn’t be detectable on Ethereum chains. On Alt-DA chains with high gas limits, with a failed activation, it could happen that a block full of large zero-data txs is produced, which doesn’t fit into a channel, and the chain would halt.

- **Recovery Path(s)**: We’d just fix it in a next protocol upgrade. Alt-DA chains wouldn’t be able to include such blocks. A hot-fix could also be implemented on Alt-DA chain sequencers to proactively check the block size while building the block and stop adding txs to the block if its rlp size would go over the max rlp channel size limit. This is possible because *individual* txs cannot be larger than 128KB due to geth gossip limits.

### Brotli/FastLZ size mismatch exploit

- **Description:** If it is possible to create transactions that have a (much) smaller FastLZ size than their actually compressed Brotli size, these transactions would be underpriced by our new cost function.

- **Risk Assessment:** Low. Brotli is a much better compression algo than FastLZ, so if at all, this should only be possible for certain crafted edge cases, which then probably don’t have any utility, so it would still cost an attacker fees to send nonsense transactions.

- **Mitigations:** We did extensive historical backtesting of FastLZ vs zlib and also Brotli compressed transactions and could confirm that the vast majority of transactions are over- and not underpriced by FastLZ (but still much more fairly priced than the old method of counting (non-)zero bytes).

- **Detection:** We’d detect this by doing historical analysis of FastLZ vs actual compressed batch size usage. We have already planned to do this kind of analysis after activating Fjord on mainnet.

- **Recovery Path(s)**: If this turned out to be a major problem (very unlikely), we can always revert back to zlib compression, which should be closer to FastLZ and then not exhibit this attack surface. If it only affects a small number of transactions, we can ship an improvement to the cost function in the next hardfork and live with a few underpriced transactions in the meantime.

### P256VERIFY is too expensive for Fault Proofs

- **Description:** The introduction of new precompiles always carries a risk of making it too costly for the op-challenger to generate execution traces and fault proofs that contain the precompile.

- **Risk Assessment:** Low. [Based on Cannon performance testing of various precompiles](https://www.notion.so/865e2b86c24a404dbf6f2aa35b94a0eb?pvs=21), including `P256VERIFY`, it’s not costly for offchain Cannon to generate fault proofs. The gas schedule correctly reflects the offchain costs of `P256VERIFY` traces*.* There aren’t known adversarial inputs to `P256VERIFY` that would induce an unexpectedly long Cannon execution trace.

- **Mitigations:** [Cannon precompile performance testing](https://www.notion.so/865e2b86c24a404dbf6f2aa35b94a0eb?pvs=21) show that it’s relatively, compared to existing precompiles, inexpensive to generate traces of `P256VERIFY` operations.

- **Detection:** The op-challenger will be slower in responding to games and may ultimately lose dispute games. op-dispute-mon contains metrics that are configured to page proofs-squad and security on either case.

- **Recovery Path(s)**:

### Generic items we need to take into account:

#### Chain Halt at Activation

- **Description:** Most hard forks have risk of bugs that cause chain halts at activation time, and this hard fork is no exception.

- **Risk Assessment:** Medium severity as no funds are at risk, though it is a full liveness failure that would be messy to recover from and require releasing and distributing an updated configuration and/or release. Likelihood is low, especially as we are using the same mechanics for dynamically upgrading L2 contracts from Ecotone, which involves inserting appropriate deposit transactions in the activation block.

- **Mitigations:** We have implemented extensive unit and end-to-end testing of the Fjord activation flow.  We will be testing the activation as well on our devnets and testnets.

- **Detection:** Detection is straightforward as the chain will stop producing blocks.

- **Recovery Path(s)**: Would not require a vote or hardfork, but we’d likely have to coordinate a chain config update that pushed back the date of the upgrade, and allowed node operators to rollback any bad blocks.

    - We should also prepare datadir backups close before the upgrade, so we can use these in an emergency to rollback.

    - Estimated sequencer downtime is 30 min in a worst-case scenario where we have to reset the chain back to a block before the activation and disable the hardfork activation. Additional steps would be required from infra providers to get back to the healthy chain. They would either need to restart their op-node and op-geth with activation override command line flags/env var, or their images to an emergency release with activation disabled.

    - It should be noted that we already prepared for this generic scenario for Ecotone, so we can use the same preparations, which we already practiced.

#### Incomplete Activation

- **Description:** The upgrade may take place but some steps may fail leaving the upgrade in a partial state. For example the contracts could get upgraded but the new cost function might not be applied, or the rest of the functionality upgrades but the contract deployments fail.

- **Risk Assessment:** Medium severity — funds should not be at risk, but chain could halt and recovery unfortunately could be very messy.

- **Mitigations:** End to end testing, and making sure the upgrade works properly and at the right time on our devnets and testnet. The end to end testing consists of all the tests in the `op-e2e` folder that were added as part of the implementation PRs.

- **Detection:** Requires we check: (1) updated L2 contracts contain the new expected bytecode, (2) transactions submitted after the hardfork are being charged data fees as expected according to the new function, (3) invoking the new precompile produces the expected output.

- **Recovery Path(s)**: Recovery would likely involve a new node release and/or tools allowing for appropriate rollback depending on the precise nature of the failures.

    - There are two approaches, depending on failure

        - if it’s minor, we just live with it and fix it in an upcoming upgrade → no downtime

        - if it’s critical, we can initiate the same rollback as described above → same downtime as above

#### Hard fork Failure to Activate

- **Description:** The upgrade may be misconfigured and fail to take place.

- **Risk Assessment:** Low severity — the chain would continue on as if the upgrade never happened. We could recover easily by rescheduling the upgrade.

- **Mitigations:** End to end testing, and making sure the upgrade works properly and at the right time on our devnets and testnest. The end to end testing consists of all the tests in the `op-e2e` folder that were added as part of the implementation PRs.

- **Detection:** We can easily see if the L2 contracts were upgraded as expected by trying to invoke their new read methods such as reading the blob base fee parameter from the gas price oracle.

- **Recovery Path(s)**: Reschedule upgrade, possibly releasing new binary though without immediate urgency.

#### Invalid `DisputeGameFactory.setImplementation` execution

- **Description:** This occurs when either the call to the `DisputeGameFactory` could not be made due to grossly unfavorable base fees on L1, an invalidly approved safe nonce, or a successful execution to a misconfigured dispute game implementation.

- **Risk Assessment:**
    - Low likelihood. The low likelihood is a result of tenderly simulation testing of safe transactions, code review of the upgrade playbook, and manual review of the dispute game implementations (which are deployed on mainnet and specified in the governance proposal so they may be reviewed).
    - High impact as Fault proofs remains incompatible with Fjord which would brick new bridge withdrawals. If we did nothing, i.e. ignored dispute-mon alerts and did not apply the FP recovery runbook, then an attacker would be able to steal the bonds of “honest” challengers that have upgraded their op-program to Fjord by using the ecotone fork chain to generate a fault proof. In this scenario, the attacker would be unable to steal funds from the bridge so long as there exists an honest challenger that is configured to generate ecotone execution traces. That said, the actual impact would be equivalent to [FP recovery](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21) as it would trigger the runbook, where we would have the opportunity to use overrides to recover the system.

- **Mitigations:** There are two cases:
    - If the safe transaction couldn’t be successfully executed, then: Revert `OptimismPortal` to use the permissioned fault dispute game as its respected game type. Depending on how much time till the upgrade, a superchain pause would be needed in order to gather necessary the signatures to change the `OptimismPortal`.
    - If the game implementation was misconfigured and this was detected prior to Fjord activation, then do the above. If detection occurs post-Fjord, then follow the [Fault Proofs Recovery Runbook](https://www.notion.so/8dad0f1e6d4644c281b0e946c89f345f?pvs=21).

- **Detection:** An un-executed safe transaction is easily detectable. In the case of a misconfigured game implementation, the op-dispute-mon will alert proofs-squad and security on any attempt to exploit this misconfiguration.

- **Recovery Path(s)**: Reschedule upgrade, possibly releasing new binary though without immediate urgency.
