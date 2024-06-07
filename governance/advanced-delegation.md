# Purpose

This design document aims to provide an overview and explanation of a hook-based integration between the [GovToken](https://github.com/voteagora/governor/blob/main/src/lib/OptimismToken.sol#L1863-L1901) and the [Alligator](https://github.com/voteagora/governor/blob/main/src/alligator/AlligatorOP_V5.sol) contract to enable advanced delegation functionality natively. The purpose is to understand how it will enable flexible and granular delegation of voting power among multiple delegatees, allowing for fractional delegation and subdelegations.

# Summary

The proposed feature will introduce a hook-based integration between the GovToken and the Alligator contract. By modifying the `_afterTokenTransfer` function in the GovToken to call the Alligator contract, the Alligator will be able to consume the hooks and update its delegation and checkpoint mappings accordingly. This approach simplifies the logic and state transitions related to delegation in the Alligator. Additionally, all delegation-related state will be shifted from the GovToken to the Alligator, allowing users and indexers to interact with a single contract for governance-related functionalities.

# Problem Statement + Context

Advanced delegation, which includes partial delegation and subdelegations, is a superset of basic delegation. Partial delegation allows delegators to distribute their voting power among multiple delegatees in a fractional manner, while subdelegations enable delegatees to further delegate their received voting power to other delegatees based on predefined rules and constraints.

Currently, delegators have to use the Alligator contract separately from the GovToken to access advanced delegation features. This separation of contracts complicates integration for indexers and front-end clients, as they need to track events and interact with both contracts. It also poses challenges for future interoperability plans. To streamline the delegation experience and reduce complexity, a hook-based integration between the GovToken and the Alligator is proposed, consolidating the delegation logic and state into a single contract.

# Alternatives Considered

A partial delegation design built into the GovToken was previously considered and designed. This approach aimed to incorporate both basic and partial delegation functionalities directly within the GovToken contract. However, due to the potential high complexity of the logic and the risks associated with modifying the token contract, this alternative was not preferred. The in-token implementation of partial delegation could introduce significant changes to the GovToken's codebase, increasing the likelihood of unintended consequences and making the contract more difficult to maintain and audit. Therefore, a hook-based implementation that leverages the existing Alligator contract is deemed a better fit, as it minimizes the modifications required to the GovToken while still providing the desired advanced delegation features.

# Proposed Solution

## Rationale

The hook-based integration with the Alligator is the preferred solution as it simplifies the implementation and reduces the complexity of the GovToken. By leveraging the existing Alligator contract and shifting all delegation-related state to it, the GovToken can focus on its core functionalities while the Alligator handles the delegation logic. This approach minimizes the modifications required to the GovToken while still providing the desired advanced delegation features, making it a better fit compared to the in-token implementation. It also improves the user experience by providing a unified contract for all governance-related functionalities.

## Implementation

The implementation involves modifying the `_afterTokenTransfer` function in the GovToken, and introducing the `afterTokenTransfer` function in the Alligator. The former function is executed after a token transfer occurs, acting as a hook to call the latter function in the Alligator to update the delegation and checkpoint mappings based on the token transfer information.

To avoid state fragmentation, all delegation-related state from the GovToken will be shifted to the Alligator's state. This includes the `delegates`, `checkpoints`, and `numCheckpoints` mappings. The Alligator will treat the original checkpoints from the GovToken as a starting point for its own checkpoints.

Backwards compatibility is maintained by updating the basic delegation functions in the GovToken to forward the calls to the Alligator. As advanced delegation allows for basic delegation, functions like `delegate` and `delegateBySig` in the GovToken can be modified to use an advanced delegation rule that mimics basic delegation, and forward it to Alligator's `subdelegate` function. This approach ensures that the Governor contract implementation will not require any changes.

Last but not least, the Alligator should be incorporated as a predeploy of the OP stack to avoid manual deployments across the Superchain.

## Invariants

The implementation should maintain the following invariants:

1. **Delegation existence**: For each delegation relationship from `from` to `to`, there must be a corresponding `SubdelegationRules` entry in the `subdelegations` mapping.
2. **Circular delegation prevention**: The Alligator contract should prevent circular delegation chains. A delegator `from` cannot delegate to a delegatee `to` if `to` is already part of the delegation chain starting from `from`.
3. **Vote weight consistency**: The sum of the votes cast by a delegatee `to` on behalf of a delegator `from` for a given proposal should not exceed the allowance specified in the `SubdelegationRules` for the delegation from `from` to `to`.
4. **Delegated voting power**: The total delegated voting power of a delegatee `to` for a given proposal should be the sum of the allowances from all its delegators, considering the `AllowanceType` (relative or absolute) and the delegators' voting power.
5. **Checkpoint consistency**: The checkpoints maintained by the Alligator contract should accurately reflect the changes in the delegated voting power of each delegatee over time, taking into account the delegations, subdelegations, and any modifications to the delegation rules.
6. **Weight cast tracking**: The `weightCast` value for a given proposal and proxy should always be less than or equal to the total voting power of the proxy at the proposal's snapshot block.
7. **Proposal-specific rules**: The validation of subdelegation rules, such as `notValidBefore`, `notValidAfter`, and `blocksBeforeVoteCloses`, should be strictly enforced based on the current block timestamp and the proposal's deadline.
8. **Custom rule validation**: If a custom rule is specified in the `SubdelegationRules`, it should be properly validated by calling the `validate` function of the `IRule` interface with the correct parameters.

# Risks & Uncertainties

1. The hook-based integration introduces a dependency between the GovToken and the Alligator, which requires careful coordination during upgrades and modifications.
2. The shift of delegation-related state from the GovToken to the Alligator may have implications for existing integrations or tools that rely on the GovToken's delegation mappings.
3. The incorporation of the Alligator as a predeploy of the OP stack may require additional considerations and coordination.