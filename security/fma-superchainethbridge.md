# SuperchainETHBridge: Failure Modes and Recovery Path Analysis

| Author              | Joxes, Gotzen, Parti          |
| ------------------- | ----------------------------- |
| Created at          | 2025-01-10                    |
| Initial Reviewers   | Kelvin Fichter, Michael Amadi |
| Needs Approval From | Kelvin Fichter                |
| Status              | Implementing Actions          |

## Introduction

This document covers the addition of the `SuperchainETHBridge` contract, an abstraction layer on top of the `L2toL2CrossDomainMessenger` specifically designed for native ETH transfers between chains. The following components are included:

- **Contracts**:
  - Introducing the `SuperchainETHBridge`predeploy proxy and implementation contract. It uses the `L2toL2CrossDomainMessenger` to send and relay messages that handle ETH transfers between chains.
  - Introducing the `ETHLiquidity` predeploy proxy and implementation contract. It is designed to provide the ETH for the `SuperchainETHBridge` to facilitate ETH transfers between chains, by maintaining a large reserve of native ETH.

Below are references for this project:

- Design doc:
  - Introduction of SuperchainETHBridge: https://github.com/ethereum-optimism/design-docs/blob/main/protocol/superchain-eth-bridge.md
  - Handling ETH transfers: https://github.com/ethereum-optimism/design-docs/blob/main/protocol/interoperable-ether-transfers.md
- Specs:
  - `SuperchainETHBridge`: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/superchain-eth-bridge.md
  - `ETHLiquidity`: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/eth-liquidity.md
- Implementation:
  - `SuperchainETHBridge`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainETHBridge.sol
  - `ETHLiquidity`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/ETHLiquidity.sol

> 🗂️ **For more context about the Interop project, refer to the following docs:**
>
> 1. [Interop System Diagram](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: Insufficient ETH in `ETHLiquidity`

- **Description:** If `ETHLiquidity` lacks sufficient ETH, mint calls will revert when the contract attempts to forward funds using `SafeSend`. This breaks bridging flows dependent on `relayETH`.
- **Risk Assessment:** Medium.
  - Potential Impact: High. Not having enough ETH balance in `ETHLiquidity` prevents relaying ETH, which would greatly degrade the user experience, as they would be temporarily stuck.
  - Likelihood: Very low. `ETHLiquidity` starts with a maximum balance (`type(uint248).max`).
- **Mitigations:** Our current codebase includes tests to check if the `mint` request exceeds the contract’s balance ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L103)). Initial ETH supply in `ETHLiquidity` is `type(uint248).max` ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L29)) and equally set in all chains.
- **Detection:** Perform checks on the `ETHLiquidity` balance as a preventive monitoring measure. User-filed support tickets may flag this issue in case of failure.
- **Recovery Path(s):** Coordinate an L2 hard fork to replenish the `ETHLiquidity` contract. Investigate the causes of the depletion to determine if other factors are involved. Messages will need to be retried for relaying, leveraging the `resendMessage` feature if needed.

### FM2: Overflow during `burn` in `ETHLiquidity` due to max Balance

- **Description:** If the `ETHLiquidity` contract has already reached the maximum allowed ETH balance (`type(uint256).max`), invoking the `sendETH` function could cause an overflow. This occurs when `burn` is called with an amount that, when added to the current balance, exceeds the maximum `uint256` value. Such an overflow would cause a revert, since the proper Solidity version is used.
- **Risk Assessment:** Low.
  - Potential Impact: Medium. Reverts during `sendETH` calls are expected when the requested amount exceeds the maximum value representable by a `uint256`.
  - Likelihood: Very low. `ETHLiquidity` starts with a maximum balance (`type(uint248).max`) which is still far from `uint256` limit.
- **Mitigations:** Initial ETH supply in `ETHLiquidity` is `type(uint248).max` properly is checked ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L29)).
- **Detection:** Perform checks on the `ETHLiquidity` balance as a preventive monitoring measure. User-filed support tickets may flag this issue in case of failure.
- **Recovery Path(s):** Coordinate an L2 hard fork to adjust the `ETHLiquidity` contract’s state. Investigate the causes of this failure to determine if other factors are involved.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

See [relevant FMAs to SuperchainETHBridge, To Do]

- [ ] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [x] Resolve all the comments.
- [ ] FM1, FM2: Establish a balance monitoring measure in `ETHLiquidity` (optional).
- [ ] Ensure the support team is aware of these failure modes and prepared to respond.
- [ ] **Ensure that the actions items specified in each FMAs on which SuperchainETHBridge depends are completed.**

## Audit Requirements

Following the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), SuperchainETHBridge fits within the second budget, which includes smart contract code that secures assets. That means the `SuperchainETHBridge.sol` and `ETHLiquidity.sol` and associated interop contract dependencies (`CrossL2Inbox`, `L2ToL2CrossDomainMessenger`, `SuperchainTokenBridge`, `L1BlockInterop`, `SystemConfigInterop`, `OptimismPortalInterop`) require an audit before going to production.
