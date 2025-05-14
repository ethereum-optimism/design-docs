<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## **Table of Contents** _generated with [DocToc](https://github.com/thlorenz/doctoc)_

- [**Table of Contents** _generated with DocToc_](#table-of-contents-generated-with-doctoc)
- [**SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**](#superchainerc20-standard-only-fma-failure-modes-and-recovery-path-analysis)
- [Introduction](#introduction)
  - [Audit Requirements](#audit-requirements)
  - [Security Considerations](#security-considerations)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: Unauthorized Access to `crosschainMint` \& `crosschainBurn` Functions](#fm1-unauthorized-access-to-crosschainmint--crosschainburn-functions)
  - [FM2: Token Contract Missing in Destination Chain: Relay Fails Until Deployed (But Deployment Is Permissioned)](#fm2-token-contract-missing-in-destination-chain-relay-fails-until-deployed-but-deployment-is-permissioned)
  - [FM3: Token Contract Missing in Destination Chain: Relay Fails Until Deployed (But Deployment Is Permissionless)](#fm3-token-contract-missing-in-destination-chain-relay-fails-until-deployed-but-deployment-is-permissionless)
  - [FM4: Token Contract Missing in Destination Chain: Token Deployer is Lost, or Unable to Deploy to Expected Address](#fm4-token-contract-missing-in-destination-chain-token-deployer-is-lost-or-unable-to-deploy-to-expected-address)
  - [FM5: Compromised Deployment Method](#fm5-compromised-deployment-method)
- [Action Items](#action-items)
- [Warning for Integrators](#warning-for-integrators)
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## **SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**

| Author              | Ng, Joxes, Particle, Gotzen |
| ------------------- | --------------------------- |
| Created at          | 2024-10-02                  |
| Needs Approval From | Mark Tyneway, Matt Solomon  |
| Other Reviewers     | Michael Amadi               |
| Status              | Final                       |

## Introduction

This document is intended to be shared publicly for review and visibility purposes. It covers the introduction of the `SuperchainERC20` standard, including its implementation and interface, the latter extending from `IERC7802`.

Token deployers building on Superchain are encouraged to review the document, as they are the main beneficiaries of this standard. It does not intend to cover contracts such as SuperchainTokenBridge or those involving migrated liquidity.

> 🗂️ The proposed implementation of the standard doesn’t prevent a token issuer from using other token standards, such as xERC20, with the `SuperchainTokenBridge` to mint and burn tokens across an [interoperable set of chains](https://specs.optimism.io/interop/overview.html). More about this is explained in [xERC20-ERC7802 compatilibility](https://defi-wonderland.notion.site/xERC20-ERC7802-compatibility-14c9a4c092c780ca94a8cb81e980d813).

Below are references for this project:

- [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md).
- [Implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainERC20.sol).

### Audit Requirements

An audit of interop contracts, including the `SuperchainERC20` contract, is expected. However, developers may use `SuperchainERC20` before the interop fork is live.

### Security Considerations

Similar to ERC20, SuperchainERC20 implementations should be considered untrusted by default because the `crosschainMint` and `crosschainBurn` methods are not constrained by `IERC7802` or the SuperchainERC20 reference implementation. Consequently, failure modes stemming from malicious SuperchainERC20 implementations are not considered here.

## Failure Modes and Recovery Paths

### FM1: Unauthorized Access to `crosschainMint` & `crosschainBurn` Functions

- **Description:** The `crosschainMint` and `crosschainBurn` functions can only be called by the `SuperchainTokenBridge`, enforced by the check `msg.sender != Predeploys.SUPERCHAIN_TOKEN_BRIDGE`. If the bridge address is badly defined or the modifier bypassed, an entity could mint and burn tokens.
- **Risk Assessment**: Medium.
  - Potential impact: High. All tokens based on this implementation could be potentially at risk.
  - Likelihood: Very Low. `Predeploys.SUPERCHAIN_TOKEN_BRIDGE` is defined via protocol upgrades. The conditional is sufficiently simple and battle-tested to give confidence in the implementation.
- **Mitigation**: Current unit tests perform authorization checks for `crosschainMint` ([test](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/L2/SuperchainERC20.t.sol#L38)) and `crosschainBurn` ([test](https://github.com/ethereum-optimism/optimism/blob/1add88a41b8cdc5c488a18976e3e4fa4b02e7da4/packages/contracts-bedrock/test/L2/SuperchainERC20.t.sol#L77)). There should be tests to ensure that the `SuperchainTokenBridge` predeploy is correctly set during the interop fork upgrade, and is not subject to unexpected changes.
- **Detection**: Ideally, run an off-chain agent to monitor every `crosschainMint` and `crosschainBurn` event in real-time, perhaps it could be resource-intensive given the permissionless usage of `SuperchainERC20` and the expected large number of deployments. If that’s not feasible, rely on user-filed support tickets to flag unusual mint/burn activity, which the TBD team (see corresponding action item) can investigate for suspected unauthorized access.
- **Recovery Path(s)**: Equivocation (i.e. another implementation being set) on the `SuperchainTokenBridge` address would require a protocol upgrade or hard fork to restore the expected code.

### FM2: Token Contract Missing in Destination Chain: Relay Fails Until Deployed (But Deployment Is Permissioned)

- **Description**: The token on the destination chain is not yet deployed, which prevents the cross-chain transfer from finalizing (`relayERC20` fails). However, the token's deployment is permissioned, meaning it is solely up to the deployer to make the token available on the destination chain
- **Risk Assessment**: Medium.
  - Potential impact: Medium. Users may lose access to their tokens temporarily, until the token is deployed.
  - Likelihood: Medium. A deployer may choose not to deploy the token on all chains.
- **Mitigation**: Trusted bridge frontends should prevent users from sending tokens to a chain where the token doesn’t exist. Double-check other trusted sources (such as Superchain Token List) for greater confidence.
- **Detection**: Support tickets filed by users reporting the issue.
- **Recovery Path(s)**: Communicate with the token owner and request deployment of the token on the missing chain.

### FM3: Token Contract Missing in Destination Chain: Relay Fails Until Deployed (But Deployment Is Permissionless)

- **Description**: The token on the destination chain is not yet deployed, which prevents the cross-chain transfer from finalizing (`relayERC20` fails). However, the token is permissionless to deploy, meaning anyone can deploy it.
- **Risk Assessment**: Medium
  - Potential impact: Medium. Users are unable to access their tokens until the token is deployed.
  - Likelihood: Medium. Tokens may not be deployed on all chains.
- **Mitigation**: Trusted bridge frontends should prevent users from sending tokens to any chain where the token does not exist. It is recommended to deploy tokens on every new chain added to the dependency set.
- **Detection**: Support tickets filed by users reporting the issue.
- **Recovery Path(s)**: Deploy the token.

### FM4: Token Contract Missing in Destination Chain: Token Deployer is Lost, or Unable to Deploy to Expected Address

- **Description**: For the `SuperchainTokenBridge` to validate cross-chain mints and burns correctly, the `SuperchainERC20` must appear at one consistent address on each chain when deployed. If a developer fails to deterministically deploy the token at the same address, the bridging logic cannot unify the token references across chains, leading to failed or incorrect cross-chain transfers.
- **Risk Assessment**: Medium
  - Potential impact: High. Without a consistent address, interoperability for the token is effectively disabled and `relayERC20` are likely to fail, causing a lost.
  - Likelihood: Very Low. The interoperable set of chains follows the same opcode behavior and ensures identical availability of deployer contracts, such as `create2Deployer`.
- **Mitigation**: For developers, employ the appropriate deterministic deployment tools, such as the one at `create2Deployer`. For users, refers to FM2 and FM3.
- **Detection**: Deployers should test the deployment method used and verify contract addresses after a mainnet deployment.
- **Recovery Path(s)**: Redeploy token contracts on a new address across chains and migrate userbase if it is needed.

### FM5: Compromised Deployment Method

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

- [x] Resolve all the comments.
- [x] FM1: Implement tests for `SuperchainTokenBridge` predeploy. ([L2Genesis test](https://github.com/defi-wonderland/optimism/blob/54d02df55523c9e1b4b38ed082c12a42087323a0/packages/contracts-bedrock/test/L2/L2Genesis.t.sol#L124), [Predeploys test](https://github.com/defi-wonderland/optimism/blob/54d02df55523c9e1b4b38ed082c12a42087323a0/packages/contracts-bedrock/test/L2/Predeploys.t.sol#L128) amd [SuperchainTokenBridge tests](https://github.com/defi-wonderland/optimism/blob/54d02df55523c9e1b4b38ed082c12a42087323a0/packages/contracts-bedrock/test/L2/SuperchainTokenBridge.t.sol#L21))
- [x] FM1: Run an off-chain script to monitor `crosschainMint` and `crosschainBurn` events for parity and consistency with the expected `msg.sender` (`SuperchainTokenBridge`). (tracked in [Interop Monitoring issue](https://github.com/ethereum-optimism/optimism/issues/15178))
- [x] FM1: Communicate with the team responsible for responding to alerts about the need to define how issues will be raised to the security team.
- [x] FM2, FM3, FM4: Communicate to bridge frontends to ensure they prevent users from sending tokens to a chain where the token doesn’t exist. (tracked in [Superchain Token List v3](https://docs.google.com/document/u/1/d/1eU09Xum8tVQnrnjx9nrhWzAouRXKzjMx0ZGNwkCmOWw))
- [x] FM4, FM5: Ensure that the documentation for SuperchainERC20 developers explains the need for deterministic deployment and how to achieve it, including that constructor args affect the resulting address, requiring consistency across chains. ([SuperchainERC20 Requirements](https://docs.optimism.io/stack/interop/superchain-erc20#requirements))
- [x] FM1, FM2, FM3, FM5: Communicate with the team in charge of responding to user-submitted support tickets about the need to create runbooks and define how issues are escalated to the security team or returned to the user when security involvement is not required. ([Interop tutorials](https://docs.optimism.io/stack/interop/tutorials))

## Warning for Integrators

A key characteristic of a standard `SuperchainERC20` is that it will share the same address across chains. This allows for simpler code with robust access control while enhancing the idea that these tokens truly are one and the same despite living in different chains. From a security standpoint, this predictability in the address come with some considerations for the integrators at the time of transferring tokens into or out of their protocol:

1. The code should not assume the `SuperchainERC20` is already deployed in that chain, but check that it actually is instead.
1. If relying on a library to perform safe transfers, ensure the library checks there's code deployed at the target `SuperchainERC20` address before returning.
1. If not relying on a library directly, but on a contract like `Permit2`, ensure the library it uses for transfers checks there's code at the target `SuperchainERC20` address. `Permit2`, as an example, uses an old `solmate` library that doesn't perform this check.

The reason why this should not be overlooked is that otherwise protocols can wrongly assume that tokens were transferred into or out of the protocol and update important state under this assumption. When a check like the one mentioned above is missing, an attacker can take advantage of the libraries not reverting when the token doesn't yet exist in the chain to, for example, increase their balance in the protocol. This can lead to the attacker stealing funds from users who actually deposited the token after it was actually deployed in that chain.

Although this is a vector of attack that is true for all tokens, the proclivity of `SuperchainERC20s` to share their address make the attack more likely as the attacker can more easily predict the address of a token that a given protocol may support in new chains.

As an example, if `SuperchainERC20_A` is doing well in `Protocol_A` in Optimism, and `Protocol_A` decides to launch on Unichain as well, it's likely it will eventually support `SuperchainERC20_A` in there as well. With this, the attacker can get ahead and increase his balance in Unichain before `SuperchainERC20_A` is deployed there.
