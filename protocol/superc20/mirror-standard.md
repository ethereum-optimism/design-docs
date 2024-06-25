# Mirror Standard Design Doc

## Purpose

_The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899)._

This document presents the `SuperchainERC20` Mirror Standard, which enables existing tokens deployed through `OptimismMintableERC20` Factory in L2 (also called legacy tokens) to be interoperable without any liquidity migration process.

## Summary

The [Migrated Liquidity Design Doc](https://github.com/ethereum-optimism/design-docs/pull/35) already discussed different approaches to making existing `OptimismMintableERC20` liquidity in L2 interoperable. The Mirror standard was highlighted as the simplest solution, as it requires no liquidity migration at all.

The Mirror approach deploys a new token contract that follows the [`ISuperchainERC20` specifications](https://github.com/ethereum-optimism/specs/pull/257) and will work as a double-entry point for an appointed `OptimismMintableERC20`. To implement the Mirror solution, the `L2StandardBridge` should be modified to allow the Mirror contract to burn and mint the underlying `OptimismMintableERC20`.

## Proposed Design

### OptimismMintableERC20 proxiable functions

The following functions will be implemented as simple proxies, that is, by having a mirror call to the appointed `OptimismMintableERC20` token:

- `name()`
- `symbol()`
- `decimals()`
- `totalSupply()`
- `balanceOf(account)`
- `transfer(recipient, amount)`
- `transferFrom(sender, recipient, amount)`

### Local functions

Approvals functions will be implemented in a conventional way, specific to the contract, and not as mirror calls:

- `approve(spender, amount)`
- `allowance(owner, spender)`

If approvals where proxied, it would be very straightforward to achieve a double spending attack:

1. Alice has a legacy token balance of N and gave Bob an approval for M.
2. Bob can call `transferFrom()` in the Mirror contract for $\rm min(M,N)$ to his account. This will not update the allowance on the legacy token.
3. If $N>M$ (Alice still has balance after `L2StandardBridge` burnt $\rm min(M,N)$ tokens), Bob can call `transferFrom()` in legacy an get $\rm min(N-M,M)$ tokens to his account.

For example, if $N>2M$, Bob could get $2M$, i.e. double its allowance.

### Code sample

Below, we show what the implementation of the mirror contract would look like. Notice it follows the `SuperchainERC20` standard, which was presented [here](https://github.com/ethereum-optimism/specs/pull/257).

```solidity
contract Mirror is SuperchainERC20 {
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

- Enable same-address deployments across OP Chains.
- Address predictability: for each `OptimismMintableERC20` token, it should be possible to deduce the corresponding Mirror address.
- [To Discuss] It may serve to create both `SuperchainERC20` standard tokens and mirror versions.

### Upgrading `L2StandardBridge`

The `L2StandardBridge` should be updated to allow the Mirror contract to mint and burn the mirrored legacy token. We can do this by adding a `mirrorMint()` and a `mirrorBurn()` functions that check the mirror is valid for the corresponding legacy token.

For validation, the `L2StandardBridge` can check if the address of the `msg.sender` (the mirror) corresponds to the correct mirror address deployed by the Mirror factory. In the Mirror factory design, each token is coupled with a single Mirror, so this can be done (see [Limited Mirror Factory](https://github.com/ethereum-optimism/design-docs/pull/37)).

Here's an example code snippet:

```solidity
function mirrorMint(address _token, uint256 _amount) external {
	address _mirror = MIRROR_FACTORY.mirrors(_token);
	if (_mirror != msg.sender) revert();
	///...mint logic
}
```

## Risks & Uncertainties

1. Legacy tokens and mirror tokens will handle their own events. This can lead to confusion in transaction logs and make it difficult for users and third-party services to accurately track token movements. Users might see duplicate or redundant events, which can be misinterpreted.
2. It is essential that we have a good method for the `L2StandardBridge` to only grant burn and mint permissions to valid mirror contracts.
3. Having more than one entry point for a single contract can be a pitfall for developers. They must be aware that the balances can be modified by more than one address.
