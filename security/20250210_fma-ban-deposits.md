# Ban `ExecutingMessage` Deposits: Failure Modes and Recovery Path Analysis

THIS DOCUMENT IS DEPRECATED AND HAS BEEN REPLACED BY [ANOTHER APPROACH](https://github.com/ethereum-optimism/design-docs/blob/f3aa2db64b1676b5e58ae602acf2ebdba34b617c/protocol/interop-access-list.md).

| Author | Skeletor, Parti, Joxes |
| --- | --- |
| Created at | 2025-01-10 |
| Initial Reviewers | Pending |
| Needs Approval From | Kelvin Fichter |
| Status | In Review |

## Introduction

This document is intended to be shared publicly for reviews and visibility purposes. It covers the changes introduced to ban calls to `validateMessage` in the `CrossL2Inbox` within a deposit. These changes involve contracts and client:

- **Contracts**:
    - Updates to the `CrossL2Inbox` to revert on deposit transactions.
    - Updates to the `L1BlockInterop` to add `isDeposit`, `depositsComplete`, and `setL1BlockValuesInterop` external functions.

- **Client**:
    - Updates to the `PreparePayloadAttributes` function within `derive/attributes.go`.
    - Adds `AfterForceIncludeSource` to `derive/deposit_source.go`.
    - Updates to the `derive/l1_block_info.go` to comply with the new contract changes specified above.

Below are references for this project:
- Both Contract and Client updates are documented in the following specs:
    - [interop: Specify deposit handling #258](https://github.com/ethereum-optimism/specs/pull/258).
- Reference implementation can be found below:
    - `CrossL2Inbox`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/CrossL2Inbox.sol
    - `L1BlockInterop`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/L1BlockInterop.sol

Note that this FMA doesn’t intend to cover the main features of core interop contracts for message passing. 

> 🗂️ **For more context about the Interop project, refer to the following docs:**
> 1. [Interop System Diagram](https://www.notion.so/Superchain-Interop-16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/Superchain-Interop-16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: An `ExecutingMessage` is emitted even if the `isDeposit` flag is correctly set, or `isDeposit` is not turned on before deposit transactions.

- **Description:** If a deposit transaction calls `validateMessage` without reverting and emitting the event `ExecutingMessage`, the sequencer could be forced to include messages that do not correspond to an existing identifier. This could break multiple interop invariants and eventually lead to a bricked chain.
- **Risk Assessment:** High.
    - Potential Impact: Critical. The impact of such an attack would vary depending on the message. Examples of economic attacks include:
        - Releasing all ETH from the `ETHLiquidity` contract.
        - Relaying an ERC20 with a custom amount never burned on the source chain.
        
        In the worst-case scenario, the chain could produce unsafe blocks that are subsequently rejected, leading to a reorg, which would effectively brick the chain afterwards. This occurs because the chain cannot progress with a deposit transaction containing an invalid execution. Services such as relay-based bridges could also face significant consequences if they fail to detect the situation promptly.
        
    - Likelihood: Low. This could only happen with a bugged implementation of the `CrossL2Inbox` check or a misconfiguration in `L1BlockInterop`. Even if the current implementation is bug-free, future upgrades can introduce these bugs.
- **Mitigations:** Our current codebase includes the following tests:
  - Test to check that the `CrossL2Inbox` contract validates messages only when `isDeposit` is `false` ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol#L139)).
  - Test to check that the `CrossL2Inbox` contract reverts messages only when `isDeposit` is `true` ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol#L166)).
  - Test to check that the `L1BlockInterop` contract enforce the same `isDeposit` logic ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L205)).
  - Test to check that the `L1BlockInterop` set `isDeposit` to `true` during the deposit context ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L239)).

    The security team should be aware of this issue and check for it in every protocol version upgrade. There should be a way to prevent to process such deposits. One solution could be update `op-geth` to revert automatically if it detects an execution of a deposit transactions that generates any logs in  the `CrossL2Inbox`.
- **Detection:** It is possible to monitor new incoming deposits and simulate them before to include in L2. This also would allow the sequencer to delay the deposit’s inclusion until the sequencing window ends, in order to have a delay a a time buffer to fix the issue. Otherwise, chain will face a halt after deposit is tried to be included, which is easily detected.
- **Recovery Path(s):** A chain halt followed by a fix and a reorg would be necessary.

### FM2: `upgradeTxs` include an invalid `ExecutingMessage`

- **Description:** The `isDeposit` bool is set off before the upgrade transactions, which are force-included. This implies that, if an upgrade transaction includes a call to `validateMessage`, the sequencer will be forced to include it, even if it doesn't point to an existing identifier.
- **Risk Assessment:** Medium.
    - Potential Impact: High. It could impact the same way described in FM1.
    - Likelihood: Low. An upgrade transaction should not call `validateMessage()` unless it is somehow intended to (and therefore not malicious). What's more, every upgrade transaction should bypass many security checks.
- **Mitigations:** Ideally, `upgradeTxs` should be enforced to be part of the deposit context, but since such upgrades have sudo privileges by nature—including the ability to turn off/remove the `isDeposit` flag—protocol changes would be required to actually enforce this rule. With the current implementation, during operations, every upgrade transaction should be simulated. An invalid Superchain state would be caught by the `op-supervisor` in simulations.
- **Detection:** Upgrades are heavily monitored transactions, making it very unlikely to go unnoticed.
- **Recovery Path(s):** Recovery would be similar to other bugged upgrades. See [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md) for more details.

### FM2: `isDeposit` is not turned off after deposit transactions

- **Description:** If the `depositComplete` call within `L1BlockInterop` fails or is never initiated, the `isDeposit` flag might remain on. This would imply that every call to `validateMessage` will be seen as a deposit and therefore revert.
- **Risk Assessment:** Low.
    - Potential Impact: Medium. Genuine cross-chain messages will not be able to execute. This should not be a major issue, as users can re-execute after the fix.
    - Likelihood: Low. This could happen if the `depositComplete` implementation is bugged or the sequencer is not triggering the call to the function. The latter could occur due to a client bug or an out-of-gas error, which is unlikely.
- **Mitigations:** Our current codebase includes tests to check `L1BlockInterop` set `isDeposit` as `false` after the deposit context ends ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L292)). The security team should know this issue and check for it in every protocol version.
- **Detection:** Offchain services should be aware of this possibility for `validateMessage` reverts.
- **Recovery Path(s):** Execute the proper fixes depending on whether it was a sequencer or contract error. Valid reverted messages can be re executed on destination, or resent if expired.

### Generic items we need to take into account:

- Every consideration already covered by the [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md) will also apply to these changes.
- It is important to have a good gas benchmark for `L1Block.depositsComplete()` in the client to minimize out of gas errors.

## Action Items

- [ ]  Resolve all the comments.
- [ ]  FM1: Confirm whether the suggested changes to `op-geth` are considered (and implemented) or if other options are chosen.
- [ ]  FM1: Confirm whether deposit simulations and procedures are feasible or if alternative monitoring methods are being considered (as action items).
- [ ]  FM2: Confirm that the security team is aware of the lack of restrictions over upgradeTxs and closely monitor any future upgrade to ensure it does not call `validateMessage`. Ideally, we would like to have strong restrictions for `upgradeTxs`.

## Audit Requirements

We suggest the modifications for banning deposit triggering `ExecutingMessage` events go through an audit, as they affect a sensitive part of the protocol. Following on the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), some presented failure modes are similar to the ”Deposit path no spoofing”, as it affects the validity of the deposits. In particular, referencing a non-existing cross-chain message with a deposit can be considered spoofing.

## Additional Notes

Something worth noticing is that implementing the ban deposit closes the door for intended `ExecutingMessage` emissions from deposits. This feature could be desirable to force valid cross-chain messages from L1 and bypass sequencer censorship. There is an active exploration to ensure that the censorship-resistance property, with the introduction of [preregistrations](https://github.com/ethereum-optimism/specs/issues/520).
