# SuperchainWETH: Failure Modes and Recovery Path Analysis

| Author | Joxes, Gotzen, Parti |
| --- | --- |
| Created at | 2025-01-10 |
| Initial Reviewers | Pending |
| Needs Approval From | Pending |
| Status | In Review |

## Introduction

This document covers the addition of SuperchainWETH, an interop-enabled version of the WETH contract that allows ETH to be moved across a set of interoperable chains. SuperchainWETH is an ERC20 contract compliant with the ERC-7802 standard and complies with the same address property. The following components are included:

- **Contracts**:
    - Introduction of the `SuperchainWETH` predeploy proxy and implementation contract. It allows to wrapping ETH into SuperchainWETH and contains the logic to allow direct ETH interop transfers ETH.
    - Introduction of the `ETHLiquidity` predeploy proxy and implementation contract, designed to ensure the conversion of SuperchainWETH into native ETH when the supply increases upon calling `crosschainMint`, by maintaining a large reserve of native ETH.

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

### FM1: Can call `sendETH` in a Custom Gas Token chain due to `isCustomGasToken`  returning `false`

- **Description:** The `sendETH` function requires `IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()` to return `true` for chains where a custom gas token is used. If this check is improperly implemented, returning `false`, the system might incorrectly allow `sendETH` calls on chains using a custom gas token.
- **Risk Assessment:** High.
    - Potential Impact: High. If the call succeeds on a chain that uses a custom gas token version, it could allow for the possibility of minting native ETH at the destination. There would be economic reasons to do so if the issue is found to be exploitable.
    - Likelihood: Low. The checker logic is simple enough and `L1Block` exposes this value at every moment, but misconfigurations may occur.
- **Mitigations:** Ensure `sendETH` checks are implemented correctly and ensure `isCustomGasToken` in L1Block is correctly configured after initialization. The sequencer in the destination chain should not validate any `relayETH` that comes from a `sendETH` call originating in a custom gas token chain.
- **Detection:** Off-chain monitoring scripts should flag `SendETH` events from custom gas token chains. The `op-supervisor` should detect such a situation within the invariants checks.
- **Recovery Path(s):** Coordinate an L2 hard fork to revert the subsequent states derived from the failure, and upgrade the predeploy contracts to patch the issue. Pause the system via `SuperchainConfig` if a withdrawal is at risk to finalize. Notify service providers that may be impacted by this failure such as bridges, oracles, and exchanges.

### FM2: Can not call `sendETH` or `crosschainMint` in native ETH chains due to `isCustomGasToken` returning `true`

- **Description:** The `sendETH` and `crosschainMint` function requires `IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()` to return `false` for chains where ETH is the native token. If this check is improperly implemented, returning `true`, the system might incorrectly deny `sendETH` or `crosschainMint` calls on chains using ETH as native asset.
- **Risk Assessment:** Medium.
    - Potential Impact: Medium. If the call revert, it would deny to send ETH and SuperchainWETH to other chains.
    - Likelihood: Low. The checker logic is simple enough and `L1Block` exposes this value at every moment, but misconfigurations may occur.
- **Mitigations:** Ensure `isCustomGasToken` in L1Block is correctly configured after initialization.
- **Detection:** Off-chain monitoring scripts should flag `SendETH` and `crosschainMint` events that reverts with custom error `isCustomGasToken` on chains that should be ETH native.
- **Recovery Path(s):** Upgrade `L1Block` contracts to properly configure `isCustomGasToken`.

### FM3: Can call `deposit` or `withdraw` in a Custom Gas Token chain due to `isCustomGasToken` returning `false`

- **Description:** The `deposit` and `withdraw` functions require `IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()` to return `true` for chains where a custom gas token is used. If this check is improperly implemented, returning `false`, the system might incorrectly allow to obtain SuperchainWETH.
- **Risk Assessment:** High.
    - Potential Impact: High. If the `deposit` call succeeds on a chain that uses a custom gas token version, it could allow for the possibility of minting SuperchainWETH using a custom gas token. There would be economic reasons to do so if the issue is found to be exploitable. The `withdraw` function is less susceptible since it requires storing native liquidity beforehand, and the custom gas token must be more valuable than SuperchainWETH. This would discourage certain types of economic attacks.
    - Likelihood: Low. The checker logic is simple enough and `L1Block` exposes this value at every moment, but misconfigurations may occur.
- **Mitigations:** Ensure `deposit` and `withdraw` checks are implemented correctly and `isCustomGasToken` in L1Block is correctly configured after initialization.
- **Detection:** Off-chain monitoring scripts should flag `Deposit` and `Withdrawal` events from custom gas token chains.
- **Recovery Path(s):** Coordinate an L2 hard fork to revert the subsequent states derived from the failure, and upgrade the predeploy contracts to patch the issue. Pause the system via `SuperchainConfig` if a withdrawal is at risk to finalize. Notify service providers that may be impacted by this failure such as bridges, oracles, and exchanges.

### FM4: Can not call `deposit` or `withdraw` in native ETH chains due to `isCustomGasToken` returning `true`

- **Description:** The `deposit` and `withdraw` function requires `IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()` to return `false` for chains where ETH is the native token. If this check is improperly implemented, returning `true`, the system might incorrectly deny `deposit` or `withdraw` calls between native ETH and SuperchainWETH.
- **Risk Assessment:** Medium.
    - Potential Impact: Medium. If any of those calls revert, it would deny to convert between native ETH and SuperchainWETH.
    - Likelihood: Low. The checker logic is simple enough and `L1Block` exposes this value at every moment, but misconfigurations may occur.
- **Mitigations:** Ensure `isCustomGasToken` in L1Block is correctly configured after initialization.
- **Detection:** Off-chain monitoring scripts should flag `Deposit` and `Withdraw` events that revert with custom error `isCustomGasToken` on chains that should be ETH native.
- **Recovery Path(s):** Upgrade `L1Block` contracts to properly configure `isCustomGasToken`.

### FM5: Insufficient ETH in `ETHLiquidity`

- **Description:** If `ETHLiquidity` lacks sufficient ETH, `mint` calls will revert when the contract attempts to forward funds using `SafeSend`. This breaks bridging flows dependent on `relayETH`  and `crosschainMint`.
- **Risk Assessment:** Medium.
    - Potential Impact: Medium. Not having enough ETH balance in `ETHLiquidity` prevents relaying ETH or cross minting SuperchainWETH, which would degrade the user experience.
    - Likelihood: Very low. `ETHLiquidity` starts with a maximum balance (`type(uint248).max`).
- **Mitigations:** Ensure initial ETH supply in `ETHLiquidity` is `type(uint248).max` after initial config.
- **Detection:** Existing off-chain scripts should detect if multiple `mint` calls revert. Additionally, check the current `ETHLiquidity` balance. Ensure that the lack of ETH is not due to other failures detailed in this document.
- **Recovery Path(s):** Coordinate an L2 hard fork to replenish the `ETHLiquidity` contract. Investigate the causes of the depletion to determine if other factors are involved.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x]  Check this box to confirm that these items have been considered and updated if necessary.

See [relevant FMAs to SuperchainWETH, To Do] 

- [ ]  Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [ ]  Resolve all the comments.
- [ ]  Proceed with the code implementation and make any necessary approved changes.
- [ ]  Move into testing of all the components.
- [ ]  Make sure there are monitoring solutions ready to cover all the cases.
- [ ]  **Ensure that the actions items specified in each FMAs on which SuperchainWETH depends are completed.**

## Audit Requirements

Following the [Audit Framework](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864), SuperchainWETH fits within the second budget, which includes smart contract code that secures assets. That means the `SuperchainWETH.sol` and `ETHLiquidity.sol` and associated interop contract dependencies (`CrossL2Inbox`, `L2ToL2CrossDomainMessenger`, `SuperchainTokenBridge`, `L1BlockInterop`, `SystemConfigInterop`, `OptimismPortalInterop`) require an audit before going to production.
