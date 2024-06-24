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

An important problem to solve is ensuring that there are no liquidity constraints. If a wrap/unwrap mechanism
is used rather than a mint/burn mechanism, it will result in liquidity constraints. Therefore we prefer a 
mint/burn mechanism. We need a solution to the liquidity constraint that specifically happens when a user is
trying to send ether to a remote domain. They have to wrap the ether into weth and then it gets sent between
chains as weth and then we need to unwrap the weth. Its possible that there isn’t enough weth to unwrap on the other side.

We introduce 2 new predeploys

- `SuperchainWETH`
    - Implements `SuperchainERC20`
    - Guaranteed to be wrapped ether rather than wrapped native asset
- `ETHLiquidity`
    - Contains an artificially large pool of `ether` to provide liquidity for cross chain sends
    - We “burn” ether by sending it here and “mint” ether by pulling it out
    - Only `SuperchainWETH` can interact with this contract
    - Only works on non custom gas token chains
    - This is similar to the Scroll bridge where in genesis they mint a ton of ether into the contract where it can only be unlocked via a L1 to L2 deposit
    - Placed in genesis with a balance equal to `type(uint248).max`

The important invariant is that `ether` cannot be withdrawn from the `SuperchainWETH` contract by entities that do not own it.
This would result in inflating the supply of ether.

### SuperchainWETH

- Users can send ether between chains as this contract will handle the wrapping and unwrapping, this only works when
  the source and destination are not custom gas token
- Users can send ether from a non custom gas token chain to a custom gas token chain and it will end up as weth
- Users can send weth from a custom gas token chain to a non custom gas token chain and it will end up as ether
- Users can send weth between custom gas token chains

```solidity
contract SuperchainWETH is WETH98, ISuperchainERC20 {
  L1Block internal l1Block = L1Block(Predeploys.L1_BLOCK);
  L2ToL2CrossDomainMessenger messenger = L2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSSDOMAIN_MESSENGER);
  ETHLiquidity liquidity = ETHLiquidity(Predeploys.ETH_LIQUIDITY);

  function deposit() public payable override {
    if (l1block.isCustomGasToken()) revert IsCustomGasToken();
    super.deposit();
  }

  function withdraw(uint256 wad) public override {
    if (l1block.isCustomGasToken()) revert IsCustomGasToken();
    super.withdraw(wad);
  }

  function sendETHTo(address to, uint256 chainId) public payable {
    if (l1block.isCustomGasToken()) revert IsCustomGasToken();

    liquidity.burn{ value: msg.value }();

    sendMessage({
      _destination: chainId,
      _message: abi.encodeCall(SuperchainWETH.relayETH, (to, msg.value))
    });
  }

  function sendERC20(uint256 wad, uint256 chainId) {
    sendERC20To(msg.sender, wad, chainID);
  }

  function sendERC20To(address to, uint256 wad, uint256 chainId) external {
    require(balanceOf[msg.sender] >= wad);
    balanceOf[msg.sender] -= wad;

    if (l1Block.isCustomGasToken() == false) {
      liquidity.burn{ value: wad }();
    }

    sendMessage({
      _destination: chainId,
      _message: abi.encodeCall(SuperchainWETH.relayERC20, (to, wad))
    });
  }

  function sendMessage(uint256 _destination, bytes memory _message) internal {
    messenger.sendMessage({
      _destination: _destination,
      _target: address(this),
      _message: _message
    });
  }

  function relayERC20(address to, uint256 amount) external {
    if (msg.sender != Predeploys.L2_TO_L2_CROSSDOMAIN_MESSENGER) revert Unauthorized();
    if (messenger.crossDomainMessageSender() != address(this)) revert Unauthorized();

    if (l1Block.isCustomGasToken() == false) {
      liquidity.source(amount);
    }

    balanceOf[to] += amount;
  }

  function relayETH(address to, uint256 amount) external {
    if (l1block.isCustomGasToken()) revert IsCustomGasToken();
    if (msg.sender != Predeploys.L2_TO_L2_CROSSDOMAIN_MESSENGER) revert Unauthorized();
    if (messenger.crossDomainMessageSender() != address(this)) revert Unauthorized();

    liquidity.source(amount);

    bool success = SafeCall.transfer({ _target: to, _amount: amount });
    require(success);
  }
}

// Placed in genesis with a balance equal to type(uint248).max
contract ETHLiquidity {
  address weth internal = Predeploys.SUPERCHAIN_WETH;
  L1Block internal l1Block = L1Block(Predeploys.L1_BLOCK);

  function burn() external payable {
    if (msg.sender != weth) revert Unauthorized();
  }

  function source(uint256 amount) {
    if (msg.sender != weth) revert Unauthorized();
    if (l1Block.isCustomGasToken()) revert OnlyEther();
    require(SafeCall.transfer({ _target: weth, _value: amount }));
  }
}
```

### Longer term nice to haves

- Enable ETH interface on `StandardBridge` for custom gas token chains and have it mint `SuperchainWETH` for deposits
    - We can call this out of scope for now, but a future upgrade can make `SuperchainWETH` an `OptimismMintableERC20` to enable this

# Open Questions

## Naming

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
