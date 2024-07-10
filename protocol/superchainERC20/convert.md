# Purpose

_This design doc is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899)._

This document presents a solution for migrating `OptimismMintableERC20`, which corresponds to locked liquidity in L1, to the `SuperchainERC20` standard to be interop compatible.

# Summary

The `OptimismMintableERC20` tokens (also known as legacy tokens) are not compatible with interop. This document identifies the migration process's potential challenges and suggests a solution called Convert.

The Convert method will take advantage of the `L2StandardBridge` mint/burn rights over the legacy representations to allow easy conversion between the `OptimismMintableERC20` and the corresponding `SuperchainERC20`.

# Problem Statement

The current design of the `StandardBridge` enables the transfer of standard ERC20 tokens between L1 and each L2 using a lock and mint mechanism. Token representations aren't fungible across different OP chains. We would like to make current liquidity adopt the new `SuperchainERC20` standard to be Interop compatible.

# Possible approaches

### Liquidity Migration Methods

**L1 migration**

One possible approach is to create a new `SharedStandardBridge` from scratch, which would interact with each `CrossDomainMessenger` and share all the liquidity. This could be implemented as a new contract for users to migrate their liquidity or by forcing a migration through an unconventional process.

However, a manual migration could take months, and a forced migration would introduce significant liability. For these reasons, we advise against this path.

**L2 migration**

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

The Convert method will accept `OptimismMintableERC20`s and convert them into `SuperchainERC20`s. The most significant decision is whether to have a single `SuperchainERC20` that accepts multiple `OptimismMintableERC20`s of the same L1 token (Unified) or to maintain one `SuperchainERC20` per `OptimismMintableERC20` (Isolated).

We prefer simplicity and will therefore focus on the Isolated approach, i.e., a single `SuperchainERC20` per legacy token. For a more detailed discussion, please refer to the Appendix.

# Proposed Solution: Isolated Convert

## Implementation

The Convert Method will require modifying the `L2StandardBridge` to include the following:

- `convertToSuper(address _*legacyAddr, address _superAddr, uint256 _amount)`: converts* `_amount` *legacy tokens with address `_*legacyAddr` to the same `_amount` of `SuperchainERC20` tokens in address `_superAddr`.
- `convertFromSuper(address _*legacyAddr, address superAddr, uint256 _amount)`: converts* `_amount` *`SuperchainERC20` with address `_super*Addr` to the same `_amount` of legacy tokens in address `_legacyAddr`.

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

The corresponding `SuperchainERC20` will need to allow the `L2StandardBridge` to mint and burn tokens. We will discuss the `isValidPair()` function in the [section below](#access-control).

> 💡 Note:
> We considered placing the `convert()` function directly in the `SuperchainERC20`, similar to how WETH manages deposits and withdrawals. However, we believe including them in the `L2StandardBridge` makes more sense. This is because:
>
> - It makes the token leaner. Users will interact with `convert()` only a few times and in a context similar to deposit and withdraw.
> - The `SuperchainERC20` representation for migrated liquidity will be as close as possible to newly deployed `SuperchainERC20`s.
> - The `L2StandardBridge` needs to be involved because of burn and mint. Having the functionality in the token would have added additional external calls.

### Access Control

The contract will need to:

- **Check `OptimismMintableERC20` address is valid**: there is no need to verify anything as any liquidity risk will be isolated.
- **Check `SuperchainERC20` is valid**: There are two ways to implement access control in the Isolated design.
  - **Registry**: If the Factory stores the `SuperchainERC20` address for each legacy address.
    ```solidity
    ISuperchainERC20Factory public superFactory = ISuperchainERC20Factory(factoryAddr);

    function isValidPair(address _legacyAddr, address _superAddr) internal view returns (bool) {
      return superFactory.deployment(_legacyAddr) == _superAddr;
    }
    ```
  - **Deterministic address**: The `SuperchainERC20` factory will use the L2 representation address as a salt, so the address of the `SuperchainERC20` can be deduced from the address of the `OptimismMintableERC20` and the Factory (predeploy).

    ```solidity
    function isValidPair(address _legacyAddr, address _superAddr) internal view returns (bool) {
    bytes32 salt = keccak256(
        abi.encodePacked(_legacyAddr)
        );
    address computedAddress = superFactory.getDeployed(salt));
    return computedAddress == _superADDR
    }
    ```
  We suggest implementing the Registry approach, even if it implies that deployments need to be recorded in the Factory storage, as in the long run, it will be much cheaper for users to convert.
- Check `SuperchainERC20` address correspondence to the correct `OptimismMintableERC20`: This is done automatically in the `SuperchainERC20` validity check.

## Flows

Conversion flows are directionally opposite but follow the same method. The entire process is completed in a single transaction.

**1. Legacy Token → SuperchainERC20**

1. User call `convertToSuper` with `_legacyAddr`, `_superAddr`, and `_amount` as parameters
2. `StandardBridge` performs conversion checks utilizing `isValidPair`
   If `isValidPair` is true, conversion is executed, else is reverted.
3. `StandardBridge` burns the `_legacyAddr` amount
4. `StandardBridge` mint the `_superAddr` amount
5. `msg.sender` receives the superc20 balance.

**2. SuperchainERC20 → Legacy Token**

1. User call `convertFromSuper` with `_legacyAddr`, `_superAddr`, and `_amount` as parameters
2. `StandardBridge` performs conversion checks utilizing `isValidPair`
   If `isValidPair` is true, conversion is executed, else is reverted.
3. `StandardBridge` burns the `_superAddr` amount
4. `StandardBridge` mint the `_legacyAddr` amount
5. `msg.sender` receives the legacy balance.

For both cases, `isValidPair` will follow the verification method used, which may vary, as described in Access Control.

# Risks and Uncertainties

### Pairing tokens with different addresses or implementations

There are many well-adopted tokens that have different addresses and implementations on various chains. Some common examples include:

- Assets that follow different official implementations (e.g. BAL in [OP Mainnet](https://optimistic.etherscan.io/token/0xfe8b128ba8c78aabc59d4c64cee7ff28e9379921#code) vs. [Base](https://basescan.org/token/0x4158734d47fc9692176b5085e0f52ee0da5d47f1#code))
- Assets that are upgradable on one of the chains (e.g. cbETH in [OP Mainnet](https://optimistic.etherscan.io/token/0xaddb6a0412de1ba0f936dcaeb8aaa24578dcf3b2#code) vs. [Base](https://basescan.org/token/0x2ae3f1ec7f1f5012cfeab0185bfc7aa3cf0dec22#code))
- Assets that use uncommon implementations (e.g. CRV in [Fraxtal](https://fraxscan.com/address/0x331B9182088e2A7d6D3Fe4742AbA1fB231aEcc56#code), MBS in [Base](https://basescan.org/token/0x8fbd0648971d56f1f2c35fa075ff5bc75fb0e39d#code))

How should Convert handle this scenario?

It is possible to address cases individually and let each token owner decide over their token, but ultimately, any decision will require an update to the `L2StandardBridge`, meaning that operators (chain owners and governors) will have the final say.
An on-chain whitelisted token list could be a good way to manage these scenarios but at the cost of adding a layer of trust and liability, such as validation through a governance process. This also suggests that it might be necessary to implement some form of governance over the `SuperchainERC20`s.

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
  convert.md
