<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents** *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [**SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**](#superchainerc20-standard-only-fma-failure-modes-and-recovery-path-analysis)
- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Unauthorized Access to `crosschainMint` & `crosschainBurn` Functions](#fm1-unauthorized-access-to-crosschainmint--crosschainburn-functions)
  - [FM2: Different Token Addresses Across Chains](#fm2-different-token-addresses-across-chains)
  - [FM3: Same Token Address but Different (or Malicious) Implementations](#fm3-same-token-address-but-different-or-malicious-implementations)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)
- [Additional Notes](#additional-notes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## **SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**

| Author | Ng, Joxes |
| --- | --- |
| Created at | 2024-10-02 |
| Needs Approval From | Mark Tyneway, Matt Solomon, and 0age |
| Other Reviewers | - |
| Status | Review |

## Introduction

This document is intended to be shared publicly for review and visibility purposes. It covers the introduction of the `SuperchainERC20` standard, including its implementation and interface, the latter extending from `IERC7802`.

Below are references for this project:

- [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md).
- [Implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainERC20.sol).

This document is specially of importance for projects token deployers building on Superchain, as they are the main beneficiaries of this standard. It does not intend to cover related contracts such as `SuperchainTokenBridge` or those involving migrated liquidity.

## Failure Modes and Recovery Paths

### FM1: Unauthorized Access to `crosschainMint` & `crosschainBurn` Functions

- **Description:** The `crosschainMint` and `crosschainBurn` functions can only be called by the `SuperchainTokenBridge`,  enforced by the check `msg.sender != Predeploys.SUPERCHAIN_TOKEN_BRIDGE`. If the bridge address is badly defined or the modifier bypassed, an entity could mint and burn tokens. 
- **Risk Assessment**: Medium.
    - Potential impact: High. All tokens based on this implementation could be potentially at risk.
    - Likelihood: Very Low. `Predeploys.SUPERCHAIN_TOKEN_BRIDGE` is defined via protocol upgrades. The conditional is sufficiently simple and battle-tested to give confidence in the implementation.
- **Mitigation**: Ensure the `SuperchainTokenBridge` is correctly set during deployment and isn’t subject to unexpected changes.
- **Detection**: Existing off-chain scripts for token monitoring should be enough to detect any unauthorized mint or burn actions triggered by this method.
- **Recovery Path(s)**: Equivocation on `SuperchainTokenBridge` address would require a protocol upgrade or hard fork. Very unlikely to need it.

### FM2: Different Token Addresses Across Chains

- Description: For the `SuperchainTokenBridge` to validate mints and burns correctly, the `SuperchainERC20` must be deployed at the same address across all interoperable chains. If different addresses are used, the bridge will be unable to successfully finalize the cross-chain transfer.
- Risk Assessment: Medium
    - Potential impact: High. Inconsistent token addresses will disable interoperability functionalities for the contract.
    - Likelihood: Very Low. The interoperable set of chains follows the same opcode behavior and ensures identical availability of deployer contracts, such as `create2Deployer`. Developers are encouraged not to use well-known flagged deployment methods, such as `CREATE`, for these purposes.
- **Mitigation**: For developers, ensure to employ the appropiate deterministic deployment tools, such as the one at `create2Deployer`.
- **Detection**: Verify contract addresses upon deployment.
- **Recovery Path(s)**: Redeploy token contracts correctly.

### FM3: Same Token Address but Different (or Malicious) Implementations

- **Description**: The bridging logic is designed to assume that ERC20 addresses are the same across multiple chains and refer to the expected ERC20 code (which, in most cases, is expected to be identical). However, an entity could deploy different or malicious code at the same address on a given chain. If the same address property is maintained but the actual implementation differs, the bridging contract, trusting the address, might inadvertently allow cross-chain burns and mints to proceed, resulting in erratic or malicious outcomes.
- **Risk Assessment**: Medium
    - Potential impact: High. As the bridging mechanism assumes the same address on multiple chains, it assumes they represent the same ERC20 token. A malicious mismatch could allow an attacker to forge bridging events and gain unauthorized minted tokens.
    - Likelihood: Low. This is possible if the deployer engages in poor practices or lacks opsec. Two examples of this are:
        - The use of custom factories that utilize CREATE and share the same address, where tokens are deployed using the same bytecode and nonce.
        - Proxy contracts where the admin is compromised.
- **Mitigation**: For developers, ensure the use of appropriate deterministic deployment tools, such as `create2Deployer`. In the case of proxy contracts, ensure proper control over their deployments and ownership.
- **Detection**: Verify deployments at the designated address to check the bytecode and ensure it matches its counterpart on other chains or the intended version.
- **Recovery Path(s)**: Pause the token contract where possible. Upgrade if feasible, or redeploy if necessary.

## Action Items

Given the small scope, there is no need for relevant actions beyond resolving all comments, and continuing code implementation and testing.

## Audit Requirements

No audit should be required, as it is simple and isn’t expected to have dependencies or impact on other core OP contracts. In any case, an audit over the whole system is expected, including the `SuperchainERC20` contract.

## Additional Notes

The proposed implementation of the standard doesn’t prevent a token issuer from using other token standards, such as xERC20, with the `SuperchainTokenBridge` to mint and burn tokens across an [interoperable set of chains](https://specs.optimism.io/interop/overview.html). More about this is explained in [xERC20-ERC7802 compatilibility](https://defi-wonderland.notion.site/xERC20-ERC7802-compatibility-14c9a4c092c780ca94a8cb81e980d813).
