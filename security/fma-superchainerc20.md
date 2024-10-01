## **SuperchainERC20 FMA (Failure Modes and Recovery Path Analysis)**

| Author | Particle, Skeletor, Joxes |
| --- | --- |
| Created at | 2024-09-30 |
| Needs Approval From | Matt Solomon, Mark Tyneway |
| Other Reviewers |  |
| Status | Review |

## Introduction

---

This document is intended to be shared publicly for reviews and visibility purposes. It covers the following:

1. Introduction of the `SuperchainERC20` standard:
    - Introducing the `SuperchainERC20` baseline implementation.
    - Introducing the `SuperchainERC20Bridge`.
2. Introduction of the liquidity migration implementation for L1 bridged tokens as a `SuperchainERC20` extension:
    - Updating the `L2StandardBridge` to allow token conversion.
    - Updating the `OptimismMintableERC20Factory`.
    - Introducing the `OptimismSuperchainERC20Factory` predeploy.
    - Introducing the `OptimismSuperchainERC20` implementation.
    - Introducing the `BeaconContract` predeploy used by the `OptimismSuperchainERC20` BeaconProxies.

Below are references for this project:

- The new contracts and updates are documented in the following specs:
    - [Predeploys specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/predeploys.md).
    - [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md)

## Failure Modes and Recovery Paths

---

### Incorrect or bypassing `_homeChain` definition

- **Description:** During a `SuperchainERC20` deployment, `_homeChain` is defined as the chain where the initial supply is issued, generally to the owner. This is verified by matching with `block.chainId`. If this check is incorrectly handled or bypassed, new supply could be created on other chains during deployment, unexpectedly inflating the total supply.
- **Risk Assessment**: Medium
    - Potential impact: High. Users will face dilution and experience other side effects where the supply directly influences the system.
    - Likelihood: Low. Setting a different `_homeChain` value or modifying the contract to relax this condition will result in generating a different address.
- **Mitigation**: A code audit should ensure that the `_homeChain` check is properly implemented.
- **Detection**: Monitor the initial token supply at every deployment to detect any unexpected supply creation.
- **Recovery Path(s)**: Depends on the token. For the basic and non-ownable `SuperchainERC20`, token redeployment and user migration will be required. Other types of tokens may opt for different recovery paths.

### Message unable to finalize in Cross-Chain Transfers

- **Description:** During a cross-chain transaction, tokens are burned on the source chain via `sendERC20` and minted on the destination chain via `relayERC20` through `SuperchainERC20Bridge` and utilizing the `L2toL2CrossDomainMessenger`. However, if the destination chain doesn’t listen to messages from the origin chain, the message will never be included in `CrossL2Inbox` and therefore the tokens will be burned on the origin but not minted on the destination. This results in users’ funds being stuck or perceived as lost.
- **Risk Assessment**: High
    - Potential impact: High. Users may lose access to their funds, indefinitely.
    - Likelihood: Medium. Currently there is no way to stop a cross-chain transaction whose destination doesn’t include the origin chain in their dependency set.
- Mitigation: To be defined at the architecture level. In the meantime, user interfaces should prevent users from initiating a message that cannot be relayed.
- **Detection**: Monitor cross-chain message queues and maintain an updated list of chain dependencies to identify potential incompatibilities proactively.
- **Recovery Path(s)**: None. If systemic, consider a protocol upgrade.

### `relayERC20` fails in Cross-Chain Transfers

- **Description:** During a cross-chain transaction, tokens are burned on the source chain via `sendERC20`, but if `relayERC20` fails, tokens are not minted. This results in users' funds being stuck or perceived as lost.
- **Risk Assessment**: Low to Medium
    - Potential impact: Medium. Users may lose access to their funds, mostly temporarily.
    - Likelihood: Low to Medium. Currently, there is no way to corroborate on-chain that a cross-chain transaction is absolutely guaranteed to finalize. However, the `SuperchainERC20Bridge` is coded to reduce at a minimum this possibility.
- **Mitigation**: Implement `expireMessage` function into `L2toL2CrossDomainMessenger` to rescue stuck transfers after a certain timeout period.
- **Detection**: Configure logging for failure events and monitors.
- **Recovery Path(s)**: Offer a user interface to reclaim the assets by calling `expireMessage` through `SuperchainERC20Bridge`. In case of systemic failure, consider a protocol upgrade to fix the issue.

### Bypass security checks in `sendERC0` or `relayERC20` functions in `SuperchainERC20Bridge`

- **Description:** The `sendERC20` and `relayERC20` generate and process outbound/inbound cross-chain transactions. The `sendERC20` will burn tokens and generate a cross-chain message. The `relayERC20` will check that the message comes from the `L2toL2CrossDomainMessenger` and the cross-chain caller is the same `SuperchainERC20Bridge` contract for the same token address and will then mint the corresponding amount.
If anyone can bypass the controls, they can access the `burn()` and `mint()` functions on the `SuperchainERC20`.
- **Risk Assessment:** Medium
    - Potential impact: High. This inflates the token supply, affecting contracts where the token is used. For the `OptimismSuperchainERC20`, it could cause other side effects, such as conversions in the `L2StandardBridgeInterop` for its legacy pair and withdrawal to L1.
    - Likelihood: Low. The contract checks seem straightforward. A protocol upgrade affecting the token and bridge implementation must go through all security checks.
    The bridges and token address invariants are well covered in the specs and should be checked.
    There could also be a problem on the message-passing side, i.e., from the `L2toL2CrossDomainMessenger`, the `CrossL2Inbox`or message validation from interop.
- **Mitigation:** The `sendERC20` and `relayERC20` functions are unit, fuzz, and invariant tested within Solidity with full coverage. They should be e2e tested, and any protocol upgrade should be tested for the bridge invariant. The new functions in the bridge will need to have their invariant tests updated to fit their new location. Interop testing campaigns should be aware of this attack.
- **Detection:** Liquidity for the collective-controlled `SuperchainERC20` and `OptimismSuperchainERC20` representations will be tracked across the Superchain. Scripts should monitor total supply invariants as described in the standard spec.
- **Recovery Path(s):** Scripts will be in place for an emergency deployment of a new implementation of `SuperchainERC20Bridge`.

### Grant access to a malicious or bugged implementation in the `convert` function on the `L2StandardBridgeInterop`

- **Description:** If someone can bypass the checks on the token's validity, it would be possible to drain locked liquidity in L1 (e.g., convert a malicious super token into a legit legacy and then withdraw to L1).
- **Risk Assessment:** Medium.
    - Potential Impact: High. The impact of such an attack would vary depending on the token, but significant liquidity is at risk.
    - Likelihood: Low. There are four ways this could happen.
        - A bug in the `_validatePair()` internal function that should only allow implementations deployed by the official factory.
        - A protocol upgrade that allows the writing of malicious or bugged addresses on the `deployments` mapping on the `OptimismMintableERC20Factory` or `OptimismSuperchainERC20Factory`.
        - A malicious or bugged implementation of the token stored in the `BeaconContract`. In particular, upgradable tokens or mintable by a party besides the `L2StandardBridgeInterop`.
        - An incorrect `creationCode` in the `OptimismSuperchainERC20Factory`, that does not deploy BeaconProxies that point to the correct `BeaconContract`.
        
        In any of the cases, the bugged implementation must bypass internal security checks.
        
- **Mitigations:**
    - The `convert` function is unit, fuzz, e2e and invariant tested with full coverage.
    - Protocol upgrades pose the greatest danger. Before implementing an upgrade, it is essential to check the `_validatePair()` invariants.
- **Detection:** Any off-chain script should maintain its registry of allowed tokens and check for anomalies in the `Converted` event emission. It is possible to introduce a minimum liquidity threshold to minimize off-chain workload, but doing so would also allow for undetected slow drain.
Moreover, withdrawals to L1 are one of the most commonly tracked variables in Optimism, so the bridge monitoring tools can also detect liquidity anomalies.
- **Recovery Path(s):** Recovery should be similar to any other attack on the `L2StandardBridgeInterop`.
There will be scripts in place to deploy a new implementation, upgrade the implementation in the `BeaconContract`, and upgrade the Factories and `L2StandardBridgeInterop` implementations.

### Block legit tokens from converting in the `L2StandardBridgeInterop`

- **Description:** The `convert` function could block a valid token on access control. This would effectively block `OptimismSuperchainERC20` from an L1 withdrawal or legacy tokens from using interop.
- **Risk Assessment:** Low.
    - Potential Impact: Low. Funds would be safe.
    - Likelihood: Low. The potential origin of this problem would be the same as those covered in the [first issue](https://www.notion.so/FMA-SuperchainERC20-V2-10b9a4c092c7806d8bd2d62ed9b9bad8?pvs=21): wrong implementation of `_validatePair` or a bugged protocol upgrade. Any error would have to bypass all internal security controls.
- **Mitigation:** The source of this issue will be the same as described in the [first issue](https://www.notion.so/FMA-SuperchainERC20-V2-10b9a4c092c7806d8bd2d62ed9b9bad8?pvs=21), so mitigation steps will also be shared.
- **Detection:** Transactions would fail, but so would simulations. The most likely scenario is that users will report the problem. Support must be aware of this potential scenario.
- **Recovery Path(s):** A protocol upgrade can be written on the `deployments` mapping to allow access for blocked tokens and fix the issue's origin.

## Existing restrictions

---

As with the current `StandardBridge` implementation, the `SuperchainERC20Bridge` may not accommodate all possible token customizations that can be built on top of the `SuperchainERC20` standard. App developers may find themselves in a position where they need to build custom bridges and variants from the standard to support their specific use cases. In some scenarios, a wrapper or lockbox might be a simple and straightforward solution; in others, it may not be.

For the following custom tokens, consider the following potential scenarios:

- **Rebasing**: Tokens that automatically adjust their total supply, affecting all holders proportionally. Balances can increase or decrease without any transfer of tokens.
    - **Case 1**: Rebase token converted 1:1 into a `SuperchainERC20` representation.
        - **Problem**: When a rebase token is locked and converted into a `SuperchainERC20` representation, it will face inconsistencies between the stored balances and the existing `SuperchainERC20` supply.
        - **Required solution**: Use a standard wrapper contract (e.g., wstETH) before bridging through the `SuperchainERC20Bridge`.
    - **Case 2**: `SuperchainERC20` with native rebase functions.
        - **Problem**: Rebasing is calculated locally for each token representation and is likely to occur asynchronously, leading to supply inconsistency and parity issues. Depending on the use case, this could affect expected calculations during cross-chain transfers. Additionally, it may incentivize malicious behavior, such as disabling or delaying minting to exploit arbitrage opportunities.
        - **Required solution**: implement a separate custom bridge where the conservation of the bridged `amount` isn’t desired to be an invariant and include additional logic to account for the time between burn and mint actions.

- **With Transfer Fees**: Tokens that charge a fee on each transfer. There is no standard way to implement this, and it may vary depending on the purpose, but generally, the fee is applied when a `transfer` function is called.
    - **Case 1**: A token with transfer fees wants to be converted 1:1 into a `SuperchainERC20` representation.
        - **Problem**: A lockbox contract would not detect that the amount deposited differs from the amount used to mint the `SuperchainERC20` tokens. The same issue applies to withdrawals. This will cause a mismatch between the existing `SuperchainERC20` supply and the amount locked.
        - **Required solution**: Implement a deposit method where the actual amount received is used to mint `SuperchainERC20` tokens, rather than the amount initially transferred.
    - **Case 2**: Collecting fees from `SuperchainERC20` transfers.
        - **Problem**: Each `SuperchainERC20` representation would collect its own fees. If set incorrectly, collected funds may be at risk of being lost, especially when the collector is intended to be a smart contract.
        - **Solution**: Use an EOA or safe same-address contract deployment method (e.g., `CREATE2` or `CREATE3`) as the collector. Alternatively, set a new collector address for each representation at deployment.

- **ERC20Votes:** Tokens that provide voting and delegation functionalities. Also known as *Gov Tokens*.
    - **Case 1**: A `SuperchainERC20` with native delegation functionalities.
        - **Problem**: During a proposal initialization, a checkpoint might be taken while there are still in-flight messages. This scenario could cause users to temporarily lose their voting power and create an opportunity for incentivized actors to disable or delay token minting in order to censor a voter or reduce the quorum.
        - **Required solution**: Users should avoid bridging their tokens close to the next expected checkpoint.
        
        **Anyways tokens and governors should be refactored at all. This involves additional challenges beyond just the token bridging method. A showcase of how to address these problems has been addressed by the Gov Token project.*

- **With deny lists**: Tokens that include functionality to prevent usage (e.g. transfer, mint) to or from specific addresses.
    - **Case 1**: `SuperchainERC20` with native deny lists that prevent transfers to specific addresses.
        - **Problem**: The `SuperchainERC20Bridge` doesn’t use a transfer function, so the `_to` address would still be able to receive tokens.
        - **Required solution**: Extend the deny list to block `_mint` to the listed addresses.
    - **Case 2**: `SuperchainERC20` with native deny lists that prevent minting to specific addresses.
        - **Problem**: When the `relayERC20` has a `_to` address that is part of the deny list, the bridging process is unable to be finalized, leaving the funds stuck.
        - **Required solution**: Use `expireMessage` to claim back the funds. Alternatively, implement a custom bridge that tracks the deny list of `SuperchainERC20`, thereby preventing the initiation of cross-chain transfers.