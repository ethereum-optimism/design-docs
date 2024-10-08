## **SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**

| Author | Ng, Joxes |
| --- | --- |
| Created at | 2024-10-02 |
| Needs Approval From | Mark Tyneway, Matt Solomon, and 0age |
| Other Reviewers | - |
| Status | Review |

## Introduction

This document is intended to be shared publicly for review and visibility purposes. It covers the introduction of the `SuperchainERC20` standard, including its implementation and interface, the latter extending from `ICrosschainERC20`.

Below are references for this project:

- [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md).
- [Implementation](https://github.com/defi-wonderland/optimism/tree/sc-feat/crosschain-erc20).

It does not intend to cover related contracts such as `SuperchainTokenBridge` or those involving migrated liquidity.

## Failure Modes and Recovery Paths

### Unauthorized Access to `__crosschainMint` & `__crosschainBurn` Functions

- **Description:** The `onlySuperchainTokenBridge` modifier only allows `__crosschainMint` and `__crosschainBurn` to be callable by the `SuperchainTokenBridge`. If the bridge address is badly defined or the modifier bypassed, an entity could mint and burn tokens.
- **Risk Assessment**: Medium.
    - Potential impact: High. All tokens based on this implementation could be potentially at risk.
    - Likelihood: Very Low. `Predeploys.SUPERCHAIN_TOKEN_BRIDGE` are defined via protocol upgrades. The modifiers are sufficiently simple and battle-tested to give confidence in the implementation.
- **Mitigation**: Ensure the `SuperchainTokenBridge` is correctly set during deployment and isn’t subject to unexpected changes.
- **Detection**: Existing off-chain scripts for token monitoring should be enough to detect any unauthorized mint or burn actions triggered by this method.
- **Recovery Path(s)**: Equivocation on `SuperchainTokenBridge` would require a protocol upgrade or hard fork. Very unlikely to need it.

### Different Token Addresses Across Chains

- Description: For the `SuperchainTokenBridge` to validate mints and burns correctly, the `SuperchainERC20` must be deployed at the same address across all interoperable chains. If different addresses are used, the bridge will be unable to successfully finalize the cross-chain transfer.
- Risk Assessment: Medium
    - Potential impact: High. Inconsistent token addresses will disable interoperability functionalities for the contract.
    - Likelihood: Very Low. The interoperable set of chains follows the same opcode behavior and ensures identical availability of deployer contracts, such as `create2Deployer`. Developers are encouraged not to use well-known flagged deployment methods, such as `CREATE`, for these purposes.
- **Mitigation**: For developers, ensure to employ the appropiate deterministic deployment tools, such as the one at `create2Deployer`.
- **Detection**: Verify contract addresses upon deployment.
- **Recovery Path(s)**: Redeploy token contracts correctly.

## Action Items

Given the small scope, there is no need for relevant actions beyond resolving all comments, and continuing code implementation and testing.

## Audit Requirements

No audit should be required, as it is simple and isn’t expected to have dependencies or impact on other core OP contracts.

## Additional Notes

The proposed implementation of the standard doesn’t prevent a token issuer from using other token standards, such as xERC20, with the `SuperchainTokenBridge` to mint and burn tokens across an [interoperable set of chains](https://specs.optimism.io/interop/overview.html).