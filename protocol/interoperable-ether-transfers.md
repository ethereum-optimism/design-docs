# Purpose

This document exists to align on a design for simplifying the process for sending `ETH` between two interoperable chains.

# Summary

New functionality is introduced to the `SuperchainWETH` contract that enables it to facilitate cross-chain `ETH` transfers to a specified recipient.

# Problem Statement + Context

Currently, L2-to-L2 `ETH` transfers between two interoperable chains require four separate transactions:

1. Wrap `ETH` to `SuperchainWETH`.
2. Call `SuperchainTokenBridge#SendERC20` to burn `SuperchainWETH` on source chain and relay a message to the destination chain that mints `SuperchainWETH` to the recipient.
3. Execute the message on the destination chain, triggering `SuperchainTokenBridge#RelayERC20` to mint `SuperchainWETH` to the recipient.
4. Unwrap the received `SuperchainWETH` to `ETH`.

The goal is to reduce the transaction count from four to two, enabling users to send `ETH` to a destination chain directly:

1. Locks `ETH` on source chain and relay a message to destination chain that relays `ETH` to recipient on destination chain.
2. Execute the relayed message on the destination chain that relays `ETH` to the recipient.

# Proposed Solution

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
/// @notice Emitted when ETH is sent from one chain to another.
/// @param from          Address of the sender.
/// @param to            Address of the recipient.
/// @param amount        Amount of ETH sent.
/// @param destination   Chain ID of the destination chain.
event SendETH(address indexed from, address indexed to, uint256 amount, uint256 destination);

/// @notice Emitted whenever ETH is successfully relayed on this chain.
/// @param from          Address of the msg.sender of sendETH on the source chain.
/// @param to            Address of the recipient.
/// @param amount        Amount of ETH relayed.
/// @param source        Chain ID of the source chain.
event RelayETH(address indexed from, address indexed to, uint256 amount, uint256 source);

/// @notice Sends ETH to some target address on another chain.
/// @param _to       Address to send ETH to.
/// @param _chainId  Chain ID of the destination chain.
function sendETH(address _to, uint256 _chainId) external payable returns (bytes32 msgHash_) {
    if (_to == address(0)) revert ZeroAddress();

    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        revert NotCustomGasToken();
    }

    // Burn to ETHLiquidity contract.
    IETHLiquidity(Predeploys.ETH_LIQUIDITY).burn{ value: msg.value }();

    // Send message to other chain.
    msgHash_ = IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER).sendMessage({
        _destination: _chainId,
        _target: address(this),
        _message: abi.encodeCall(this.relayETH, (msg.sender, _to, msg.value))
    });

    // Emit event.
    emit SendETH(msg.sender, _to, msg.value, _chainId);
}

/// @notice Relays ETH received from another chain.
/// @param _from       Address of the msg.sender of sendETH on the source chain.
/// @param _to         Address to relay ETH to.
/// @param _amount     Amount of ETH to relay.
function relayETH(address _from, address _to, uint256 _amount) external {
    if (msg.sender != Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER) revert Unauthorized();

    (address crossDomainMessageSender, uint256 source) =
        IL2ToL2CrossDomainMessenger(Predeploys.L2_TO_L2_CROSS_DOMAIN_MESSENGER).crossDomainMessageContext();

    if (crossDomainMessageSender != address(this)) revert InvalidCrossDomainSender();

    if (IL1Block(Predeploys.L1_BLOCK_ATTRIBUTES).isCustomGasToken()) {
        _mint(_to, _amount);

        emit RelayETH(_from, _to, _amount, source);

        return;
    }

    IETHLiquidity(Predeploys.ETH_LIQUIDITY).mint(_amount);

    new SafeSend{ value: _amount }(payable(_to));

    emit RelayETH(_from, _to, _amount, source);
}
```

### Advantages
- Since the `SuperchainWETH` contract already has permissions to mint and burn `ETH` with the `ETHLiquidity` contract no changes to the `ETHLiquidity` contract are necessary. Any security risks with this change are limited to `SuperchainWETH` and do not affect other `SuperchainERC20` tokens.

### Downsides
- This approach introduces an additional entry point for asset transfers outside of `SuperchainTokenBridge`. While `SuperchainERC20` tokens that implement custom bridging will already have alternative entry points, adding another for native `ETH` transfers could lead to confusion and diverge from a clean separation of concerns. Additionally, by having `SuperchainWETH` serve as both a `SuperchainERC20` and a bridge for native `ETH`, there is an increased risk of ambiguity around its dual function, which could impact usability and clarity for developers.

# Considerations

## SuperchainWETH transfers

This solution does not eliminate the `SuperchainWETH` ERC20 token. `SuperchainWETH` ERC20 transfers would still go through the `SuperchainTokenBridge`.

## Custom Gas Token Chains

The `SendETH` function will not support custom gas token chains and will revert if called on a custom gas token chain. If the `RelayETH` function is called on a custom gas token chain, it will mint `SuperchainWETH` to the recipient.

# Open Questions
- **Long-term improvements**: Could the `SendETH` functionality eventually extend to custom gas token chains?
- **Rollbacks**: How would rollbacks be handled in this implementation?

# Alternatives Considered

## Integrate ETH transfers into `ETHLiquidity` contract

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

### Advantages
- All ETH liquidity management occurs within a single, dedicated contract.

### Downsides
- Since `ETHLiquidity` holds so much ether, we have to be really careful with what functionality is added to it. It should be callable in the most minimal amount of ways. The original design of `ETHLiquidity` intended there to be only a single useful caller. We need to be very careful with any modifications to it

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
- The advantage of this solution is that `SuperchainTokenBridge`would handle both `ETH` transfers and `SuperchainERC20` transfers, simplifying developer integrations.

### Downsides
- This solution creates changes to a highly sensitive contract. `SuperchainTokenBridge` has permissions to `mint` and `burn` standard `SuperchainERC20` tokens, so updates must be treated with extreme caution.