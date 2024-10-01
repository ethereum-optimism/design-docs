# Liquidity Migration

## Purpose

_The document presented below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899)._

This document proposes a solution for migrating the liquidity of tokens locked in L1, deployed through the current and legacy `OptimismMintableERC20` Factories in L2 (also known as _legacy tokens_), to make them fungible and compatible with Superchain interoperability.

## Summary

The current L2 tokens can't be upgraded to be compatible with Superchain interoperability. The main goal is to achieve compatibility whie minimizing liquidity migration issues.

This desgin doc will explore multiple approaches to create a representation that complies with `ISuperchainERC20`. Among these, [Mirror](https://github.com/ethereum-optimism/design-docs/pull/36) seems to be the best option as it doesn't force users to migrate to new token contracts.

## Problem Statement + Context

The current design of the `StandardBridge` allows for the transfer of standard ERC20 tokens between L1 and each L2 using a lock and mint mechanism. `OptimismMintableERC20` representations aren't fungible across different OP chains. These tokens need a way to adopt the new `SuperchainERC20` standard to be Interop compatible.

## Alternatives Considered

### L1 migration

One possible approach is to create a new `SharedStandardBridge` from scratch. This would involve building a single contract connected to each `CrossDomainMessenger` of the Superchain, consolidating all liquidity. It could be implemented as a new contract for users to migrate their liquidity, or via a forced migration on L1 to the new contract. However, a manual migration could take months and a forced migration would introduce significant liability.

### L2 migration

Another approach is to make changes only on the L2 side to make the current `OptimismMintableERC20` tokens interoperable without many protocol-level changes. This could involve using the burn/mint permissions of the `L2StandardBridge` and introducing new token contracts to work alongside the existing legacy tokens.

**Wrap**

Introduce a new contract that wraps one or multiple legacy tokens into an `ISuperchainERC20`.

**Convert**

Create a new `ISuperchainERC20` extension for the legacy tokens that can be exchanged with the legacy representation using the burn and mint rights of `L2StandardBridge` .

**Mirror**

Create an `ISuperchainERC20` representation that behaves (with some exceptions) like the legacy "mirrored" token while allowing compatibility with Interop. Users will not realize that these are two distinct tokens, as the two contracts (legacy and Mirror) will act as secondary entry points to each other.

A more detailed design doc can be found [here](https://github.com/ethereum-optimism/design-docs/pull/36).

## Tradeoffs and conclusion

- The migration flow in L1 is very complex to achieve. An L2 migration seems like a simpler approach.
- The Convert and Wrap approaches require users to migrate to the new representation to support interoperability, while Mirror representation requires no additional action from the user.
- Wrap is the traditional approach to extend functionalities for an asset, as it works in any case. Convert requires a way of minting and burning the token, which legacy representations will have in the `L2StandardBridge`.
- For Wrap and Convert `SuperchainERC20` should only work with trusted `OptimismMintableERC20` tokens, without hardcoding legacy token addresses into it. For that, it is necessary to upgrade the legacy `OptimismMintableERC20Factory` to keep track of legacy tokens for each L1 token. However, verifying these legacy tokens, especially those from unofficial factories, is complex and poses security challenges. The Mirror representation does not face this issue.
- When accepting multiple legacy tokens for the same interoperability representation, the Convert method is a bit cleaner than Wrap. This is because Wrap locks liquidity and differentiates each valid source, whereas Convert doesn't. Since legacy tokens are burned and minted, convert doesn't lock liquidity. Mirror can, in theory, also support multiple representations.
- The Mirror approach could lead to some UX confusion, especially in displaying double balances.

All things considered, the [Mirror Design](https://github.com/ethereum-optimism/design-docs/pull/36) seems to be the cleanest solution, as it does not require to perform any Liquidity Migrations. It is necessary, however, to be very careful with any integration issues. Protocols must be aware of the new design and look for potential bugs before having a new standard in production.

## Additional considerations

- Withdrawals to L1 will remain problematic when introducing interoperability without making changes to the `L1StandardBridge`. Users might attempt to withdraw an amount greater than what the `L1StandardBridge` can handle. In that case, several options can be considered:
  - Modifying the `L2StandardBridge` to conduct liquidity checks by comparing the total amount deposited with the total amount withdrawn and canceling the `withdrawal` if the amount is exceeded.
  - Let the Bridge UIs monitor liquidity availability in both `L1StandardBridge` and choose the best path.
