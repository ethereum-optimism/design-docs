<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## **Table of Contents** *generated with [DocToc](https://github.com/thlorenz/doctoc)*  

- [**SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**](#superchainerc20-standard-only-fma-failure-modes-and-recovery-path-analysis)
- [Introduction](#introduction)
  - [Security Considerations](#security-considerations)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Unauthorized Access to `crosschainMint` & `crosschainBurn` Functions](#fm1-unauthorized-access-to-crosschainmint--crosschainburn-functions)
  - [FM2: Token contract not Deployed in Destination Chain, but Able to Deploy](#fm2-token-contract-not-deployed-in-destination-chain-but-able-to-deploy)
  - [FM3: Token Deployer is Lost, or Unable to Deploy to Expected Address](#fm3-token-deployer-is-lost-or-unable-to-deploy-to-expected-address)
  - [FM4: Compromised Deployment Method](#fm4-compromised-deployment-method)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)


<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## **SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**

| Author | Ng, Joxes, Particle, Gotzen |
| --- | --- |
| Created at | 2024-10-02 |
| Needs Approval From | Mark Tyneway, Matt Solomon |
| Other Reviewers | Michael Amadi |
| Status | Review |

## Introduction

This document is intended to be shared publicly for review and visibility purposes. It covers the introduction of the `SuperchainERC20` standard, including its implementation and interface, the latter extending from `IERC7802`.

Token deployers building on Superchain are encouraged to review the document, as they are the main beneficiaries of this standard. It does not intend to cover contracts such as SuperchainTokenBridge or those involving migrated liquidity.

>🗂️ The proposed implementation of the standard doesn’t prevent a token issuer from using other token standards, such as xERC20, with the `SuperchainTokenBridge` to mint and burn tokens across an [interoperable set of chains](https://specs.optimism.io/interop/overview.html). More about this is explained in [xERC20-ERC7802 compatilibility](https://defi-wonderland.notion.site/xERC20-ERC7802-compatibility-14c9a4c092c780ca94a8cb81e980d813).

Below are references for this project:

- [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md).
- [Implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainERC20.sol).

### Security Considerations

Similar to ERC20, SuperchainERC20 implementations should be considered untrusted by default because the `crosschainMint` and `crosschainBurn` methods are not constrained by `IERC7802` or the SuperchainERC20 reference implementation. Consequently, failure modes stemming from malicious SuperchainERC20 implementations are not considered here.

## Failure Modes and Recovery Paths

### FM1: Unauthorized Access to `crosschainMint` & `crosschainBurn` Functions

- **Description:** The `crosschainMint` and `crosschainBurn` functions can only be called by the `SuperchainTokenBridge`,  enforced by the check `msg.sender != Predeploys.SUPERCHAIN_TOKEN_BRIDGE`. If the bridge address is badly defined or the modifier bypassed, an entity could mint and burn tokens.
- **Risk Assessment**: Medium.
    - Potential impact: High. All tokens based on this implementation could be potentially at risk.
    - Likelihood: Very Low. `Predeploys.SUPERCHAIN_TOKEN_BRIDGE` is defined via protocol upgrades. The conditional is sufficiently simple and battle-tested to give confidence in the implementation.
- **Mitigation**: Current unit tests perform authorization checks for `crosschainMint` ([test](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/L2/SuperchainERC20.t.sol#L38)) and `crosschainBurn` ([test](https://github.com/ethereum-optimism/optimism/blob/1add88a41b8cdc5c488a18976e3e4fa4b02e7da4/packages/contracts-bedrock/test/L2/SuperchainERC20.t.sol#L77)). There should be tests to ensure that the `SuperchainTokenBridge` predeploy is correctly set during the interop fork upgrade, and is not subject to unexpected changes.
- **Detection**: Ideally, run an off-chain agent to monitor every `crosschainMint` and `crosschainBurn` event in real-time, perhaps it could be resource-intensive given the permissionless usage of `SuperchainERC20` and the expected large number of deployments. If that’s not feasible, rely on user-filed support tickets to flag unusual mint/burn activity, which the TBD team (see corresponding action item) can investigate for suspected unauthorized access.
- **Recovery Path(s)**: Equivocation (i.e. another implementation being set) on the `SuperchainTokenBridge` address would require a protocol upgrade or hard fork to restore the expected code.

### FM2: Token contract not Deployed in Destination Chain, but Able to Deploy

- **Description**: When a cross-chain transfer is initiated (via `sendERC20`), the destination chain must have the  `SuperchainERC20` contract deployed at the same address. If the contract does not exist on the destination chain, the `relayERC20`  cannot mint the tokens and thus will fail.
- **Risk Assessment**: Medium
    - Potential impact: Medium. Users can lose access to their tokens temporarily or permanently if the deployer (in the case of permissioned tokens) doesn’t deploy the token afterwards.
    - Likelihood: Medium. A deployer may choose not to deploy the token on all chains.
- **Mitigation**: Trusted bridge frontends should prevent users from sending tokens to a chain where the token doesn’t exist. Double check other trusted sources (such as Superchain Token List) for greater confidence.
- **Detection**: Support tickets filed by users reporting the issue.
- **Recovery Path(s)**: Deploy the token.

### FM3: Token Deployer is Lost, or Unable to Deploy to Expected Address

- **Description**: For the `SuperchainTokenBridge` to validate cross-chain mints and burns correctly, the `SuperchainERC20` must appear at one consistent address on each chain when deployed. If a developer fails to deterministically deploy the token at the same address, the bridging logic cannot unify the token references across chains, leading to failed or incorrect cross-chain transfers.
- **Risk Assessment**: Medium
    - Potential impact: High. Without a consistent address, interoperability for the token is effectively disabled.
    - Likelihood: Very Low. The interoperable set of chains follows the same opcode behavior and ensures identical availability of deployer contracts, such as `create2Deployer`.
- **Mitigation**: For developers, employ the appropriate deterministic deployment tools, such as the one at `create2Deployer`.
- **Detection**: Deployers should test the deployment method used and verify contract addresses after a mainnet deployment.
- **Recovery Path(s)**: Redeploy token contracts on a new address across chains and migrate userbase if it is needed.

### FM4: Compromised Deployment Method

- **Description**: The bridging logic assumes that if a `SuperchainERC20` contract address is identical across multiple chains, it refers to the expected code. However, if a deployer uses flawed or deployment methods that get compromised, a non-desired implementation may be deployed at the same address on one chain.
- **Risk Assessment**: Medium
    - Potential impact: High. In the worst-case scenario, compromised methods open the door to malicious behavior, e.g. where an attacker can occupy the address, and forge bridging events, and gain the ability to mint tokens in other chains.
    - Likelihood: Low. This is possible if the deployer engages in poor practices or lacks opsec. Two examples of this are:
        - Custom factories that utilize CREATE and share the same address are used, where tokens are deployed using the same bytecode and nonce.
        - Proxy contracts where the admin is compromised or the proxy address front-runned.
- **Mitigation**: For developers, employ the appropriate deterministic deployment tools, such as the one at `create2Deployer`. In the case of permissioned deployments, common for proxy contracts, ensure to maintain control over those deployments and ownership.
- **Detection**: Support tickets filed by users reporting such issues.
- **Recovery Path(s)**: Pause the token contract where possible and proceed to redeploy and migrate userbase if it is needed.

## Action Items

The following action items need to be done:

- [ ]  Resolve all the comments.
- [ ] Communicate to bridge frontends to ensure they prevent users from sending tokens to a chain where the token doesn’t exist (FM2)
- [ ]  Implement tests for `SuperchainTokenBridge` predeploy. Currently, the L2 Genesis project is under development and it should take care of these tests (FM1)
- [ ]  Decide whether to use off-chain scripts or rely on a user-support system for FM1.
- [ ] Ensure docs for SuperchainERC20 developers explain the need for deterministic deployment and how to achieve it (FM3, FM4)
- [ ]  Ensure the support team is aware of these failure modes and prepared to respond.

## Audit Requirements

An audit of interop contracts, including the `SuperchainERC20` contract, is expected. However, developers may use `SuperchainERC20` before the interop fork is live.
