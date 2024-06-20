# Liquidity Migration

## Purpose

_The document presented below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899)._

This document proposes a solution for migrating the liquidity of tokens locked in L1, deployed through the current and legacy `OptimismMintableERC20` Factories in L2 (also known as _legacy tokens_), to make them fungible and compatible with Superchain interoperability.

## Summary

We must deploy new token representations because the current L2 tokens can't be upgraded to be compatible with Superchain interoperability. The main goal is to minimize any issues with liquidity migration.

In this document, we will explore three approaches to create a representation that complies with `ISuperchainERC20`: Wrap, Convert, and Mirror. Among these, Mirror seems to be the best option as it doesn't force users to migrate to new token contracts.

## Problem Statement + Context

The current design of the `StandardBridge` allows for the transfer of standard ERC20 tokens between L1 and each L2 using a lock and mint mechanism. Token representations aren't fungible across different OP chains. We need a way to allow current liquidity to adopt the new `SuperchainERC20` standard to be Interop compatible.

One possible approach is to create a new `SharedStandardBridge` from scratch. This would involve building a single contract connected to each `CrossDomainMessenger` of the Superchain, consolidating all liquidity. It could be implemented as a new contract for users to migrate their liquidity, or via a forced migration on L1 to the new contract. However, a manual migration could take months and a forced migration would introduce significant liability.

Another approach is to make changes only on the L2 side to make the current `OptimismMintableERC20` tokens interoperable without many protocol-level changes. This could involve using the burn/mint permissions of the `L2StandardBridge` and introducing new token contracts to work alongside the existing legacy tokens. While this is not the only way to achieve interoperability, we believe it is the simplest, as the `L2StandardBridge` is upgradable and `OptimismMintableERC20` tokens are not.

## Alternatives Considered

We have considered multiple approaches:

### Wrap

Introduce a new contract that wraps one or multiple legacy tokens into an `ISuperchainERC20`.

### Convert

Create a new `ISuperchainERC20` extension for the legacy tokens that can be exchanged with the legacy representation using the burn and mint rights of `L2StandardBridge` .

### Mirror

Create an `ISuperchainERC20` representation that behaves (with some exceptions) like the legacy "mirrored" token while allowing compatibility with Interop. Users will not realize that these are two distinct tokens, as the two contracts (legacy and Mirror) will act as secondary entry points to each other.

## Tradeoffs and Comments

- The Convert and Wrap approaches require users to migrate to the new representation to support interoperability, while mirror representation requires no additional action from the user.
- The Mirror approach could lead to some UX confusion, especially in displaying double balances.
- For Wrap and Convert `SuperchainERC20` should only work with trusted `OptimismMintableERC20` tokens, without hardcoding legacy token addresses into it. For that, we would need to upgrade the legacy `OptimismMintableERC20Factory` to keep track of legacy tokens for each L1 token. However, verifying these legacy tokens, especially those from unofficial factories, is complex and poses security challenges. The Mirror representation does not face this issue.
- When accepting multiple legacy tokens for the same interoperability representation, the Convert method is a bit cleaner than Wrap. This is because Wrap locks liquidity and differentiates each valid source, whereas Convert doesn't. Since legacy tokens are burned and minted, convert doesn't lock liquidity. Mirror can, in theory, support multiple representations, but we suggest sticking to a single one.

### **Considerations**

- Withdrawals to L1 will remain problematic when introducing interoperability without making changes to the `L1StandardBridge`. Users might attempt to withdraw an amount greater than what the `L1StandardBridge` can handle. In that case, several options can be considered:
  - We can modify `L2StandardBridge` to conduct liquidity checks by comparing the total amount deposited with the total amount withdrawn and canceling the `withdrawal` if the amount is exceeded.
  - Bridge UIs could monitor liquidity availability in both `L1StandardBridge` and choose the best path.

## Proposed Solution: Mirror design

The Mirror Design is the cleanest solution compared to the Wrap and Convert approaches, as it does not require to perform liquidity migrations. From the point of view of various actors, each will notice the following:

- EOAs: No action is required. Users can continue using the legacy or mirror versions for typical actions. When they want to use new `SuperchainERC20` functions, they should interact with the mirror contract.
- Wallets and explorers: Double balances may appear.
- dApps: If necessary, we will provide advance notice.

The specifications of this solution will be outlined in the Mirror Standard Design Document.
