# SuperchainWETH: Failure Modes and Recovery Path Analysis

| Author | Joxes, Gotzen, Parti |
| --- | --- |
| Created at | 2025-01-10 |
| Initial Reviewers | Pending |
| Needs Approval From | Pending |
| Status | In Review |

## Introduction

This document covers the addition of SuperchainWETH, an interop-enabled version of the WETH contract that allows ETH to be moved across a set of interoperable chains. It act as a wrapper contract compliant with the SuperchainERC20 standard and also have built-in bridging logic. The following components are included:

- **Contracts**:
    - Introducing the `SuperchainWETH` predeploy proxy and implementation contract. It enables wrapping ETH into SuperchainWETH and includes logic for direct ETH interop transfers.
    - Introducing the `ETHLiquidity` predeploy proxy and implementation contract. It is designed to facilitate the conversion of SuperchainWETH into native ETH—triggered when the supply increases through `crosschainMint` via `relayERC20`—by maintaining a large reserve of native ETH.

Below are references for this project:

- Design doc:
    - Introduction of SuperchainWETH: https://github.com/ethereum-optimism/design-docs/blob/main/protocol/interoperable-ether.md
    - Handling SuperchainWETH transfers: https://github.com/ethereum-optimism/design-docs/blob/main/protocol/interoperable-ether-transfers.md
- Specs: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/superchain-weth.md
- Implementation:
    - `SuperchainWETH`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainWETH.sol
    - `ETHLiquidity`: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/ETHLiquidity.sol


>🗂️ **For more context about the Interop project, refer to the following docs:**
> 1. [Interop System Diagram](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)



## Failure Modes and Recovery Paths

### FM1: Insufficient ETH in `ETHLiquidity`

- **Description:** If `ETHLiquidity` lacks sufficient ETH, mint calls will revert when the contract attempts to forward funds using `SafeSend`. This breaks bridging flows dependent on `relayETH`  and `crosschainMint`.
- **Risk Assessment:** Low.
    - Potential Impact: Medium. Not having enough ETH balance in `ETHLiquidity` prevents relaying ETH or mint SuperchainWETH through `crosschainMint`, which would degrade the user experience.
    - Likelihood: Very low. `ETHLiquidity` starts with a maximum balance (`type(uint248).max`).
- **Mitigations:** Our current codebase includes tests to check if the `mint` request exceeds the contract’s balance ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L103)). Initial ETH supply in `ETHLiquidity` is `type(uint248).max` ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L29)) and equally set in all chains.
- **Detection:** Ideally, off-chain scripts should detect if multiple `mint` calls revert. Additionally, check the current `ETHLiquidity` balance. If that’s not feasible, user-filed support tickets may flag this issue, since funds are not at risk of loss due to this failure.
- **Recovery Path(s):** Coordinate an L2 hard fork to replenish the `ETHLiquidity` contract. Investigate the causes of the depletion to determine if other factors are involved.

### FM2: Overflow during `burn` in `ETHLiquidity` due to max Balance

- **Description:** If the `ETHLiquidity` contract has already reached the maximum allowed ETH balance (`type(uint256).max`), invoking the `sendETH` function (which triggers `crosschainBurn`) could cause an overflow. This occurs when `burn` is called with an amount that, when added to the current balance, exceeds the maximum `uint256` value. Such an overflow would cause a revert, since the proper Solidity version is used.
- **Risk Assessment:** Low.
    - Potential Impact: Medium. Reverts during `sendETH` calls are expected when the requested amount exceeds the maximum value representable by a `uint256`.
    - Likelihood: Very low. `ETHLiquidity` starts with a maximum balance (`type(uint248).max`) which is still far from `uint256` limit.
- **Mitigations:** Initial ETH supply in `ETHLiquidity` is `type(uint248).max` properly is checked ([test](https://github.com/ethereum-optimism/optimism/blob/dd37e6192c37ed4c5b18df0269f065f378c495cc/packages/contracts-bedrock/test/L2/ETHLiquidity.t.sol#L29)).
- **Detection:** Given the unlikely nature and potential impact of this case, user-filed support tickets should be sufficient to flag this issue since funds are not at risk of loss due to this failure.
- **Recovery Path(s):** Coordinate an L2 hard fork to adjust the `ETHLiquidity` contract’s state. Investigate the causes of this failure to determine if other factors are involved.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x]  Check this box to confirm that these items have been considered and updated if necessary.

See [relevant FMAs to SuperchainWETH, To Do] 

- [ ]  Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [ ]  Resolve all the comments.
- [ ]  FM1: Decide whether to use off-chain scripts or rely on a user-support system for detection.
- [ ]  Ensure the support team is aware of these failure modes and prepared to respond.
- [ ]  **Ensure that the actions items specified in each FMAs on which SuperchainWETH depends are completed.**

## Audit Requirements

Following the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), SuperchainWETH fits within the second budget, which includes smart contract code that secures assets. That means the `SuperchainWETH.sol` and `ETHLiquidity.sol` and associated interop contract dependencies (`CrossL2Inbox`, `L2ToL2CrossDomainMessenger`, `SuperchainTokenBridge`, `L1BlockInterop`, `SystemConfigInterop`, `OptimismPortalInterop`) require an audit before going to production.
