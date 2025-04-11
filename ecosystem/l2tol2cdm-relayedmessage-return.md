# L2ToL2CrossDomainMessenger RelayedMessage return event field: Design Doc

|                    |                                         |
| ------------------ | --------------------------------------- |
| Author             | Hamdi Allam                             |
| Created at         | 2025-03-24                              |
| Initial Reviewers  | Wonderland, Mark Tyneway, Harry Markley |
| Need Approval From | Wonderland, Mark Tyneway, Harry Markley |
| Status             | In Review                               |

## Purpose

The ability to read events with Superchain interop is incredibly powerful. We should make sure we maximize the potential of all the default events emitted when interacting with the OP-Stack predeploys.

## Summary

With an added field to the `RelayedMessage` event emitted by the `L2ToL2CrossDomainMessenger`, we open the door to better async contract development within the Superchain.

## Problem Statement + Context

As we horizontally scale, contract development will look more asynchronous. A lot of cross chain messages are predominantly write-based.

- Bridge Funds
- Invoke calldata on a remotely controlled proxy account

Read-based cross chain messages are less common but will increasingly become so in an asynchronous world. Solidity has a `view` modifier specifically targeted for reading -- it would be natural to also want to also query this state onchain very quickly.

- Fetch remote swap quotes
- Query balance/ownership status
- Chain on subsequent write actions, only after success (deposit -> borrow -> bridge)

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

## Proposed Solution

When the `L2ToL2CrossDomainMessenger` invokes a target with calldata, the return data of that call is captured and returned via `relayMessage`. However this return data is only actionable for the caller of this function, typically the relayer, or a faciliating smart contract.

`relayMessage` emits the `RelayedMessage` event on successful execution which itself can be consumed remotely via the `CrossL2Inbox` predeploy. By including the return data in this event, it becomes immediately visible to the sending chain (and all others in the interop set) without any additional message or handler contract.

To limit the gas overhead of this inclusion, the hash of the return data is emitted such that authentication is possible by the consumer with the provided data.

```solidity
contract L2ToL2CrossDomainMessenger {
    function relayMessage(Identifier calldata _id, bytes calldata _sentMessage) {
        // include return data
        (, returnData_) = target.call{ value: msg.value }(message);
        emit RelayedMessage(source, nonce, messageHash, keccak256(returnData_));
    }
}
```

The `messageHash` known ahead of time can be correlated with the corresponding `RelayedMessage`. Thus the return value of any read function or return value of a write call becomes immediately actionable for the sender, without requiring special integration.

See the Promise Library [Design Doc](https://github.com/ethereum-optimism/design-docs/pull/216)

### Resource Usage

The extra gas cost associated with including a variable sized input into the `RelayedMessage` log. There's a gas cost of 8 per non-zero byte (4 for zeros) in log data.

We cap the overhead by including a hash of the return data that can be authenticated by the consumer of the `RelayedMessage` event. It is up to the relayer of this event to include the appropriate return data for the caller.

### Single Point of Failure and Multi Client Considerations

This proposal doesnt change the callpath of cross domain messenger, thus has changes no impact on how messages are relayed. The L2ToL2CrossDomainMessenger is not used in production outside of the existing devnet. Thus it is a good opportunity to make this change even thought it breaks the event signature for any indexers that may rely on it.

- `RelayedMessage` also is not indexed in the callpath relative to `SentMessage` which is required to relaye a message, thus additionally safe to break right now.

There's some additional gas overhead to relaying messages. A malicious contract can variable return a large amount of data to make it hard to query. Hashing the return data included in the event mitigates the blast radius of the gas increase.

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

However this limits additional tooling such as a general Promise library. Where every outbound message from the L2ToL2CDM returns a message hash that can behave like a Promise. In this approach, only outbound messages leveraging this entrypoint could.

Entrypoint composition is also a bit undefined. Thus creates more uncertain behavior with a Promise-like abstraction or capturing the return amongst the involvement of different entrypoints.

## Risks & Uncertainties

### Resource Usage

See the [Resource Usage](#resource-usage) section.
