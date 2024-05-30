# Introduction

This document is intended to be shared in a public space for reviews and visibility purposes. It covers the Fjord superchain upgrade, on a mainnet level, which involves four changes:

- enables a precompile for the secp256r1 elliptic curve ([RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md))
- a more accurate L1 data cost function that estimates the batch-compressed transaction size using a [FastLZ](https://code.google.com/archive/p/fastlz/) based linear regression.
- Introduces Brotli as new channel compression algo
- Increase max-sequencer-drift to 30 minutes
- 10x the max rlp bytes per channel and max channel bank size

Below are references for this project:

- The Fjord upgrade will be fully documented in the monorepo [specifications](https://github.com/ethereum-optimism/specs) once the implementation is finalized.
- The unfinalized fork specification describing the EIP-7212 changes are [here](https://github.com/ethereum-optimism/specs/blob/main/specs/fjord/overview.md).
- FastLZ related spec changes can be found [here](https://github.com/ethereum-optimism/specs/pull/25).
- [PR](https://github.com/ethereum-optimism/specs/pull/146) for max-sequencer-drift spec update.
- Brotli channel compression
    - Specs: https://github.com/ethereum-optimism/specs/pull/168
    - Impl: https://github.com/ethereum-optimism/optimism/pull/10358

# **Failure Modes and Recovery Paths**

## **Bugs introduced in the L2 contract upgrades**

Fjord introduces changes to the `GasPriceOracle` precompile contract on the L2. As with any contract change, there is the potential for new bugs to be introduced. The contract changes introduced for Fjord are contained primarily in this PR: https://github.com/ethereum-optimism/optimism/pull/9618 (main PR with upgrade txs) and the GPO change in its separate PR https://github.com/ethereum-optimism/optimism/pull/10534

- `GasPriceOracle.sol`:
    - Adds one new member variable, `isFjord`, a boolean value, and a setter `setFjord()` to flip it to true. The setter will revert unless the sender is the `DEPOSITOR_ACCOUNT` from the L1Block contract.
    - Updates `getL1Fee()` such that if `isFjord` is true, then the updated Fjord cost function is invoked (`_getL1FeeFjord`) instead of previous ones.
    - adds a convenience function `getL1FeeUpperBound` which is not consensus-critical. Its purpose is to allow applications to get an upper bound on the l1 data fee for any transaction with very low gas cost, as it does not require the entire transaction to be provided as calldata.
- **Risk Assessment:** Low. These changes are similar but far more limited than the related changes made in the previous Ecotone hardfork. In this upgrade, while the function itself is changed, the data required to compute the function is the same as before. Ecotone additionally changed the L1 block attributes, and there are no such changes in Fjord. If there are bugs in the new cost function implementation, then external wallets may inaccurately report expected fee values to users. The Solidity implementation however is not actually used by the system itself for determining fees during consensus; the Solidity cost function is mostly a convenience function for external tooling.
- **Mitigations:** The new functions are unit and fuzz tested within Solidity. Additionally the solidity implementation of FastLZLen is differentially tested against the Go implementation and a reference C implementation. Tests can be found [here](https://github.com/ethereum-optimism/optimism/pull/9618/files#diff-92afca4468688532309fa0b00ba9c48e03158c272aab774307d088372cf11e63R74), they consist of 3 fuzz tests.
- **Detection:** Most chains have monitoring in place for tracking L1 data fees and recovery, and would expect alerts to fire if say we are significantly underpricing. Detecting overpricing might require manual inspection of transaction fees after the hard fork.
- **Recovery Path(s)**:  Contracts are upgradable, and if there are any bugs, an emergency deployment of a fixed contract could be scheduled. This will require a multi-sig ceremony for each chain and could take some time.

## **Incorrect L1 Data Fee computation**

- **Description:** Fjord updates the cost function for computing the L1 data fee of each transaction. If there are bugs in this implementation it could result in inappropriate L1 data fees being charged, either too high (costing users significant fees) or too low (causing revenue loss for chain operators).  Implementation of this change is primarily in [this PR](https://github.com/ethereum-optimism/op-geth/pull/249).
- **Risk Assessment:** Low. The new cost function introduces no new inputs, but instead implements a more sophisticated handling and transformation of existing inputs, in particular the bytes in the transaction data. Instead of naively counting zero and non-zero bytes to estimate compressibility, the Fjord function uses an actual but high-performance compression algorithm, FastLZLen, to estimate compressibility of the transaction. The output of this function is fed into a linear regression, with hard-coded coefficients tuned to minimize estimation error across various slices of chain data (details in [this protocol quest issue](https://github.com/ethereum-optimism/protocol-quest/issues/210) and the [compression analysis repo](https://github.com/roberto-bayardo/compression-analysis)).
- **Mitigations:** Unit and end-to-end testing, extensive evaluation of the cost function showing far better robustness in minimizing estimation error compared to what has been used pre-Fjord.
- **Detection:** Dashboards would quickly reveal if we are charging too little and alerts should fire if we are no longer recovering revenue from L1 data posting. L1 fee parameters are tuned to maintain a 5% margin above expected cost. If we are charging too much then we’ll observe revenue recovery well above this margin, and could even here reports from users/dApp operators. Chains such as Base have alerts set to fire If we are charging too little and revenue recovery goes below 0.
- **Recovery Path(s)**: Fjord preserves the ability to adjust fee recovery margin via fee scalar and blob fee scalars should we be under or over charging

## Failures with RIP-7212 precompile

- **Description:** The 7212 precompile may not get activated properly at the point of the hardfork, or it may be activated properly but there may be bugs in the implementation.
- **Risk Assessment:** Low.  If 7212 fails to deploy, since there are no existing uses of it, there will be no immediate impact. The impact is in prohibiting future potential uses of this predeploy. If the issue is a bug in the implementation, then users could end up trying to use it anyway and get incorrect results.  The implementation however is the reference implementation that has been been vetted by the broader community, so this outcome is unlikely.
- **Mitigations:** Precompiles activate new code paths and do not require dynamic deployment of new contract code, which implies the risk is lower than a predeploy update.  Mitigations mostly involve e2e testing, and vetting activation in our dev & testnets.  The 7212 implementation itself that we are using has been extensively tested and vetted not only by us but also the broader community.
- **Detection:** A simple transaction that attempts to invoke the precompile will quickly reveal whether it was deployed successfully. If so, we can then re-run the test-suites against it to make sure it still is working as intended.
- **Recovery Path(s)**: Because of the low impact, if the precompile fails to activate, the appropriate recovery path would be to attempt to redeploy it in a subsequent hard-fork. If the deployment is buggy, we could discourage its use until we update it in a subsequent hardfork. We could even block transactions that invoke it at the sequencer level if we want to be extra cautious. (Users could still invoke it via forced inclusion, but this would require they first understand why it’s being blocked in the first place.)

## Brotli Channel Compression

- **Description:** Channel compression currently uses the zlib algorithm, which is fast and works well for small data compression. With the advent of multi-blob batches, we found that Brotli-10 offers significant improvements in compression ratios (10-20% in some cases), motivating its inclusion in the Fjord upgrade. One risk is the brotli compression doesn’t activate to become a valid compression algo and we continue to use zlib after the hardfork.
    - Implementation: https://github.com/ethereum-optimism/optimism/pull/10358
- **Risk Assessment:** Low; batch format itself isn’t changing, other than a new version type indicating which compression algorithm was used to create the transaction. Upon activation, chain derivation will be able to interpret the new versions and apply brotli decompression instead of zlib in response.
- **Mitigations:** We extended e2e tests to exercise all variations of compression (zlib / brotli-10) and batch type (span-batch vs. singular), and will confirm brotli-compression activates safely on testnets.  We will also allow zlib-versioned batches to continue to be handled appropriately during chain derivation even after the Fjord hardfork.
- **Detection:** Inspecting batches posted after Fjord will reveal if the new version indicator is being set correctly.
- **Recovery Path(s)**: Because we’ll continue to allow chain derivation to interpret zlib-compressed batches, an upgraded release (not involving a hardfork) could revert to previous zlib-compression behavior if there are any issues detected with using Brotli.

## Max Sequencer Drift update

- **Description:** Max sequencer drift change might introduce unexpected consequences. It may either fail to activate, or use a wrong value. Implementation of this change can be found in [this PR](https://github.com/ethereum-optimism/optimism/pull/10465); it involves simply hardcoding max-sequencer-drift to the desired value and using that value after the upgrade instead of the previous chain-configured parameter. It is raised from the default parameter value of 600 seconds = 10 min to a constant value of 1800 seconds = 30 min.
    - If it fails to activate, nothing happens and the old chain parameter (10 min for almost all chains) will continue to be used.
    - If a wrong unintended value is used post-activation, it would have the impact that L2 blocks would accept or reject blocks that shouldn’t be.
- **Risk Assessment:** Very Low, as the change simply amounts getting the value from a hard-coded constant instead of the previous parameter.
- **Mitigations:** Activation of the new max sequencer drift is by L1 origin timestamp, so L2 blocks of the same epoch will consistently have the same max sequencer drift around the activation time. Activation by L1 origin is straight forward, as simply the first set of L2 blocks to use an L1 origin with timestamp ≥ the Fjord timestamp will be subject to the new max sequencer drift rules. New tests were introduced to test the correct activation of this change in the sequencer’s block creation code as well as the derivation pipeline.

- **Detection:** This wouldn’t be obviously detectable unless we experienced an L1 outage and blocks would get rejected before the intended 30 minute sequencer drift.  We could force this condition on our testnets to confirm it’s working as intended.
- **Recovery Path(s)**: If the failure is a simple failure to activate, it’s simple enough to perform the change again in the next hardfork. If the failure is a wrong value, then the recovery depends on the longest actual sequencer drift since Fjord activation.
    - If there weren’t any L1 outages, thus having maintained a low sequencer drift, then on such chains this could be fixed by a hotfix release.
    - If there was an actual L1 outage that lead to unintended rejected or accepted blocks, then the current faulty max sequencer drift rule could probably still be maintained for a while and then fixed in a quick follow-up fork.

## 10x of channel size constants

- **Description:** Two constants related to reading channels from L1 in the derivation pipeline were increased by a factor of 10x, the `MAX_RLP_BYTES_PER_CHANNEL` and `MAX_CHANNEL_BANK_SIZE`. This change is important for chains that run with higher gas limit configurations that would allow single blocks to become larger than the current limit of 10MB for the max rlp size. Specifically if the gas limit is greater than 40 Million, transactions with all zeros in the calldata could create a 10MB block which would exceed the existing limit.
    
    The only conceivable failure mode is that this change doesn’t properly activate. This would be inconsequential to Ethereum chains, and only affect Alt-DA chains.
    
- **Risk Assessment:** Very low likelihood of failure and impact. The implementation simply changes to the new constants on the Fjord activation timestamp. The impact to Ethereum chains would be none, as they don’t need a larger max rlp or channel bank size. The impact for Alt-DA chains would be that in theory there could be blocks that don’t fit into a single channel, which would invalidate such blocks and halt the chain.
- **Mitigations:** A new `ChainSpec` was added to encapsulate the constants changes for this upgrade and future changes of constants. Extensive unit tests of this new abstraction were added.
https://github.com/ethereum-optimism/optimism/pull/10357
- **Detection:** It wouldn’t be detectable on Ethereum chains. On Alt-DA chains with high gas limits, with a failed activation, it could happen that a block full of large zero-data txs is produced, which doesn’t fit into a channel, and the chain would halt.
- **Recovery Path(s)**: We’d just fix it in a next protocol upgrade. Alt-DA chains wouldn’t be able to include such blocks. A hot-fix could also be implemented on Alt-DA chain sequencers to proactively check the block size while building the block and stop adding txs to the block if its rlp size would go over the max rlp channel size limit. This is possible because *individual* txs cannot be larger than 128KB due to geth gossip limits.

## Brotli/FastLZ size mismatch exploit

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: *How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?*

## Generic items we need to take into account

### Chain Halt at Activation

- **Description:** Most hard forks have risk of bugs that cause chain halts at activation time, and this hard fork is no exception.
- **Risk Assessment:** Medium severity as no funds are at risk, though it is a full liveness failure that would be messy to recover from and require releasing and distributing an updated configuration and/or release. Likelihood is low, especially as we are using the same mechanics for dynamically upgrading L2 contracts from Ecotone, which involves inserting appropriate deposit transactions in the activation block.
- **Mitigations:** We have implemented extensive unit and end-to-end testing of the Fjord activation flow.  We will be testing the activation as well on our devnets and testnets.
- **Detection:** Detection is straightforward as the chain will stop producing blocks.
- **Recovery Path(s)**: Would not require a vote or hardfork, but we’d likely have to coordinate a chain config update that pushed back the date of the upgrade, and allowed node operators to rollback any bad blocks.
    - We should also prepare datadir backups close before the upgrade, so we can use these in an emergency to rollback.
    - Estimated sequencer downtime is 30 min in a worst-case scenario where we have to reset the chain back to a block before the activation and disable the hardfork activation. Additional steps would be required from infra providers to get back to the healthy chain. They would either need to restart their op-node and op-geth with activation override command line flags/env var, or their images to an emergency release with activation disabled.
    - It should be noted that we already prepared for this generic scenario for Ecotone, so we can use the same preparations, which we already practiced.
