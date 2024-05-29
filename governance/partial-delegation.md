# Purpose

This design document aims to provide an overview and explanation of the partial delegation feature to be implemented in the [GovToken](https://github.com/voteagora/governor/blob/main/src/lib/OptimismToken.sol#L1869). The purpose is to understand how it will enable flexible and granular delegation of voting power among multiple delegatees, allowing for fractional delegation.

# Summary

The proposed feature will introduce a partial delegation mechanism that allows accounts to distribute their voting power among multiple delegatees in a fractional way. Compatible with any token contract with voting units that correspond to token units (like `ERC20Votes`), this mechanism will maintain checkpoints to track the changes in delegated voting power over time, allowing accurate retrieval of historical voting power. The goal is to increase participation, decrease the risk of Sybil attacks, align users' expectations, and improve UX (especially for large delegators).

# Problem Statement + Context

With the current GovToken, accounts can only delegate their entire voting power to a single delegatee. This limitation restricts the ability of accounts to distribute their voting rights among multiple parties or to allocate different proportions of their voting power to different delegatees. The lack of partial delegation can lead to centralization of voting power and reduced flexibility for account holders. Agora developed [Alligator](https://github.com/voteagora/governor/blob/main/src/alligator/AlligatorOP_V5.sol), a periphery liquid delegator contract, to fix this problem and support advanced delegations, including partial delegation. However, using the Alligator requires users to interact with multiple contracts and adds complexity to data reporting/aggregation and future interoperability plans.

# Alternatives Considered

1. **Partial Delegation with Alligator (current)**: Basic delegation is handled by the GovToken, while partial delegation is handled by the Alligator.
2. **Partial Delegation in Token**: Basic and partial delegation are handled by the GovToken.

# Proposed Solution

## Rationale

The proposed solution is to implement partial delegation directly in the token contract (option 2) rather than using a separate contract like Alligator (option 1). Incorporating partial delegation into the token provides a seamless user experience by allowing users to interact with a single contract for both basic and advanced delegation features. This eliminates the need to navigate multiple contracts, reducing complexity and improving usability. With all delegation-related information consolidated within a single contract, it becomes easier to minimize the domain knowledge needed to track and analyze delegation data, enabling better visibility and transparency for external parties. Moreover, by including partial delegation in the token, the deployment of Alligator is only required for advanced delegations (like sub-delegations and rulesets). This reduces the overall deployment complexity and minimizes the number of contracts that need to be managed and maintained. The trade-offs of incorporating partial delegation come in increased token complexity and potential gas usage increases for certain contract calls.

When considering the use of percentages versus fixed token amounts for partial delegations, percentages emerge as the superior choice due to their flexibility, scalability, and positive impact on user experience. With percentages, delegators can specify the proportion of their voting power they want to allocate to each delegatee, regardless of the absolute number of tokens they hold. This allows for more granular and precise distribution of voting power. Using percentages ensures that the delegated voting power remains proportional to the delegator's total voting power, even if the delegator's token balance changes over time. For example, if a delegator has 100 tokens and delegates 50% to one delegatee and 30% to another, the delegated voting power will automatically adjust if the delegator's balance increases or decreases, maintaining the intended proportions. In contrast, using fixed token amounts would require the delegator to manually update their delegations whenever their token balance changes to maintain the desired distribution. Additionally, percentages provide a more intuitive and user-friendly way for delegators to express their delegation preferences, as they can think in terms of relative proportions rather than absolute token amounts. The use of percentages may result in remainders of voting power due to rounding or precision limitations. However, this can be effectively managed by distributing any remaining voting power equally among the delegatees or to one specific delegatee.

## Implementation

The implementation of the partial delegation mechanism will involve developing a base abstract contract compatible with any token contract that supports voting. This base contract will define the core components and functionality required for partial delegation. It will include a `PartialDelegation` struct to represent a partial delegation, containing the delegatee address and the numerator of the fraction of voting power delegated. A `_delegatees` mapping will store an array of `PartialDelegation` for each account, allowing multiple partial delegations per account. The contract will provide a `delegate` function, the entry point that will enable an account to delegate its voting power to one or more delegatees by providing an array of `PartialDelegation`. The logic will retrieve any previous delegations of the delegator and then calculate the weight distribution of both the previous and new delegations. Afterwards, the logic will collate the changes between the old and new delegations to create checkpoints. This will involve iterating through the old and new delegations simultaneously and comparing the delegatees. Adjustments will be combined for the same delegatee, amounts will be subtracted for removed delegatees, and amounts will be added for new delegatees. The system will also handle the remainder votes for the delegatees involved in the delegation changes. For each non-zero delegation adjustment, the changes will be pushed to the delegatee's checkpoints, and a corresponding event will be emitted. Finally, the storage will be updated by replacing the old delegatees with the new ones, ensuring proper ordering and uniqueness of delegatees, and delegation events will be emitted to reflect the changes.

## Invariants

The implementation should maintain the following invariants:

1. **Sum of numerators**: When delegating voting power, the sum of the numerators provided in the `PartialDelegation` array must not exceed the fixed denominator. This invariant ensures that the total delegated voting power does not exceed 100% of an account's available voting power.
2. **Delegatee uniqueness**: Within an account's `PartialDelegation` array, each delegatee address should be unique. This invariant prevents duplicate delegations to the same delegatee and ensures the integrity of the delegation relationships.
3. **Checkpoint consistency**: The checkpoints maintained for each delegatee should accurately reflect the changes in their delegated voting power over time. This invariant ensures that the historical voting power can be reliably retrieved at any given point.
4. **Delegation adjustment**: When a delegation is made or cancelled, the contract should correctly calculate and apply the necessary adjustments to the delegatees' voting power. This invariant maintains the accuracy and consistency of the delegated voting power across all affected delegatees.
5. **Votable supply**: The total votable supply should always equal the sum of all delegated voting power across all delegatees. This invariant ensures that the votable supply accurately represents the total available voting power in the system.
6. **Non-negative voting power**: The delegated voting power of each delegatee should never become negative. This invariant protects against any unintended or malicious attempts to undermine the voting system.

# Risks & Uncertainties

1. This introduces additional complexity compared to the out-of-token approach. It requires careful handling of delegation adjustments and checkpoint management.
2. The creation of checkpoints for each delegatee and the calculation of delegation adjustments would increase gas costs for general token operations.