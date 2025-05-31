# Message Passing (Contracts-only): Failure Modes and Recovery Path Analysis

| Author              | Joxes                       |
| ------------------- | --------------------------- |
| Created at          | 2025-02-04                  |
| Initial Reviewers   | AgusDuha                    |
| Needs Approval From | Michael Amadi, Matt Solomon |
| Status              | Final                       |

## Introduction

This document covers the failure modes and recovery paths for the core L2 interop contracts used for cross-chain message passing. These contracts enable interoperability between chains and are used by other core L2 contracts such as `SuperchainWETH` and `SuperchainERC20`. It does not focus on the infrastructure or client-side implementation within the interop fork. It extends the failure modes considered in the SuperchainWETH and SuperchainERC20 documents.

The following components are included:

- **Contracts**:
  - Introduction of the `CrossL2Inbox` predeploy proxy and implementation contract. It provides the `validateMessage` function to verify cross-chain message integrity according to interop invariants.
  - Introduction of the `L2ToL2CrossDomainMessenger` predeploy proxy and implementation contract. It is a higher-level abstraction on top of the `CrossL2Inbox`. Serves to send and receive messages and it has replay protection, message context, and domain binding.
  - Introduction of the `SuperchainTokenBridge` predeploy proxy and implementation contract. It allows asset bridging (following ERC-7802) that leverages the `L2ToL2CrossDomainMessenger` to securely transfer tokens across chains. The contract assumes the same address property to properly relay a cross-chain transfer.

Below are references for this project:

- Specs: https://github.com/ethereum-optimism/specs/tree/main/specs/interop
- Implementation:
  - `CrossL2Inbox`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/CrossL2Inbox.sol
  - `L2ToL2CrossDomainMessenger`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/L2ToL2CrossDomainMessenger.sol
  - `SuperchainTokenBridge`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainTokenBridge.sol

> 🗂️
> **For more context about the Interop project, refer to the following docs:**
>
> 1. [Interop System Diagram](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: Valid message is initiated but destination chain lacks origin chain in its dependency set

- **Description:** A valid cross-chain message is initiated via `sendMessage` (or indirectly through contracts such as `SuperchainWETH` or `SuperchainTokenBridge`) with a specified destination chain ID. However, if the destination chain’s dependency set does not include the originating chain, the relay process (executed via `relayMessage`) will fail—preventing the message from being finalized.
- **Risk Assessment:** High.
  - Potential Impact: Critical. Users may lose access to assets such as `SuperchainWETH` or `SuperchainERC20` after burn/locked.
  - Likelihood: High. `L2ToL2CrossDomainMessenger` doesn’t check whether the `destination` is a valid value that guarantees the message will be finalized there. Users can inadvertently target an arbitrary chain ID that is not recognized in the destination’s dependency set.
- **Mitigations:** Currently, the op-interop set is intended to contain chains following a complete graph, meaning all chains include each other in their dependency set. This reduces the chance of equivocation given a known set of chain IDs. However, there are other approaches to consider:
  - Ideally, there could be some pre-send validation. The `L2ToL2CrossDomainMessenger` (or `SuperchainTokenBridge`/`SuperchainWETH`) should be able to check against a list of chain IDs whose dependency set includes the chain where the message is about to be initiated. This was previously implemented by checking the `L1BlockInterop` dependency list, serving as a convenient method since the op-interop set follows a complete graph—ensuring that dependencies are identical across all chains. However, this code is being removed.
  - Block builders/sequencers/`op-supervisor` may check if a transaction sent through the `L2ToL2CrossDomainMessenger` targets a `destination` that does not have the origin chain in its dependency set and drop it.
  - There should be a way to claim back bridged funds, for example, only after the message expiration time has elapsed.
  - Trusted bridge frontends should prevent users from sending tokens to unsupported chains.
- **Detection:** Monitor `L2ToL2CrossDomainMessenger` events that target a `destination` that does not have the origin chain in its dependency set. Support tickets filed by users reporting the issue.
- **Recovery Path(s):** There is no smooth recovery path for such equivocation other than considering the previous mitigation suggestions and formulating a potential plan for users to recover their assets, if feasible.

### FM2: `validateMessage` references an invalid or non-existent initiated message

- **Description:** An invalid or non-existent cross-chain message could be fakely relayed on the destination chain by calling `validateMessage` with an existing (but still invalid) or fabricated identifier. If executed, the message could trigger unintended actions or compromise the certainty of security of the chain’s state.
- **Risk Assessment:** High.
  - Potential Impact: Critical. Invalid message execution can lead to unauthorized token minting, asset theft, or other state inconsistencies until the dependency graph can be resolved. For example:
    - Releasing all ETH from the `ETHLiquidity` contract.
    - Relaying an ERC20 through `SuperchainTokenBridge` with a custom amount never burned on the source chain.
    - Any other contract that relies on `relayMessage`.
      Services such as relay-based could also face significant consequences if they fail to detect the situation promptly.
  - Likelihood: Medium. Sequencers and block builders could reference invalid messages by falsely indicating that an initiated message exists on a chain whose blocks are still in an unsafe state. Additionally, a seemingly legitimate initiated message could be dropped if the origin chain undergoes a reorg before posting the batch or if Ethereum itself experiences a reorg.
- **Mitigations:** `op-supervisor` ensures that sequencers can reference valid messages and reduces the risk of reorgs. Services should be aware of current [safety levels](https://specs.optimism.io/interop/verifier.html?highlight=cross-safe#safety). Chains should batch blocks at an acceptable frequency to mitigate risks.
- **Detection:** Monitoring tools should track every validated message to ensure it was observed and validated at the origin chain.
- **Recovery Path(s):** The chain will eventually resolve the issue as the dependency graph advances, but the resolution could be slow, especially if the sequencer is down or equivocating, or if chains in the dependency graph stop batching. Investigate the causes of such incidents.

### FM3: Valid message is initiated but never relayed in destination due to sequencer censorship or inaction

- **Description:** A user (or contract) may send a valid cross-chain message, but the final relay step—`relayMessage` or `validateMessage`—never occurs. This can happen even if the message is included in a finalized block on the origin chain. In some cases, a sequencer on the destination chain may intentionally choose not to relay the message (e.g., if the origin chain is not trusted or politically unfavorable to the operator).

- **Consequences:**
  - Inability to `relayETH` in `SuperchainETHBridge`
  - Inability to `relayERC20` in `SuperchainTokenBridge` to mint `SuperchainERC20`
  - Any other contract awaiting a time-sensitive `relayMessage`.

- **Risk Assessment:** High
  - Potential Impact: High. Time-sensitive or critical messages may never be processed.
  - Likelihood: Medium. Block builders/sequencers are generally controlled by a single entity per chain.

- **Mitigations:**
  - Use `resendMessage` on the origin chain to re-emit the event and allow re-processing in case the time-window has expired.
  - From a smart contract perspective, allowing calls to `validateMessage` within a deposit context to improve censorship resistance. This is currently under discussion [here](https://github.com/ethereum-optimism/specs/issues/520).

- **Detection:** Monitoring tools should track whether every initiated message has been validated at the destination by checking identifiers. Support tickets filed by users reporting the issue sustain the severity of the case in some situations.

- **Recovery Path(s):** Depends on sequencer/chain governor policy operations.

### FM4: Invalid Replayed Message (Replay attack) in `L2ToL2CrossDomainMessenger`

- **Description:** The `L2ToL2CrossDomainMessenger` enforces replay protection by marking each message as relayed in `succesfulMessages[messageHash]`. If an attacker can bypass or reset this check, a valid but previously executed message may be re-executed (relayed).
- **Risk Assessment:** High.
  - Potential Impact: High. A successful replayed message in this scenario can cause duplicate minting transfers, directly impacting `SuperchainERC20` assets or `SuperchainWETH`.
  - Likelihood: Low. `successfulMessages[messageHash]` is properly stored and cannot be overwritten but misconfigurations may occur.
- **Mitigations:** Our current codebase includes tests to ensure `relayMessage` reverts when the message has already been relayed ([test](https://github.com/ethereum-optimism/optimism/blob/66fea80cb9d06a74f0084e1b4fd1a1aa2317b19c/packages/contracts-bedrock/test/L2/L2ToL2CrossDomainMessenger.t.sol#L688)).
- **Detection:** There should be monitoring tools able to detect any repeated relayed event so that `messageHash` reappears in calls to `relayMessage`.
- **Recovery Path(s):** Coordinate an L2 hard fork to revert the subsequent states derived from the failure, and upgrade the predeploy contracts to patch the issue. Pause the system via `SuperchainConfig` if there are dependencies in L1 smart contracts (e.g., a withdrawal from a canonical or custom bridge nearing finalization in `OptimismPortal`). Notify service providers that may be impacted by this failure such as bridges, oracles, and exchanges.

### FM5: `relayMessage` reentrancies in `L2ToL2CrossDomainMessenger`

- **Description**: The `relayMessage` in `L2ToL2CrossDomainMessenger` executes a message on the destination chain after `validateMessage` in `CrossL2Inbox`, which is safeguarded from reentrancies by `nonReentrant`. If this function is buggy or poorly implemented, an entity may exploit it to execute a reentrancy attack.
- **Risk Assessment:** Medium.
  - Potential Impact: Critical. This could lead to repeated relay of messages. Examples include:
    - Multiple calls that force execution of `crosschainMint` or `mint` in `SuperchainERC20` and `SuperchainWETH`, respectively. This would break asset invariants, allowing `SuperchainERC20` or ETH assets to be minted improperly, impacting other smart contracts, initiating unauthorized withdrawals, and affecting services such as exchanges and third-party bridges.
  - Likelihood: Very low. The use of `TransientContext` with reentrancy guards, combined with a careful implementation of the check-effect-interaction pattern in the contract code, makes it robust against such scenarios.
- **Mitigations:** Our current codebase includes tests to ensure `relayMessage` reverts when reentrancy is attempted ([test](https://github.com/ethereum-optimism/optimism/blob/66fea80cb9d06a74f0084e1b4fd1a1aa2317b19c/packages/contracts-bedrock/test/L2/L2ToL2CrossDomainMessenger.t.sol#L481)).
- **Detection:** There should be monitoring tooling that detect repeated calls and event emissions for the same transaction.
- **Recovery Path(s):** Coordinate an L2 hard fork to revert the subsequent states derived from the failure, and upgrade the predeploy contracts to patch the issue. Pause the system via `SuperchainConfig` if there are dependencies in L1 smart contracts (e.g., a withdrawal from a canonical or custom bridge nearing finalization in `OptimismPortal`). Notify service providers that may be impacted by this failure such as bridges, oracles, and exchanges.

### FM6 : A repeated identifier is validated through `CrossL2Inbox`

- **Description:** The `validateMessage` in `CrossL2Inbox` does not store or track whether a given identifier has been used before. While this is an intended feature that could be used for different purposes, if the same identifier is provided in multiple calls when it shouldn't be able to do so, messages could be executed multiple times, leading to unintended consequences.
- **Risk Assessment:** Medium.
  - Potential Impact: High. A `validateMessage` call with a repeated identifier could cause an action to be executed multiple times in contracts that rely on `CrossL2Inbox`, similar to the impact in FM4. Currently, `L2ToL2CrossDomainMessenger` has replay protection.
  - Likelihood: Low. Core contracts such as `SuperchainTokenBridge` and `SuperchainWETH` relies on `L2ToL2CrossDomainMessenger`, but other contracts that rely solely on `CrossL2Inbox` could be affected.
- **Mitigations:** This possibility needs to be properly documented and communicated, as developers will likely find it relevant for use cases that don't rely on `L2ToL2CrossDomainMessenger` and instead rely directly on `CrossL2Inbox` and custom contracts.
- **Detection:** Monitoring tools should track validated messages to ensure they have been seen and validated at the origin and are not duplicated, for the case of the `L2ToL2CrossDomainMessenger`, already covered by FM4.
- **Recovery Path(s):** If the issue impacts a core contract such as `L2ToL2CrossDomainMessenger`, it would require a fix and a rollback.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [x] Resolve all the comments.
- [x] FM1: Decide whether to implement any of the mitigation approaches described, either now or in the future. (re-introduce the L2 Dependency manager predeploy in the future)
- [x] FM1: Decide whether to implement a monitoring solution that tracks when a message is sent to a `destination` that is not part of the dependency set. (tracked on [Interop Monitoring issue](https://github.com/ethereum-optimism/optimism/issues/15178))
- [x] FM2: Implement a monitoring tool that tracks invalid/non-existent initiated messages. (tracked on [Interop Monitoring issue](https://github.com/ethereum-optimism/optimism/issues/15178))
- [x] FM3: Check for any L2Beat Stage 1 considerations related to the inability to successfully call `validateMessage` as part of a force-include a transaction. (tracked in the [Censorship Resistance issue](https://github.com/ethereum-optimism/specs/issues/520))
- [x] FM3: Implement a mechanism that can execute `validateMessage` that is censorship-resistant to mitigate the current risks, as discussed [here](https://github.com/ethereum-optimism/specs/issues/520).
- [x] FM3: Implement a monitoring tool that tracks and flags initiated messages that are not relayed despite being ready. (tracked on [Interop Monitoring issue](https://github.com/ethereum-optimism/optimism/issues/15178))
- [x] FM4: Implement a monitoring tool that alerts when relayed events are repeated. (Covered by tests no monitoring solution)
- [x] FM5: Implement a monitoring tool that alerts successful reentrancies. (Covered by tests no monitoring solution)
- [x] FM6: Ensure that the Interop documentation for developers explains the possibility of repeated identifiers being validated during a `validatedMessage`, the role of `L2ToL2CrossDomainMessenger`, and whether replay protections are required, depending on the expected use case when developing custom messengers. (The current documentation is in [Interop docs](https://docs.optimism.io/stack/interop/message-passing))
- [x] Write a FMA for `op-supervisor`, as this component is critical and transversal to many of the cases explained, since its failure could degrade liveness or the safety of interop. (Follow in [Supervisor FMA PR](https://github.com/ethereum-optimism/design-docs/pull/233))
- [x] Ensure the support team is aware of these failure modes and prepared to respond.

## Audit Requirements

Following the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), interop contracts fit within the second budget, which includes smart contract code that secures assets. That means the `CrossL2Inbox`, `L2ToL2CrossDomainMessenger`, `L1BlockInterop`, and `SuperchainTokenBridge` require an audit before going to production. It is also recommended to audit `op-supervisor`, as it is a critical component for message passing which sequencers and other full nodes may rely on.
