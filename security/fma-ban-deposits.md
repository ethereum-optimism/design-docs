# Ban `ExecutingMessage` Deposits: Failure Modes and Recovery Path Analysis

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
- Reference implementation and tests can be found in the following PR:
    - `CrossL2Inbox`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/CrossL2Inbox.sol
    - `L1BlockInterop`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/L1BlockInterop.sol

Note that this FMA doesn’t intend to cover the main features of core interop contracts for message passing. 

> 🗂️ **For more context about the Interop project, refer to the following docs:**
> 1. [Interop System Diagram](https://www.notion.so/Superchain-Interop-16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/Superchain-Interop-16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: An `ExecutingMessage` is emitted even if the `isDeposit` flag is correctly set

- **Description:** If a deposit transaction calls `validateMessage` without reverting and emitting the event `ExecutingMessage`, the sequencer could be forced to include messages that do not correspond to an existing identifier. This could break multiple interop invariants and eventually lead to a bricked chain.
- **Risk Assessment:** Medium.
    - Potential Impact: High. The impact of such an attack would vary depending on the message. Examples of economic attacks include:
        - Releasing all ETH from the `ETHLiquidity` contract.
        - Relaying an ERC20 with a custom amount never burned on the source chain.
        
        In the worst-case scenario, the chain could produce unsafe blocks that are subsequently rejected, leading to a reorg, which would effectively brick the chain afterwards. This occurs because the chain cannot progress with a deposit transaction containing an invalid execution. Services such as relay-based bridges could also face significant consequences if they fail to detect the situation promptly.
        
    - Likelihood: Low. This could only happen with a bugged implementation of the `CrossL2Inbox` check. Even if the current implementation is bug-free, future upgrades can introduce these bugs.
    Something important to notice is that most chains within the same cluster will probably share implementations, so a bug might affect all chains.
- **Mitigations:** Our current codebase includes tests to check that the `CrossL2Inbox` contract validate messages only when `isDeposit` is `false` ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol#L139)) and revert when it is `true` ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol#L166)). The `L1BlockInterop` contract also includes test to check this property ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L205)).  The security team should be aware of this issue and check for it in every protocol version upgrade. There should be a way to prevent to process such deposits (through `op-supervisor` or other), so the chain can continue, at the impact of halting deposits, until [sequencing windows](https://specs.optimism.io/glossary.html?highlight=sequencing%20window#sequencing-window) end.
- **Detection:** The `op-supervisor` could detect such a situation within the invariants checks. If `op-supervisor` simulations (or if there are tools for local sequencers to simulate deposits) are properly implemented before including new deposits, it would allow the sequencer to delay the deposit’s inclusion until the sequencing window ends. This delay would provide a time buffer to fix the issue.
- **Recovery Path(s):** Stop processing deposits and stay aware of the sequencing window. A chain halt followed by a fix and a reorg would be necessary.

### FM2: `isDeposit` is not turned on before deposit transactions

- **Description:** This would allow deposit transactions to emit `ExecutingMessage` events, even if the check in the `CrossL2Inbox` is working correctly.
- **Risk Assessment:** Medium.
    - Potential Impact: High. All the consequences from FM1 would apply.
    - Likelihood: Low. This would also correspond to a bugged implementation in the `L1Block` contract or client-triggered calls to it.
- **Mitigations:** Our current codebase include tests to check `L1BlockInterop` set `isDeposit` as `true` during the deposit context ([reference](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L239)). The security team should know this issue and check for it in every protocol version.
- **Detection:** Same as FM1. Off-chain services can detect flag misbehavior by checking every block's first transaction.
- **Recovery Path(s):** Stop processing deposits and stay aware of the sequencing window. A chain halt followed by a fix would be necessary. A reorg can also be considered if invariant breaking messages were created.

### FM3: `isDeposit` is not turned off after deposit transactions

- **Description:** If the `depositComplete()` call within `L1BlockInterop` fails or is never initiated, the `isDeposit` flag might remain on. This would imply that every call to `validateMessage()` will be seen as a deposit and therefore revert.
- **Risk Assessment:** Low.
    - Potential Impact: Medium. Genuine cross-chain messages will not be able to execute. This should not be a major issue, as users can re-execute after the fix.
    - Likelihood: Low. This could happen if the `depositComplete()` implementation is bugged or the sequencer is not triggering the call to the function. The latter could occur due to a client bug or an out-of-gas error, which is unlikely.
- **Mitigations:** Our current codebase includes tests to check `L1BlockInterop` set `isDeposit` as `false` after the deposit context ends ([test](https://github.com/ethereum-optimism/optimism/blob/ef6ef6fd45fc2b7ccd4bc06dc7e24f75c0dda362/packages/contracts-bedrock/test/L2/L1BlockInterop.t.sol#L292)). The security team should know this issue and check for it in every protocol version.
- **Detection:** Offchain services should be aware of this possibility for `validateMessage` reverts.
- **Recovery Path(s):** Execute the proper fixes depending on whether it was a sequencer or contract error. Valid reverted messages can be re executed on destination, or resent if expired.

### FM4: `upgradeTxs` include an invalid `ExecutingMessage`

- **Description:** The `isDeposit` bool is set off before the upgrade transactions, which are force-included. This implies that, if an upgrade transaction includes a call to `validateMessage`, the sequencer will be forced to include it, even if it doesn't point to an existing identifier.
- **Risk Assessment:** Low to Medium.
    - Potential Impact: High. It could impact the same way described in FM1.
    - Likelihood: Low. An upgrade transaction should not call `validateMessage()` unless it is somehow intended to (and therefore not malicious). What's more, every upgrade transaction should bypass many security checks.
- **Mitigations:** Every upgrade transaction should be simulated. An invalid Superchain state would be caught by the `op-supervisor` in simulations.
- **Detection:** Upgrades are heavily monitored transactions, making it very unlikely to go unnoticed.
- **Recovery Path(s):** Recovery would be similar to other bugged upgrades. See [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md) for more details.

### Generic items we need to take into account:

- Every consideration already covered by the [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md) will also apply to these changes.
- It is important to have a good gas benchmark for `L1Block.depositsComplete()` in the client to minimize out of gas errors.

## Action Items

- [ ]  Resolve all the comments.
- [ ]  FM1, FM2, FM3: Make sure `op-supervisor`, deposit simulations and other relevant off-chain components are properly put in place (TO BE DECIDED) to monitor and cover all the cases, being ready before to this implementation goes to production.

## Audit Requirements

We suggest the modifications for banning deposit triggering `ExecutingMessage` events go through an audit, as they affect a sensitive part of the protocol. Following on the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), some presented failure modes are similar to the ”Deposit path no spoofing”, as it affects the validity of the deposits. In particular, referencing a non-existing cross-chain message with a deposit can be considered spoofing.

## Additional Notes

Something worth noticing is that implementing the ban deposit closes the door for intended `ExecutingMessage` emissions from deposits. This feature could be desirable to force valid cross-chain messages from L1 and bypass sequencer censorship. There is an active exploration to ensure that the censorship-resistance property, with the introduction of [preregistrations](https://github.com/ethereum-optimism/specs/issues/520).
