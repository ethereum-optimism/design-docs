# Purpose

This document proposes a solution for implementing the [bridged USDC standard](https://www.circle.com/blog/bridged-usdc-standard) that can be easily upgraded to native USDC. This solution accomplishes the Circle requirements and accommodates OP Stack specifications to allow Chain Operators to perform an easy and secure migration in agreement with Circle.

# Summary

OpUSDC is a modular and efficient reference implementation that extends USDC across the Optimism ecosystem and OP Stack implementations. By leveraging the existing message-passing system and dedicated adapters, USDC can be deployed from day one and seamlessly migrate to native USDC in the future.

This implementation of OpUSDC for the OP Stack allows for:

- Deploy and set up OpUSDC with a single transaction.
- Accomplish with the Bridged USDC standard.
- Leverage from the OP Stack security model used.
- Enabling a chain operator to coordinate with Circle to upgrade into native USDC.

In the end, the goal is to provide a secure pathway for Circle's participation in any OP Chain ecosystem, which favors [Cross-Chain Transfer Protocol (CCTP)](https://developers.circle.com/stablecoins/docs/cctp-getting-started) integration and [other benefits](https://www.circle.com/blog/bridged-usdc-standard) once the migration is completed.

# Problem Statement + Context

USDC is one of the most widely used assets across the crypto ecosystem, often utilized for bridging when new chains emerge before any action from Circle. This situation presents recurring challenges when bridged USDC gains significant market share, but native USDC is preferred by the ecosystem, leading to fragmentation between multiple representations of USDC.

To address this pain point, Circle has rolled out the [Bridged USDC Standard](https://www.circle.com/blog/bridged-usdc-standard) to ensure that chains can easily migrate into the Native USDC version of the token in the future and prevent the fragmentation problem.

However, the Bridged USDC Standard as defined does not work with the current OP Stack `StandardBridge` specifications, causing confusion from chain operators on how to conveniently host USDC on their chain. Circle’s primary goal is to enable chain developers to easily deploy the Bridged USDC Standard. 

To address the issues faced by chain operators and align with Circle's motivations, OpUSDC is proposed as a reference implementation for OP Stack chain operators. This allows them to comply with the Bridged USDC standard and potentially access native USDC.

# Alternatives Considered

**Do nothing**

The `StandardBridge` contracts have been implemented since the chain's inception, allowing the use of any common ERC20 token, such as USDC. As the number of OP Chains within the Superchain ecosystem grows, a problematic scenario will appear where an OP Chain might host different implementations of the same asset: *legacy* bridged USDC, new bridged USDC that accomplish with the Superc20, 3rd party bridged USDC, and native USDC. This fragmentation would significantly complicate the migration to native USDC and negatively impact the user experience.

**Dedicated shared bridge**

To align architecturally with the Superchain design, a proposal was made to enable full fungibility of opUSDC across OP Chains within the same interoperability cluster. This design featured a `SharedL1Adapter` and a factory that allowed trustless deployments of a fungible `bridgedUSDC` implementation on any chain. Additionally, it included access control restrictions for contract upgrades and migrations to native USDC. However, this design required a trusted third party to manage the relevant keys, introducing various layers of liability. These concerns did not meet the issuer's requirements at the time but may be revisited in the future.

# Proposed Solution

The solution involves a [Custom Token Bridge](https://docs.optimism.io/builders/app-developers/bridging/custom-bridge) implementation, which provides desired features such as pausability and the ability to perform migrations. The OP Stack security model secures the proposed design, similar to how `StandardBridge` operates today.

### Implementation

There are four main pieces:

- `BridgedUSDC`: An upgradable contract that implements the Bridged USDC token standard.
- `L1OpUSDCFactory`: A factory on L1 that implements cross-chain deployment logic.
- `L1OpUSDCBridgeAdapter`: An upgradeable bridge contract that locks and secures USDC. It is also responsible for triggering the migration and properly burning the USDC amount once it is completed.
- `L2OpUSDCBridgeAdapter`: An upgradeable bridge contract capable of minting and burning `BridgedUSDC` on the L2 side.

## Flows

Recreation of the flows for each stage of the implementation of this design.

### 1) L1→L2 Deployments

Obtaining the factory, adapters, and bridged token involves the following steps:

1. Anyone deploy the `L1OpUSDCFactory`.
2. The chain operator deploys the Bridged USDC implementation on the L2 of his choice.
3. The chain operator initializes deployment by calling the  `L1OpUSDCFactory` contract, which will:
    1. Deploy the `L1OpUSDCBridgeAdapter` and its proxy on L1.
    2. Call the `CrossDomainMessenger` with the deployment calldata.
    3. Use the calldata to deploy and set up the L2 contracts: `L2OpUSDCBridgeAdapter` and its proxy, and the proxy for the `BridgedUSDC` implementation which will point to the implementation deployed in step 2.

Once everything is set, the adapters are ready to transfer USDC between domains.

### 2) Deposits & withdrawals

For an user to make a deposit, the process remains as simple as follows:

1. Users approve the `L1OpUSDCBridgeAdapter` to spend USDC.
2. Users proceed to deposit USDC by calling the contract.
3. The `L1OpUSDCBridgeAdapter` sends the message to the appointed `CrossDomainMessenger`.
4. The sequencer is digested and included by the sequencer.
5. The `L2OpUSDCBridgeAdapter` mints the specified amount of `bridgedUSDC` to the user.

Similarly, for withdrawals:

1. Users send `bridgedUSDC` to the `L2OpUSDCBridgeAdapter`.
2. The `L2OpUSDCBridgeAdapter` burns the token.
3. The `L2OpUSDCBridgeAdapter` sends the message to the appointed `CrossDomainMessenger`.
4. The message is eventually included and proven on L1.
5. Wait for the challenge period (at least 7 days).
6. The receiving user (or relayers) withdraws the message after the challenge period, which is then forwarded to the `L1OpUSDCBridgeAdapter` that releases the specified amount of USDC to the user.

### 3) Migrating to Native USDC

Deprecation of the current system and upgrade of `BridgedUSDC` to native USDC involves the following steps:

1. Circle communicates to the chain operator their intention to migrate to native USDC.
2. The chain operator initiates `migrationToNative` to set the address for ownership transfer during the upgrade process. This will result in:
    1. Setting the `circle` address in `L1OpUSDCBridgeAdapter`.
    2. Transferring ownership of `BridgedUSDC` to Circle.
    3. Stopping new messages from being sent on both adapters.
    4. Triggering the `setBurnAmount` function to send back to L1. 
3. Once settled, typically after at least seven days, Circle can call `burn` in `L1OpUSDCBridgeAdapter` to effectively burn the USDC supply in L1.
4. Meanwhile, Circle can upgrade `BridgedUSDC` to native USDC.

By doing so, Circle is fully onboarded in the OP Chain, and the old system is deprecated.

### **Additional aspects**

- Anyone can deploy an `L1OpUSDCFactory`, but only one is needed since it is permissionless and immutable. Additionally, it will return the deployment addresses when the `deploy` function is called, and will also emit an event with the addresses.
- `L1OpUSDCBridgeAdapter` and `L2OpUSDCBridgeAdapter` are upgradable.
- Once the migration process starts, it is irreversible.
- In case of a failed transaction on L2 during migration, the `migrateToNative` function will remain open, allowing the executor to retry if needed.
- Liability remains solely on each chain operator during the implementation until the migration process is completed.

# Challenges ahead

1. **Interop-compatibility**: A solution is still needed for cross-chain transfers between `BridgedUSDC` and other USDC versions implemented on other OP Chains, such as native ones. An intent-based design promises to be a viable solution for moving USDC across the Superchain.
2. **Token pathfinding:** OP Chains must collaborate with bridge UI maintainers, third-party operators, and token list maintainers to provide accurate token bridging paths. This is important when transferring from one OP Chain to another where the USDC used is not fungible between them. Users would utilize a combination of CCTP and `OpUSDCBridgeAdapter`, passing through L1 to reach the destination and receive the exact amount bridged. However, L2-L2 transfers can be economically more convenient through a combination of both CCTP and Interop via intents. A well-designed token pathfinding system should ultimately help users save costs and avoid potential mistakes.