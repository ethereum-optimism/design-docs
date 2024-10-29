# Purpose

This document exists to align on a design for simplifying the process for sending `ETH` between two interoperable chains.

# Summary

New functionality is introduced to the `ETHLiquidity` contract that enables it to facilitate cross-chain `ETH` transfers to a specified recipient.

# Problem Statement + Context

Currently, L2-to-L2 `ETH` transfers between two interoperable chains require four separate transactions:

1. Wrap `ETH` to `SuperchainWETH`.
2. Call `SuperchainTokenBridge#SendERC20` to burn `SuperchainWETH` on source chain and relay a message to the destination chain that mints `SuperchainWETH` to the recipient.
3. Execute the message on the destination chain, triggering `SuperchainTokenBridge#RelayERC20` to mint `SuperchainWETH` to the recipient.
4. Unwrap the received `SuperchainWETH` to `ETH`.

The goal is to reduce the transaction count from four to two, enabling users to send `ETH` to a destination chain directly:

1. Burn `ETH` on source chain and relay a message to destination chain that mints `ETH` to recipient on destination chain.
2. Execute the relayed message on the destination chain that mints `ETH` to the recipient.

# Proposed Solution

Introduce `SendETH` and `RelayETH` functions to the `ETHLiquidity` contract. By adding these functions directly to `ETHLiquidity`, the contract retains its original purpose of centralizing the minting and burning of `ETH` for cross-chain transfers, ensuring that all liquidity management occurs within a single, dedicated contract.

### `SendETH` function

The `SendETH` function combines the first two transactions as follows:

1. Burns `ETH` within the `ETHLiquidity` contract equivalent to the `ETH` sent.
2. Sends a message to the destination chain encoding a call to `RelayETH`.

### `RelayETH` function

The `RelayETH` function combines the last two transactions as follows:

1. Mints the specified `_amount` of `ETH` from the `ETHLiquidity` contract.
2. Sends the minted `ETH` to the specified recipient.

### Contract changes

Add an internal `_burn` function that will be called by `ETHLiquidity#SendETH` and update the external `burn` function to call `_burn`:

```solidity
/// @notice Allows an address to lock ETH liquidity into this contract.
function burn() external payable {
    if (msg.sender != Predeploys.SUPERCHAIN_WETH) revert Unauthorized();
    _burn(msg.sender, msg.value);
}

/// @notice Allows an address to lock ETH liquidity into this contract.
/// @param _sender Address that sent the ETH to be locked.
/// @param _amount The amount of liquidity to burn.
function _burn(address _sender, uint256 _amount) internal {
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) revert NotCustomGasToken();
    emit LiquidityBurned(_sender, _amount);
}
```

Add an internal `_mint` function that will be called by `ETHLiquidity#RelayETH` and update the external `mint` function to call `_mint`:

```solidity
/// @notice Allows an address to unlock ETH liquidity from this contract.
/// @param _amount The amount of liquidity to unlock.
function mint(uint256 _amount) external {
    if (msg.sender != Predeploys.SUPERCHAIN_WETH) revert Unauthorized();
    _mint(msg.sender, _amount);
}

/// @notice Allows an address to unlock ETH liquidity from this contract.
/// @param _recipient Address to send the unlocked ETH to.
/// @param _amount    The amount of ETH liquidity to unlock.
function _mint(address _recipient, uint256 _amount) internal {
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) revert NotCustomGasToken();
    new SafeSend{ value: _amount }(payable(_recipient));
    emit LiquidityMinted(_recipient, _amount);
}
```

Add `SendETH` and `RelayETH` function to `ETHLiquidity`:

```solidity
/// @notice Sends ETH to some target address on another chain.
/// @param _to       Address to send ETH to.
/// @param _chainId  Chain ID of the destination chain.
function sendETH(address _to, uint256 _chainId) public payable {
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        revert NotCustomGasToken();
    }

    _burn(msg.sender, msg.value);

    // Send message to other chain.
    IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER).sendMessage({
        _destination: _chainId,
        _target: address(this),
        _message: abi.encodeCall(this.relayETH, (msg.sender, _to, msg.value))
    });
    emit SendETH(msg.sender, _to, msg.value, _chainId);
}

/// @notice Relays ETH received from another chain.
/// @param _from    Address of the msg.sender of sendETH on the source chain.
/// @param _to      Address to relay ETH to.
/// @param _amount  Amount of ETH to relay.
function relayETH(address _from, address _to, uint256 _amount) external {
    IL2ToL2CrossDomainMessenger messenger = IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER);
    if (msg.sender != address(messenger)) revert Unauthorized();
    if (messenger.crossDomainMessageSender() != address(this)) revert Unauthorized();
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        revert NotCustomGasToken();
    }

    _mint(_to, _amount);
    emit RelayETH(_from, _to, _amount, messenger.crossDomainMessageSource());
}
```

# Considerations

## SuperchainWETH transfers

This solution does not eliminate the `SuperchainWETH` contract and `SuperchainWETH` transfers would still go through the `SuperchainTokenBridge`.

## Custom Gas Token Chains

To simplify the solution, custom gas token chains will not be supported and must follow the original four-transaction flow, which includes wrapping and unwrapping `SuperchainWETH`.

# Open Questions
- **Long-term improvements**: Could this functionality eventually extend to custom gas token chains?
- **Rollbacks**: How would rollbacks be handled in this implementation?

# Alternatives Considered

## Integrate `ETH` transfer into `SuperchainWETH`

Add two new functions to the `SuperchainWETH` contract: `SendETH` and `RelayETH`.

### `SendETH`

The `SendETH` function combines the first two transactions as follows:

1. Burns `ETH` within the `ETHLiquidity` contract equivalent to the `ETH` sent.
2. Sends a message to the destination chain encoding a call to `RelayETH`.

### `RelayETH`

The `RelayETH` function combines the last two transactions as follows:

1. Mints the specified `_amount` of `ETH` from the `ETHLiquidity` contract.
2. Sends the minted `ETH` to the specified recipient.

### Contract changes

```solidity
/// @notice Sends ETH to some target address on another chain.
/// @param _to      Address to send tokens to.
/// @param _chainId  Chain ID of the destination chain.
function sendETH(address _to, uint256 _chainId) public payable {
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        revert NotCustomGasToken();
    }

    // Burn to ETHLiquidity contract.
    IETHLiquidity(Predeploys.ETH_LIQUIDITY).burn{ value: msg.value }();

    // Send message to other chain.
    IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER).sendMessage({
        _destination: _chainId,
        _target: address(this),
        _message: abi.encodeCall(this.relayETH, (msg.sender, _to, msg.value))
    });

    // Emit event.
    emit SendETH(msg.sender, _to, msg.value, _chainId);
}

/// @notice Relays ETH received from another chain.
/// @param _from    Address of the msg.sender of sendETH on the source chain.
/// @param _to     Address to relay tokens to.
/// @param _amount     Amount of tokens to relay.
function relayETH(address _from, address _to, uint256 _amount) external {
    // Receive message from other chain.
    IL2ToL2CrossDomainMessenger messenger = IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER);
    if (msg.sender != address(messenger)) revert CallerNotL2ToL2CrossDomainMessenger();
    if (messenger.crossDomainMessageSender() != address(this)) revert InvalidCrossDomainSender();
    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        revert NotCustomGasToken();
    }

    // Mint from ETHLiquidity contract.
    IETHLiquidity(Predeploys.ETH_LIQUIDITY).mint(_amount);

    // Get source chain ID.
    uint256 source = messenger.crossDomainMessageSource();

    new SafeSend{ value: _amount }(payable(_to));

    // Emit event.
    emit RelayETH(_from, _to, _amount, source);
}
```

### Advantages

Since the `SuperchainWETH` contract already has permissions to mint and burn `ETH` with the `ETHLiquidity` contract no changes to the `ETHLiquidity` contract are necessary. Any security risks with this change are limited to `SuperchainWETH` and do not affect other `SuperchainERC20` tokens.

### Downsides

This approach introduces an additional entry point for asset transfers outside of `SuperchainTokenBridge`. While `SuperchainERC20` tokens that implement custom bridging will already have alternative entry points, adding another for native `ETH` transfers could lead to confusion and diverge from a clean separation of concerns. Additionally, by having `SuperchainWETH` serve as both a `SuperchainERC20` and a bridge for native `ETH`, there is an increased risk of ambiguity around its dual function, which could impact usability and clarity for developers.

## Integrate `ETH` transfer into `SuperchainTokenBridge`

One alternative is to add two new functions to the `SuperchainTokenBridge` contract: `SendETH` and `RelayETH`.

```solidity
/// @notice Sends ETH to a target address on another chain.
/// @param _to       Address to send ETH to.
/// @param _amount   Amount of ETH to send.
/// @param _chainId  Chain ID of the destination chain.
/// @return msgHash_ Hash of the message sent.
function sendETH(address _to, uint256 _amount, uint256 _chainId) external payable returns (bytes32 msgHash_) {
    if (_to == address(0)) revert ZeroAddress();

    ISuperchainWETH(payable(Predeploys.SUPERCHAIN_WETH)).deposit{ value: msg.value }();
    ISuperchainWETH(payable(Predeploys.SUPERCHAIN_WETH)).crosschainBurn(msg.sender, msg.value);

    bytes memory message = abi.encodeCall(this.relayETH, (msg.sender, _to, _amount));
    msgHash_ = IL2ToL2CrossDomainMessenger(MESSENGER).sendMessage(_chainId, address(this), message);

    emit SendERC20(Predeploys.SUPERCHAIN_WETH, msg.sender, _to, _amount, _chainId);
}

/// @notice Relays ETH received from another chain.
/// @param _from    Address of the msg.sender of sendETH on the source chain.
/// @param _to      Address to relay ETH to.
/// @param _amount  Amount of ETH to relay.
function relayETH(address _from, address _to, uint256 _amount) external {
    if (msg.sender != MESSENGER) revert Unauthorized();

    (address crossDomainMessageSender, uint256 source) =
        IL2ToL2CrossDomainMessenger(MESSENGER).crossDomainMessageContext();
    if (crossDomainMessageSender != address(this)) revert InvalidCrossDomainSender();

    ISuperchainWETH(payable(Predeploys.SUPERCHAIN_WETH)).crosschainMint(address(this), _amount);
    ISuperchainWETH(payable(Predeploys.SUPERCHAIN_WETH)).withdraw(_amount);

    new SafeSend{ value: _amount }(payable(_to));

    emit RelayERC20(Predeploys.SUPERCHAIN_WETH, _from, _to, _amount, source);
}
```

### Advantages

The advantage of this solution is that `SuperchainTokenBridge`would handle both `ETH` transfers and `SuperchainERC20` transfers, simplifying developer integrations.

### Downsides

This solution creates changes to a highly sensitive contract. `SuperchainTokenBridge` has permissions to `mint` and `burn` standard `SuperchainERC20` tokens, so updates must be treated with extreme caution.