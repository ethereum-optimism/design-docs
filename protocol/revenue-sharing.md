# Revenue Sharing

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Mark Tyneway_                                      |
| Created at         | _2025-07-28_                                       |
| Initial Reviewers  | _Skeletor, Aghus, Joxes_                 |
| Need Approval From | _Skeletor, Aghus, Joxes, Pierce_                                    |
| Status             | _Draft_ |

## Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->
Enabling both the Superchain and OP Stack chain operators to manage their revenue in an easy to configure and transparent way will help to enable sustainable onchain businesses. The goals of this design doc are to enshrine fee split contracts as predeploys so that any existing or future OP Stack chain can easily benefit from battle tested fee split logic.
<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->
A new predeploy (`FeeSplitter`) is introduced that is responsible for computing the revenue sharing. Both Base and Uniswap built this contract with slightly different implementation details, we will be introducing another version of it with the key difference of it being configurable by the L2 `ProxyAdmin.owner()`. This ensures that consistent admin roles are applied across chains.

This design doc is specific to the fee splitting logic and does not cover any logic around where those fees go after they are split.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Revenue collection across the Superchain/OP Stack is not standardized. There is a standard fee split that many chains share with Optimism, but there are multiple implementations of this logic. We want to standardize as much as possible on a single implementation that just works.

### Fees Today

Fees for an OP Stack chain accumulate in predeploys called `FeeVault`. There are a total of 4 `FeeVault` contracts

- `SequencerFeeVault`
- `BaseFeeVault`
- `L1FeeVault`
- `OperatorFeeVault`

Each `FeeVault` has a `RECIPIENT` and a `WithdrawalNetwork` that can be L1 or L2. This enables the `FeeVault` to be able to send its collected fees to any account on L1 or L2. These values can only be changed with a full redeploy of these contracts because they are currently defined as immutable values in the smart contract. This creates a lot of friction when trying to change these values. The fee split predeploy depends on the `RECIPIENT` of each of these contracts to be set to the fee split contract itself on L2.

Both the Base and Uniswap implementations have about the same logic, they are modularized in slightly different ways

- Uniswap: https://github.com/Uniswap/unichain-contracts/blob/main/src/FeeSplitter/FeeSplitter.sol
- Base: https://github.com/base/contracts/blob/main/src/revenue-share/FeeDisburser.sol

### Production Today

### Base

The `FeeDisburser` is deployed to https://basescan.org/address/0x09C7bAD99688a55a2e83644BFAed09e62bDcCcBA. The `OPTIMISM_WALLET` is a Gnosis safe at https://basescan.org/address/0x9c3631dDE5c8316bE5B7554B0CcD2631C15a9A05.

### Unichain

The `FeeSplitter` is deployed to https://uniscan.xyz/address/0x4300c0D3c0d3c0d3c0d3c0d3C0D3c0d3c0d30001. The `OPTIMISM_WALLET` is an `L1FeeSplitter` deployed to https://uniscan.xyz/address/0x4300C0D3C0D3C0D3C0d3C0d3c0d3C0d3C0d30002. The `L1FeeSplitter` withdraws funds to [https://etherscan.io/address/0xa3d596EAfaB6B13Ab18D40FaE1A962700C84ADEa](https://etherscan.io/address/0xa3d596EAfaB6B13Ab18D40FaE1A962700C84ADEa#code).

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->
The complexity of this design doc comes with the deployment and upgrades of predeploys while maintaining backwards compatibility. Given that there are already 2 different implementations in production without a standardized flow of funds after the fee split and the open source nature of the OP Stack, we will opt for an opt in mechanism for adopting the fee split predeploys. This ensures that both Base and Unichain are not automatically migrated to the new fee split predeploy given that their fee split logic already works and it is high risk to automate such an upgrade given that they both use different implementations of the fee split.

### `FeeDisburser` vs `FeeSplitter`

The `FeeDisburser` by Base and the `FeeSplitter` by Uniswap are both very similar. The main difference is how they are configured. They both have an `OPTIMISM_WALLET` that is a configured immutable. On Base, the `OPTIMISM_WALLET` is a Safe while on Unichain the `OPTIMISM_WALLET` is a `L1FeeSplitter`, which withdraws the fees to L1.

### Implementation Details

The OP Stack uses Network Upgrade Transactions to manage predeploys. These are deterministic system transactions that are placed into the chain on the block of a network upgrade.

#### `ProxyAdmin`

The `ProxyAdmin` is a L2 predeploy. It is an `Ownable` contract that is at a deterministic address. This means that we can call `IProxyAdmin(Predeploys.PROXY_ADMIN).owner()` from any predeploy and have it correspond to a multisig that manages the chain. No changes to the `ProxyAdmin` itself are required.

#### `FeeSplitter`

All of the `FeeVault` implementations need to have their `RECIPIENT` set to the `FeeSplitter`. This means that all fees that are accumulated in the system will be sent through the `FeeSplitter`.

We take the `FeeDisperser` from Base and make the following changes:

- Add Operator Fee Vault support
- Make `OPTIMISM_WALLET` and `L1_WALLET` modifiable by L2 `ProxyAdmin.owner()`
- Replace `L1_WALLET` with sending to an L2 account similar to the Unichain architecture so that the design is more flexible. The L2 account that Unichain sends to is called `L1Splitter` ([link](https://github.com/Uniswap/unichain-contracts/blob/main/src/FeeSplitter/L1Splitter.sol)) and withdraws to L1. This architecture will make it more flexible for custom gas token chains or architectures where fees want to be collected on the L2 directly.

#### `FeeVault`

We make the following modifications to the `FeeVault` contract to make it more flexible

- Add setters for `RECIPIENT` and `WithdrawalNetwork` that can be modified by `ProxyAdmin.owner()`

The network specific config in the `FeeVault` is defined as:

- `MIN_WITHDRAWAL_AMOUNT`
- `RECIPIENT`
- `WITHDRAWAL_NETWORK`

#### `FeeVaultInitializer`

Since the `FeeVault` contracts have network specific configuration in them, we need to deploy a migrator contract as part of the network upgrade transactions that can propagate the network specific config into the new implementations.
This contract will be responsible for deploying and initializing the new `FeeVault` implementations. It will be able to call the public interface of the existing `FeeVault` implementation and then pass through the same information to its call to initialize. This is helpful because it removes the need to deploy a new `FeeVault` contract each time that the `RECIPIENT` is changed. It may be the case that this cannot atomically operate due to the underlying `Proxy` needing to be called from `address(0)` when changing the implementation, but there is definitely a set of network upgrade transactions that can achieve this task.

#### Upgrade

- On the hardfork block, the following network upgrade transactions are sent:
    - Deploy and initialize `FeeSplitter` but do not set `OPTIMISM_WALLET` and `L1_WALLET`
    - Deploy and initialize a new `FeeVault` for each of the fee vaults using the `FeeVaultInitializer`. The network specific config is unchanged.
- We work with chain operators and send multisig transactions to:
    - Configure the `FeeSplitter`
    - Set the `RECIPIENT` in each `FeeVault` to the `FeeSplitter`

#### Genesis (New Chains)

- We set the proper config in the genesis but allow for overrides to enable anybody to set whatever values they want for the configuration. The defaults would plug right into the superchain revenue sharing.

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The only consideration is the gas usage of the network upgrade transactions but not particularly worried about it given the number of times that we have executed on this type of upgrade.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

Both the Go and Rust derivation pipelines will need to create the correct network upgrade transactions.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

tba

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

There are no protocol level changes that will impact application developers. Chain operators will not have anything automatically migrated, all configuration will be exactly the same.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

### Leave out `FeeVault` changes

It is possible to ship this without any changes to the `FeeVault` contracts. This would reduce the scope of the upgrade, but it would also create more overhead when working with chain operators to migrate to the new fee split contracts. Given that there are so many different versions of the predeploys running in production, we should strive to standardize and upgrade code along the way. This is a perfect opportunity to get everybody on the OP Stack to be using the exact same version of the `FeeVault` predeploys.

### Auto Migrate to new Fee Splitter System

It is possible to force all OP Stack chains to auto migrate to the new fee splitter system through network upgrade transactions. This would remove the need to work with existing chain operators and send multisig transactions that configure and migrate to the new system. Given that there are already fee splitter contracts in production that have slightly different logic, it becomes risky to do this through network upgrade transactions. We could also technically not block on requiring the chains with production fee splits to change anything and we could just onboard the new chains into the new system.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

- Upgrading the flow of fees is risky due to the potential of funds becoming stuck or stolen in some way.
- This does not account for the inaccurate accounting with how L1 fees are charged. The L1 fees are charged at time `t` but then there can be volatility by time `t+1` when the L1 data is actually purchased for batch submission.