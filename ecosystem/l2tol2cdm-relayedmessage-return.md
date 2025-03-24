# Purpose

The ability to read events with Superchain interop is incredibly powerful. We should make sure we maximize the potential of all the default events emmitted when interacting with the OP-Stack predeploys.

## Goals

- Faciliate cross chain async programming

# Problem Statement + Context

As we horizontally scale, contract development will look more asynchronous. A lot of cross chain messages are predominalty write-based.

- Bridge Funds
- Invoke calldata on via a remotely controlled proxy account

Read-based cross chain messages are less common but will increasingly become so in an asynchronous world. Solidity has a `view` modifier specifically targeted for off chain reading -- it would be natural to also want to also query this state onchain very quickly.

- Fetch remote swap quotes
- Query balance/ownership status
- Chain on subsequent write actions, only after success (deposit -> borrow -> brige)

In order to support use cases like these, custom handler contracts per-application must be written to capture these side effects and return data backwards or chain on additional side effects. For example

```solidity
contract ERC20BalanceGateway {
    L2ToL2CrossDomainMessenger messenger;

    // Gateway contract that can be used to query ERC20s with a specified handler
    function query(IERC20 token, address account, bytes4 selector) external {
        uint256 balance = token.balanceOf(account);

        // Message sent back with the value as an encoded argument
        (address sender, uint256 source) = messenger.crossDomainMessageContext();
        messenger.sendMessage(source, sender, abi.encode(selector, balance));
    }
}
```

# Proposed Solution

When the `L2ToL2CrossDomainMessenger` invokes a target with calldata, the return data of that call is captured and returned within the `relayMessage` function. However this return data is only actionable for the caller of this function, typically the relayer, or a faciliating smart contract.

`relayMessage` emits the `RelayedMessage` event on successful execution which itself can be consumed remotely via the `CrossL2Inbox` predeploy. By including the return data in this event, it becomes immediately visible to the sending chain (and all others in the interop set) without any additional message or handler contract.

```solidity
contract L2ToL2CrossDomainMessenger {
    function relayMessage(Identifier calldata _id, bytes calldata _sentMessage) {
        // include return data
        (, returnData_) = target.call{ value: msg.value }(message);
        emit RelayedMessage(source, nonce, messageHash, returnData_);
    }
}
```

The `messageHash` known ahead of time can be correlated with the corresponding `RelayedMessage`. Thus the return value of any read function or return value of a write call becomes immediately actionable for the sender, without requiring special integration.

See the Promise Library [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/216)

## Alternatives Considered

### Entrypoint

With [Entrypoints](https://github.com/ethereum-optimism/specs/pull/484), we could envision a general purpose entrypoint that simply emits the captured return values

```solidity
contract ReturnEmitterEntrypoint {
    function relayMessage(Identifier calldata _id, bytes calldata _sentMessage) {
        bytes32 parsedMessageHash;
        bytes memory returnData = messenger.relayMessage(_id, _sentMessage);
        emit ReturnValue(parsedMessageHash, returnData)
    }
}
```

However this limits additional tooling such as the Promise library. Where every outbound message from the L2ToL2CDM returns a message hash that can behave like a Promise. In this approach, only outbound messages leveraging this entrypoint could.

Entrypoint composition is also a bit undefined. Thus creates more uncertain behavior with a Promise-like abstraction or capturing the return amongst the involvement of different entrypoints.

# Risks & Uncertainties

The extra gas cost associated with including a variable sized input into `RelayedMessage`. There's a gas cost of 8 per non-zero bytes (4 for zeros) in log data. This is not a real concern for a couple of reasons

1. Dynamically sized return values are generally discouraged, and it is good practice in solidity to ensure bounds here when writing contracts.
2. The caller to `relayMessage` is already gas-sensitive to the captured return data since it returns this to the caller
3. Just as these gas costs are a concern in a single-chain development experience, the developer looking to query a contract cross chain should also be aware of the gas costs of invoking any dynmically sized return values.
