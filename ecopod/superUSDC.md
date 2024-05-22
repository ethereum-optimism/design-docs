# SuperUSDC design doc

# Purpose

> 📑 *This section is also sometimes called “Motivations” or “Goals”. It is fine to remove this section from the final document, but understanding the purpose of the doc when writing is very helpful.*

This document proposes a solution for implementing a bridged USDC design that can be easily upgraded to native USDC. This solution leverages Superchain capabilities, including shared bridges, shared upgrades, and interoperability, providing an all-in-one solution.

# Summary

> 📑 *[Most (if not all) documents should have a summary. While the length will likely be proportional to the length of the full document, the summary should be as succinct as possible.]*


**superUSDC**, is a modular and efficient refence implementation to expand USDC across the Superchain ecosystem. Leveraging the existing message-passing system and dedicated adapters, to ensure seamless access to native and bridged USDC liquidity, across all OP Chains.

This implementation of superUSDC for the OP Stack allows for:

- Utilizing the Bridged USDC standard without any modifications to it.
- Enabling a chain operator to coordinate alongside OP an upgrade to native USDC.
- Enabling a chain operator to upgrade to the superUSDC version of USDC (when available).
- Moving between L1 and L2 via the OP Stack cross-domain messenger.
- Moving between L2 and L2 through liquidity networks and eventually native interoperability (when available).
- Deploy and setup SuperUSDC for new chains with a single transaction.
- Fungibility of Bridged USDC between interop-chains.

The goal is to provide a secure pathway for Circle's participation in any OP Chain ecosystem, facilitate [Cross-Chain Transfer Protocol (CCTP)](https://www.notion.so/CCTP-USDC-d464d146ce244502af760a9b9e4765f4?pvs=21) integration and enable future integration with native Superchain interoperability.

# Problem Statement + Context


> 📑 *Describe the specific problem that the document is seeking to address as well as information needed to understand the problem and design space. If more information is needed on the costs of the problem, this is a good place to that information.*


USDC is one of the most bridged assets across the crypto ecosystem, and USDC is often bridged to new chains prior to any action from Circle. This can create a challenge when Bridged USDC achieves substantial market share, but Native USDC is preferred by the ecosystem, leading to fragmentation between multiple representations of USDC.

To address this pain point, Circle has rolled out the [Bridged USDC Standard](https://www.circle.com/blog/bridged-usdc-standard) to ensure that chains can easily migrate into the Native USDC version of the token in the future and prevent the fragmentation problem.

However, the Bridged USDC Standard as defined does not work with the OP Stack bridge causing confusion from chain operators on how to safely get USDC on their chain. Circle’s primary goal is to enable chain developers to easily deploy the Bridged USDC Standard. 

To address the issues of chain operators and Circle's motivation, superUSDC was developed, a reference implementation for OP Stack chain operators to access native USDC and continue to benefit from the Superchain.

# Alternatives Considered

> 📑 *List out a short summary of each possible solution that was considered. Comparing the effort of each solution*

**Do nothing**

StandardBridge contracts are implemented since the chain's inception, allowing the use of any common ERC20 token, such as USDC. As the number of OP Chains within the Superchain ecosystem grows and the Superc20 standard is introduced, it is likely that USDC will adopt a generalized standard in the absence of a dedicated solution. This could lead to a problematic scenario where an OP Chain might host different implementations of the same asset: *legacy* bridged USDC, new bridged USDC (for example, under Superc20 [WIP]), 3rd party bridged USDC, and native USDC. This fragmentation would significantly complicate the migration to native USDC and negatively impact the user experience.

**Separate bridge contracts for every OP Chain**

Originally proposed for this work, an alternative approach involves each OP Chain separate and independent lockbox/bridge contract instead of coupling to a shared L1 bridge contract with other OP Chains. This method allows each chain to fully own the bridge and token contracts during the initial stages before coordinating the migration to native USDC.

While this approach is simpler and less architecturally complex, it is not future-proof for the Superchain ecosystem. It makes it challenging to achieve important objectives such as liquidity unification and full compatibility with Interop. This alternative may be suitable for individual OP Chains that do not intend to integrate into the Superchain ecosystem.

# Proposed Solution

> 📑 *A high level overview of the proposed solution. When there are multiple alternatives there should be an explanation of why one solution was picked over other solutions. As a rule of thumb, including code snippets (except for defining an external API) is likely too low level.*


Recreation of the flows for each stage of the implementation of this design. Note that each flow is unique for each OP chain, except for contract deployment on L1, which is done once and remains common throughout the Superchain.

### 1) L1→L2 Deployments

Getting the factory, upgrade-manager, bridge-adapter and the implementations of the L2 adapter and token on L1. This is necessary to securely deploy on existing and new chains:

1. Anyone can deploy `SuperUSDCFactory`, `UpgradeManager`, the Proxy Adapter, and the initial implementation for the Adapter on L1. The `UpgradeManager` owns the Proxy Adapter on L1.
2. Anyone set up the Trusted 3rd Party as the owner of the `UpgradeManager`, which can be done at deployment or via a change owner function.
3. The Trusted 3rd Party whitelists `CrossDomainMessenger` addresses in the `UpgradeManager`. There are two possible flows:
    1. Allow users to suggest addresses and let the Trusted 3rd Party approve them.
    2. The Trusted 3rd Party sets addresses directly.
4. Anyone can initialize a deployment to a chain with a whitelisted `CrossDomainMessenger` by calling the `SuperUSDCFactory` contract.
    1. The `SuperUSDCFactory` calls the `CrossDomainMessenger` with the deployment calldata.
    2. The calldata is used to deploy and set up the L2 contracts: Proxy Adapter, Implementation Adapter, Proxy `SuperUSDC`, and Implementation `SuperUSDC`.
    3. The Trusted 3rd Party (same as in step 1) is set as the owner of the four new contracts on L2.

### 2) L1→L2 Upgrades

Using the `UpgradeManager` it is possible to signal new implementations (contract code) for existing contracts to upgrade to. This could happen if Circle adds/changes any standard related to their bridged USDC contract or the OP canonical bridge is upgraded.

1. On L1, anyone can deploy a new proposed implementation of the contract bridges: L1 `SuperUSDCBridgeAdapter` and L2 `SuperUSDCBridgeAdapter`, as well as a new `SuperUSDC` implementation.
2. The Trusted 3rd Party must:
    1. Review the candidate implementations.
    2. If approved, set the `UpgradeManager` to point to the newly deployed implementations.
    3. If not approved, ignore the deployment.
3. Anyone can propagate the upgrades to other OP Chains.
    1. Optionally, for some chains, the propagator address might need to be whitelisted by the Trusted 3rd Party in the `UpgradeManager`.
4. The Sequencer propagates the message in L2 via `CrossDomainMessenger`.
    1. The message deploys the new L1-approved L2 `SuperUSDCBridgeAdapter` and/or `SuperUSDC` implementations.
    2. The message updates the L2 `SuperUSDCBridgeAdapter` and/or `SuperUSDC` proxy contracts to point to the new implementations.

### 3) Migrating to Native USDC

Using the `UpgradeManager` it is possible to signal, after agreeing with Circle, a “circle-controlled-address” that will be in charge of finishing both the role migration on superUSDC and burning the corresponding locked USDCs on L1.

1. Circle communicates to the Trusted 3rd Party their intention to migrate to native USDC on a specific OP Chain.
2. The Trusted 3rd Party initiates the `prepareMigrationToNative()` function in the `UpgradeManager`.
3. Anyone can propagate the migration to L2.
    1. Optionally, for some chains, the propagator address might need to be whitelisted by the Trusted 3rd Party in the `UpgradeManager`.
4. The propagated messages trigger the adapter to whitelist the Circle-controlled address in `SuperUSDC`.
5. The Circle-controlled address calls `setUpNewRoles()` on `SuperUSDC` to obtain ownership.
    1. The adapter will no longer be allowed to mint with L1 messages.
    2. The same transaction triggers a message back to L1.
6. Wait for the challenge period (at least 7 days).
7. The message gets propagated in L1 back to the L1 Adapter, setting the burn amount for the Circle-controlled address to burn the corresponding supply amount.
8. Now, the previously known `SuperUSDC` is considered native USDC and can be bridged via CCTP.

**Notes:**

- It is important to ONLY deploy using the factory on trusted chains, this means the trusted-3rd-party needs to whitelist or signal in any way which are these trusted chains.
- In the event that the chain operator needs to upgrade from SuperUSDC to Native USDC, the Trusted 3rd party would be responsible for initiating the `prepareMigrationToNative()` function in the `UpgradeManager`.
- One key benefit of this design is that it will enable seamless Interoperability to allow all deployed superUSDC tokens to move freely between interoperable L2s. This will only require a contract update (point 2).

# Risks & Uncertainties

> 📑 *An overview of what could go wrong. Also any open questions that need more work to resolve.*

**1) OP Chain leaving the Superchain**

In the edge case where an OP Chain chooses not to be part of the Superchain or unilaterally follows a different chain version than the one approved by governance and supported by other OP Chains, it could negatively impact asset compatibility and security, including superUSDC. This divergence can cause severe problems in the representation of the asset and necessitate a migration process, which remains unclear. Such an event could result in a painful experience for users.

**2) Token path finding**

Currently, token bridging into any OP Chain relies on token lists to manage different existing bridge implementations. This information is maintained off-chain, which presents a risk. With the upcoming Superc20 standard and Interop, operational failures could lead to assets becoming stuck or erratic minting. Ideally, users bridging assets should be protected from these scenarios and routed to the correct representation via a contract registry and on-chain lists.

**Open questions:**

- What happens when there are already deployed Bridged USDC tokens on different OP Chains? How can we merge them into a single "Superchain ERC20" token that shares the same address across all superchains?
    - We could upgrade the Birdged USDC to also use the SuperUSDC as a secondary entry-point. Making both addresses share the same underlying storage.