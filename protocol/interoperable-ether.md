# Purpose

This document exists to align on a design for interoperable `ether`. Without this, user experience would
be greatly hurt and `ether` is commonly used to pay for gas, which is necessary to use a chain.

# Summary

A new `WETH` predeploy is introduced that adds the `SuperchainERC20` functionality. It has a simple migration path from
legacy `WETH` and maintains the ability to have interop with custom gas token chains.

# Problem Statement + Context

The enshrinement of native asset sends between chains was removed as part of the interop design. This greatly reduced
the scope of the project as a native ether send feature could be subject to a double spend attack. Instead, we opt
for using `WETH` to send between chains. The existing `WETH` contract is not proxied meaning that it cannot easily
be upgraded. It also existed from before the times of `SuperchainERC20`, meaning that it has no built in cross chain
sending ability.

With the introduction of custom gas token, the existing `WETH` predeploy now has the semantics of "wrapped native asset",
meaning that the `WETH` predeploy is not guaranteed to be `WETH`. This means one of two things:

- A new `WETH` predeploy is introduced that supports `SuperchainERC20`
- Custom gas token chains can never be interoperable

# Proposed Solution

A new `WETH` predeploy is introduced that supports interfaces `ISuperchainERC20`.
This means that the new `WETH` predeploy allows for cross chain transfers.

The main problem with this solution is the fact that it will break liquidity for wrapped `ether` into
2 different contracts. Many applications already exist today that integrate with the existing predeploy.
This will be annoying, but it should be very simple to migrate from the old `WETH` to the new `WETH`.
Its a simple unwrap and wrap, and technically we could build support directly into the new `WETH` contract
to make the migration extra simple.

We need to come up with a new name for this "superchain wrapped ether" to differentiate it. Not sure
if it should be called `SWETH` or just stick with `WETH`. Without a different name, it will be confusing,
but then it will create more overhead to get people to understand that its portable `WETH`.

Since the usage of custom gas token is legible from within both L1 and L2, this means that the superchain `WETH`
can block deposits of native asset when its a custom gas token chain.

## `IOptimismMintableERC20` Support

It is also possible to add in support for `IOptimismMintableERC20` so that `WETH` can be deposited directly into
this predeploy. This would solve the problem of having `ETH` and native asset liquidity on the L2, since the chain
operator likely needs to sell the earned native asset into `ether` to pay for DA. This could look like the `L1StandardBridge`
automatically converting `ether` into `WETH` and then depositing it such that it mints the superchain `WETH` on L2.
Without this solution, it means there is no easy way to get fungible `ether` on to a custom gas token chain.

# Alternatives Considered

## Upgrade Existing WETH Predeploy

The existing `WETH` predeploy is not proxied, meaning the only way to upgrade
it is with an irregular state transition. This is not a solution that we should
take often, it adds technical debt that must be implemented in every execution
layer client that supports OP Stack.

Following this decision would be inconsistent with previous decision making where
we decided specifically to remove native ether sends from within the protocol
so that custom gas token chains can be interoperable.

# Risks & Uncertainties

- This adds a lot of bridge risk as its a new way to send `ether` between chains. We need to be sure to think in terms of state machines and invariants.
