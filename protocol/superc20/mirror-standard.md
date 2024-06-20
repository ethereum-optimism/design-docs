# Mirror Standard Design Doc

## Purpose

_The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899)._

This document presents the `SuperchainERC20` Mirror Standard, which enables existing tokens deployed through `OptimismMintableERC20` Factory in L2 (also called legacy tokens) to be interoperable without any liquidity migration process.

## Summary

We discussed different approaches to making existing liquidity interoperable in the [Migrated Liquidity Design Doc](https://github.com/ethereum-optimism/design-docs/pull/35). We introduced the Mirror standard and highlighted it as the simplest solution, as it requires no liquidity migration at all.

The Mirror approach deploys a new token contract that follows the `ISuperchainERC20` specifications and will work as a double-entry point for an appointed `OptimismMintableERC20`. We must modify the `L2StandardBridge` to enable the Mirror contract to burn and mint the underlying `OptimismMintableERC20` to implement the Mirror.

## Proposed Design

The proposed design includes a specification on implementing a double-entry point to mirror an existing `OptimismMintableERC20`, as well as approaches to the Factory and `L2StandardBridge`.

### OptimismMintableERC20 proxiable functions

The following functions will be implemented as simple proxies, that is, by having a mirror call to the appointed `OptimismMintableERC20` token:

- `totalSupply()`
- `balanceOf(account)`
- `transfer(recipient, amount)`
- `transferFrom(sender, recipient, amount)`

### Local functions

Approvals functions will be implemented in a conventional way, specific to the contract, and not as mirror calls:

- `approve(spender, amount)`
- `allowance(owner, spender)`

### Code sample

Below, we show what the implementation of the mirror contract would look like.

```solidity
contract Mirror is Superc20 {
  IERC20 public immutable LEGACY_TOKEN;
  IL2StandardBridge public constant L2_STANDARD_BRIDGE = IL2StandardBridge(0x4200000000000000000000000000000000000010);

  constructor(IERC20 _legacyToken) Superc20(_name, _symbol, _decimals) {
    LEGACY_TOKEN = _legacyToken;
    _name = sting.concat("Mirror",_legacyToken.name());
    _symbol = string.concat("m:",_legacyToken.symbol());
    _decimals = _legacyToken.decimals();
  }

  function totalSupply() public view override returns (uint256) {
    return LEGACY_TOKEN.totalSupply();
  }

  function balanceOf(address account) public view override returns (uint256) {
    return LEGACY_TOKEN.balanceOf(account);
  }

  function transfer(address recipient, uint256 amount) public override returns (bool) {
    L2_STANDARD_BRIDGE.mirrorBurn(msg.sender, amount);
    L2_STANDARD_BRIDGE.mirrorMint(recipient, amount);

    emit Transfer(msg.sender, recipient, amount);
    return true;
  }

  function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
    uint256 currentAllowance = allowance(sender, msg.sender);
    require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");

    _approve(sender, msg.sender, currentAllowance - amount);
    L2_STANDARD_BRIDGE.mirrorBurn(sender, amount);
    L2_STANDARD_BRIDGE.mirrorMint(recipient, amount);

    emit Transfer(sender, recipient, amount);
    return true;
  }

  // TODO: discuss possible events
  function _mint(address _to, uint256 _amount) internal virtual override {
    L2_STANDARD_BRIDGE.mirrorMint(_to, _amount);

    emit Transfer(address(0), _to, _amount);
    emit Mint(_to, _amount);
  }

  function _burn(address _from, uint256 _amount) internal virtual override {
    L2_STANDARD_BRIDGE.mirrorBurn(_from, _amount);

    emit Transfer(_from, address(0), _amount);
    emit Burn(_from, _amount);
  }
}
```

### Deployment

Deploying mirror versions will require introducing a new predeploy for`SuperchainERC20` factory. This factory would accomplish the following:

- Designed to enable same-address deployments across OP Chains.
- Address predictability for each token, we should know the corresponding Mirror.
- Simple Factory code.
- It may serve to create both `SuperchainERC20` standard tokens and mirror versions.

### Upgrading `L2StandardBridge`

We must update the `L2StandardBridge` to allow the mirror contract to mint and burn the mirrored legacy token. We can do this by adding a `mirrorMint()` and a `mirrorBurn()` functions that check the mirror is valid for the corresponding legacy token.

## Risks & Uncertainties

1. In this iteration, we have proposed having independent approvals for legacy and mirror token to minimize security risks.
2. Legacy tokens and mirror tokens will handle their own events. This can lead to confusion in transaction logs and make it difficult for users and third-party services to accurately track token movements. Users might see duplicate or redundant events, which can be misinterpreted.
3. We need to ensure that `L2StandardBridge` can only grant burn and mint permissions to valid mirror contracts.
4. Having more than one entry point for a single contract can be a pitfall for developers. They must be aware that the balances can be modified by more than one address.
