# Purpose

This design document aims to provide an overview and explanation of a hook-based integration between the [GovernanceToken](https://github.com/ethereum-optimism/specs/blob/main/specs/governance/gov-token.md) and the [GovernanceDelegation](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/gov-delegation.md) contract to enable advanced delegation functionality natively. The purpose is to understand how it will enable relative and absolute partial delegation of voting power among multiple delegates.

# Summary

The proposed feature will introduce a hook-based integration between the `GovernanceToken` and the `GovernanceDelegation` contract. By modifying the `_afterTokenTransfer` function in the `GovernanceToken` to call the `GovernanceDelegation` contract, the `GovernanceDelegation` will be able to consume the hooks and update its delegation and checkpoint mappings accordingly. This approach simplifies the logic and state transitions related to delegations in the `GovernanceDelegation` contract. Additionally, all delegation-related state will be shifted from the `GovernanceToken` to the `GovernanceDelegation`, allowing users and indexers to interact with a single contract for governance-related functionalities.

# Problem Statement + Context

Advanced delegation, which mainly includes relative and absolute partial delegations, is a superset of basic delegation. Partial delegation allows delegators to distribute their voting power among multiple delegates in a fractional manner.

Currently, delegators have to use the `GovernanceDelegation` contract separately from the `GovernanceToken` to access advanced delegation features. This separation of contracts complicates integration for indexers and front-end clients, as they need to track events and interact with both contracts. It also poses challenges for future interoperability plans. To streamline the delegation experience and reduce complexity, a hook-based integration between the `GovernanceToken` and the `GovernanceDelegation` is proposed, consolidating the delegation logic and state into the `GovernanceDelegation`.

# Alternatives Considered

A partial delegation design built into the `GovernanceToken` was previously considered and designed. This approach aimed to incorporate both basic and partial delegation functionalities directly within the `GovernanceToken` contract. However, due to the potential high complexity of the logic and the risks associated with modifying the token contract, this alternative was not preferred. The in-token implementation of partial delegation could introduce significant changes to the `GovernanceToken`'s codebase, increasing the likelihood of unintended consequences and making the contract more difficult to maintain and audit. Therefore, a hook-based implementation that leverages the existing `GovernanceDelegation` contract is deemed a better fit, as it minimizes the modifications required to the `GovernanceToken` while still providing the desired advanced delegation features.

# Proposed Solution

## Rationale

The hook-based integration with the `GovernanceDelegation` is the preferred solution as it simplifies the implementation and reduces the complexity of the `GovernanceToken`. By leveraging the existing `GovernanceDelegation` contract and shifting all delegation-related state to it, the `GovernanceToken` can focus on its core functionalities while the `GovernanceDelegation` handles the delegation logic. This approach minimizes the modifications required to the `GovernanceToken` while still providing the desired advanced delegation features, making it a better fit compared to the in-token implementation. It also improves the user experience by providing a unified contract for all governance-related functionalities.

## Implementation

The implementation involves modifying the `_afterTokenTransfer` function in the `GovernanceToken`, and introducing a function in the `GovernanceDelegation` that can receive the hook call from the `GovernanceToken`. The former function is executed after a token transfer occurs, acting as a hook to call the latter function in the `GovernanceDelegation` to update the delegation and checkpoint mappings based on the token transfer information.

To avoid state fragmentation, all delegation-related state from the `GovernanceToken` will be shifted to the `GovernanceDelegation`'s state. This includes the `delegates`, `checkpoints`, and `numCheckpoints` mappings. The `GovernanceDelegation` will treat the original checkpoints from the `GovernanceToken` as a starting point for its own checkpoints.

Backwards compatibility is maintained by updating the basic delegation functions in the `GovernanceToken` to forward the calls to the `GovernanceDelegation`. As advanced delegation allows for basic delegation, functions like `delegate` and `delegateBySig` in the `GovernanceToken` can be modified to use an advanced delegation rule that mimics basic delegation, and forward it to `GovernanceDelegation`'s delegate function. This approach ensures that the Governor contract implementation will not require any changes.

Last but not least, the `GovernanceDelegation` should be incorporated as a predeploy of the OP stack to avoid manual deployments across the superchain.

## Invariants

The implementation should maintain the following invariants:

1. **Delegation existence**: Every delegation in the `GovernanceDelegation` must be defined as an advanced delegation.
2. **Circular delegation prevention**: The `GovernanceDelegation` contract should prevent circular delegation chains. A delegator `from` cannot delegate to a delegate `to` if `to` is already part of the delegation chain starting from `from`.
3. **Vote weight consistency**: For relative partial delegations, the sum of the basis points should not exceed 100%. For absolute partial delegations, the sum of the votes should not exceed the voting power of the delegator.
4. **Checkpoint consistency**: The checkpoints maintained by the `GovernanceDelegation` contract should accurately reflect the changes in the delegated voting power of each delegate over time, taking into account existing and new advanced delegations.

# Risks & Uncertainties

1. The hook-based integration introduces a dependency between the `GovernanceToken` and the `GovernanceDelegation`, which requires careful coordination during upgrades and modifications.
2. The incorporation of the `GovernanceDelegation` as a predeploy of the OP stack may require additional considerations and coordination.