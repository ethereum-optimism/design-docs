# Purpose

*This design doc is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

This document presents a solution for migrating `OptimismMintableERC20`, which corresponds to locked liquidity in L1, to the `SuperchainERC20` standard to be interop compatible. 

# Summary

The `OptimismMintableERC20` tokens (also known as legacy tokens) are not compatible with interop. Several approaches are possible to make it possible. In this document, we identify the migration process's potential challenges and suggest a solution called Convert. 

The Convert method will take advantage of the `L2StandardBridge` mint/burn rights over the legacy representations to allow easy conversion between the `OptimismMintableERC20` and the corresponding `SuperchainERC20`.

# Problem Statement

The current design of the `StandardBridge` enables the transfer of standard ERC20 tokens between L1 and each L2 using a lock and mint mechanism. Token representations aren't fungible across different OP chains. We would like to make current liquidity adopt the new `SuperchainERC20` standard to be Interop compatible.

# Possible approaches

### L1 migration

One possible approach is to create a new `SharedStandardBridge` from scratch, which would interact with each `CrossDomainMessenger` and share all the liquidity. This could be implemented as a new contract for users to migrate their liquidity or by forcing a migration through an unconventional process.

However, a manual migration could take months, and a forced migration would introduce significant liability. For these reasons, we advise against this path.

### L2 migration

Another approach is to make changes only on the L2 side to make the current `OptimismMintableERC20` tokens interoperable without many protocol-level changes. This approach is simpler and safer than L1 migration. 

There are several ways to achieve this result:

- **Wrap:** Introduce a new contract that wraps one or multiple legacy tokens into a `SuperchainERC20`.
- **Convert:** Use the burn/mint rights of the `L2StandardBridge` and allow users to convert back and forth between the `OptimismMintableERC20` and the corresponding `SuperchainERC20`.
- **Mirror:** Create a `SuperchainERC20` representation that behaves (with some exceptions) like the legacy "mirrored" token. They will act as secondary entry points to each other.

**Tradeoffs**

- The Wrap approach depends on locked liquidity. This can generate some friction for withdrawals, as `SuperchainERC20`s will move across chains while liquidity remains locked. 
In contrast, the Convert approach uses mint and burn rights for flexible liquidity exchange between standards.
- Convert and Wrap require user migration to be interop compatible. On the other hand, Mirror needs no action and will work from moment zero.
- The Mirror approach might bring security implications at the application layer, as already happened with double entry point designs. Wrap and Convert are very well-known designs that should not alter the existing applications.


> ✅ Of all three methods, the Convert solution represents the cleanest solution to transform existing liquidity into `SuperchainERC20`. This is because of the balance between UX and safety.


### Isolated vs Unified

The most significant design decision is whether to have a single `SuperchainERC20` that accepts multiple `OptimismMintableERC20`s of the same L1 token or to maintain one `SuperchainERC20` per `OptimismMintableERC20`.

Even though the Isolated approach is simpler, we believe fragmentation is a substantial enough issue to make the Unified approach worth the complexities. The migration process from the Isolated approach might be very similar to a regular liquidity migration, which we aim to avoid. For a more detailed discussion, please refer to the Appendix.

# Proposed Solution: Unified Convert

## Implementation

The Convert Method will require modifying the `L2StandardBridge` to include the following:

- `convertToSuper(address _legacyAddr, address _superAddr, uint256 _amount)`:  converts **`_amount` **legacy tokens with address `_legacyAddr` to the same `_amount` of `SuperchainERC20` tokens in address `_superAddr`.
- `convertFromSuper(address _legacyAddr, address superAddr, uint256 _amount)`:  converts `_amount` `SuperchainERC20` with address `_superAddr` to the same `_amount` of legacy tokens in address `_legacyAddr`.

These functions will:

1. Check that the input and output addresses are valid.
2. `burn` the input address balance from the caller. 
3. `mint` the output address balance to the caller.

An example implementation that depends on the `isValidPair()` access control function looks like this:

```solidity
function convertToSuper(address _legacyAddr, address _superAddr, uint256 _amount) public {
  require(isValidPair(_legacyAddr, _superAddr), "Invalid address pair");

  IERC20(_legacyAddr).burn(msg.sender, _amount);
  IERC20(_superAddr).mint(msg.sender, _amount);
}

function convertFromSuper(address _legacyAddr, address _superAddr, uint256 _amount) public {
  require(isValidPair(_legacyAddr, _superAddr), "Invalid address pair");

  IERC20(_superAddr).burn(msg.sender, _amount);
  IERC20(_legacyAddr).mint(msg.sender, _amount);
}
```

The corresponding `SuperchainERC20` will need to allow the `L2StandardBridge` to mint and burn tokens.


>💡 Note:
We considered placing the `convert()` function directly in the `SuperchainERC20`, similar to how WETH manages deposits and withdrawals. However, we believe including them in the `L2StandardBridge` makes more sense. This is because:
>- It makes the token leaner. Users will interact with `convert()` only a few times and in a context similar to deposit and withdraw.
>- The `SuperchainERC20` representation for migrated liquidity will be as close as possible to newly deployed `SuperchainERC20`s.
>- The `L2StandardBridge` needs to be involved because of burn and mint. Having the functionality in the token would have added additional external calls.


### Access Control

To check input addresses are valid, we will need to:

- Check `SuperchainERC20` is valid: This depends on how the Factory is built.
    - Store addresses: The simplest method is to store in the Factory a mapping that tracks the address of the corresponding `SuperchainERC20` for each L1 token (and other factors, such as decimals).
    - Deterministic address: The Factory address is known, and so is the L1 token address, so the `SuperchainERC20` address can potentially be predicted. This heavily depends on the final choice of salt.
- **Check `OptimismMintableERC20` is valid**: This involves verifying that the address was deployed from a valid factory. It is still undecided which Factories should be whitelisted. Possible approaches include:
    - **Use a Registry**: An entity can update the registry at inception, and whitelisted factories can update it upon deployment. The Convert function will check mappings in this registry to allow a pair.
    - **Use deterministic address**: This is only possible for some factories.
        - The [new factory](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/packages/contracts-bedrock/src/universal/OptimismMintableERC20Factory.sol) uses CREATE2 and will have a single representation. Deterministic address computation can verify the validity of the `OptimismMintableERC20`.
        - The [old factory](https://github.com/ethereum-optimism/optimism-legacy/blob/develop/packages/contracts-bedrock/contracts/universal/OptimismMintableERC20Factory.sol) uses CREATE, meaning that the only possible approach is to iterate through the potential nonces. The maximum used nonce is not available in the contract, so the best approach would be to have some kind of registry (single-time update).
- **Check `SuperchainERC20` address corresponds to the correct L1 address**: This is verified during the same validity check of the `SuperchainERC20`.

A possible implementation would look like this:

```solidity

ISuperchainERC20Factory public superFactory = ISuperchainERC20Factory(factoryAddr);

function isValidPair(address _legacyAddr, address _superAddr) internal view returns (bool) {
  IOptimismMintableERC20 legacyToken = IOptimismMintableERC20(_legacyAddr);
  address l1address = legacyToken.l1Token();
  if (!registry.legacyDeployments(l1address)) {
      return false;
  }
  return superFactory.deployment(l1address) == _superAddr;
}
```

### Flows

Conversion flows are directionally opposite but follow the same method. The entire process is completed in a single transaction.

**1. Conversion Legacy Token → SuperchainERC20**

1. User call `convertToSuper` with `_legacyAddr`, `_superAddr`, and `_amount` as parameters
2. `StandardBridge` performs conversion checks utilizing `isValidPair`
If `isValidPair` is true, conversion is executed, else is reverted.
3. `StandardBridge` burn the `_legacyAddr`
4. `StandardBridge` mint the `_superAddr`
5. `msg.sender` receives the superc20 balance.

**2. Conversion SuperchainERC20 → Legacy Token**

1. User call `convertFromSuper` with `_legacyAddr`, `_superAddr`, and `_amount` as parameters
2. `StandardBridge` performs conversion checks utilizing `isValidPair`
If `isValidPair` is true, conversion is executed, else is reverted.
3. `StandardBridge` burn the `_superAddr`
4. `StandardBridge` mint the `_legacyAddr`
5. `msg.sender` receives the legacy balance.

For both cases, `isValidPair` will follow the verification method used, which may vary, as described in Access Control.

# Risks and Uncertainties

### Liquidity paths

In its current version, the `L2StandardBridge` will not know beforehand if the L1 side has liquidity in its pair. This implies that users can drain one path by converting, causing others to get stuck if they are not careful (burn and revert). Ignoring this and expecting the user to figure out the path is possible, but it can lead to poor UX.

It is important to note this issue gets also introduced by Interop tokens, as the `deposit` mapping has no information on the cross-chain movements of `SuperchainERC20`s. Unified will only make this issue more common.

### Ownership and Fragmentation

Not all the bridged tokens vía `StandardBridge` are identically implemented. We usually call legacy tokens any token that utilizes a valid interface and then works with `StandardBridge`, but technically, there are different implementations, for example:

- `L2StandardToken`
- `OptimismMintableERC20`
- `OptimismMintablePermitERC20`
- `UpgradableOptimismMintableERC20`
- Others (tied to the token)

Similarly, when it comes to deployments, it is possible to identify two categories:

- Utilizing an *official* Factory: Found as a predeploy, whose version may differ in each OP Chain.
- Employing other deployment paths: Such as utilizing third-party factories or direct deployments.

Grouping different existing token implementations can be extremely risky. Access controls should be limited to accept only those tokens that follow certain criteria, such as::

- Be immutable
- Implement a valid interface
- Deployed with supply 0
- Has the correct `l2bridge` address stored
- Doesn't implement fancy functions (e.g. additional mint rights, ability to change `l1token`, `l2bridge`, etc.)

It's known that trusted factories deploy tokens that meet all the expected criteria, and there are ways to verify if a token has been deployed via a factory. However, things become complex with tokens created by direct deployment or non-trusted factories, making it difficult to safely verify on-chain if they follow the criteria. A whitelisted token list could be a solution to address this issue.

## Appendix: Isolated vs Unified

Below, we list the pros and cons of using a Unified approach over an Isolated approach:

Pros:

- Reduced fragmentation among representations.
- Simplify `SuperchainERC20` deployment and maintenance over chains.
- Simplify same address deployment: With the Unified approach, using the L1 address as salt for deployment makes sense. This would separate address predictability from the L2 address, which varies across different chains. With the Isolated approach, addressing different L2 addresses with specific modifications to the `SuperchainERC20` implementation would be necessary.

Cons:

- Complex Access Control: The `L2StandardBridge` must perform more checks before converting between tokens. While we know how to manage access control for the Unified approach, it becomes complicated since `OptimismMintableERC20` tokens were created using different methods (Factory, non-factory) and may vary in implementations (upgradability, decimals, etc.).
- Liquidity paths on L1 withdrawals: The unified approach will allow users to switch between L2 representations freely. The invariant of `totalSupply` in L2 equalling locked liquidity in the path (in each `deposit` mapping) was already broken by interop, but this breaks it as well on its own. There might be more reverts on withdrawals.
- Ownership and Fragmentation: Deciding which representations are included in the `SuperchainERC20` and which are not can be challenging. Differentiating by factory could fragment important representations.