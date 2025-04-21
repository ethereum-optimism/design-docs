# Callbacks to Sender: Design Doc

|                    |            |
| ------------------ | ---------- |
| Author             | Jeff Huang |
| Created at         | 2025-04-21 |
| Initial Reviewers  |            |
| Need Approval From |            |
| Status             | Draft      |

## Purpose

To help developers better manage `sender`'s state consistency that is contingent on whether `target`'s execution is successful or reverted.

## Summary

This feature introduces an overload of `sendMessage()` with custom `onSuccess` and/or `onRevert` callbacks which is automatically invoked by `L2ToL2CrossDomainMessenger` after relaying message to `target`.

This enables cross-chain state changes to be atomic without introducing logical coupling between `sender` and `target` (i.e. `target` does not need to know the implementation details on `sender`). Also, this is all achieved through the same single user transaction to `sendMessage()`.

## Problem Statement + Context

Interops often lead to state changes on both `source` and `destination` that must be consitent. A simple example is `sender` contract locks user fund on `source` and request `target` to debit the approperiate amount on `destination`. Currently there is no mechanism to ensure the two actions occur atomically, so a failed debit for any reason would lead chain states to be inconsistent.

One way around this is for `source` to cache actions on its side and provide an additional function that `target` can call to roll them back. However, this introduces a tight coupling between `source` and `target` while also require `target` to self-call with a `try/catch` to capture any revert.

## Proposed Solution

The proposed solution is to expand `SendMessage` event and overload `sendMessage()` with `onSuccess` and/or `onRevert` callbacks that `L2ToL2CrossDomainMessenger` will automatically invoke depending on what `target.call()` returns.

```solidity
contract L2ToL2CrossDomainMessenger {
    event SentMessage(
        uint256 indexed destination, address indexed target, uint256 indexed messageNonce, address sender, bytes message, bytes onSuccess, bytes onRevert
    );

    function sendMessage(uint256 _destination, address _target, bytes calldata _message) external returns (bytes32 messageHash_) {
        return sendMessage(_destination, _target, _message, "", "");
    }

    function sendMessage(uint256 _destination, address _target, bytes calldata _message, bytes calldata _onSuccess, bytes calldata _onRevert)
        external returns (bytes32 messageHash_)
    {
        // ...

        emit SentMessage(_destination, _target, nonce, msg.sender, _message, _onSuccess, _onRevert);
    }

    function relayMessage(Identifier calldata _id, bytes calldata _sentMessage)
        external payable nonReentrant returns (bytes memory returnData_)
    {
        // ...

        (uint256 destination, address target, uint256 nonce, address sender, bytes memory message, bytes memory onSuccess, bytes memory onRevert) =
            _decodeSentMessagePayload(_sentMessage);

        // ...

        bool success;
        (success, returnData_) = target.call{ value: msg.value }(message);

        if (success) {
            if (onSuccess.length > 0) sendMessage(_id.chainId, sender, onSuccess);
        } else {
            if (onRevert.length > 0) {
                sendMessage(_id.chainId, sender, onSuccess);
            } else {
                assembly {
                    revert(add(32, returnData_), mload(returnData_))
                }
            }
        }
    }
}
```

### Resource Usage

It adds additional bytes to `L2ToL2CrossDomainMessenger` calldata and `SendMessage` event.

### Single Point of Failure and Multi Client Considerations

TBD

## Failure Mode Analysis

TBD

## Impact on Developer Experience

This is a quality-of-life feature that should improve developer experience.

- Ensure `sender` state is consistent with `target` execution outcome
- Reduce self-calling and boilerplate code on `target`
- Decouple `sender` from `target`, `sender` can specify its own callback signatures and not confined to some interface that `target` requires

## Alternatives Considered

[Promise](https://github.com/ethereum-optimism/supersim/blob/main/contracts/src/Promise.sol) has some similarities but is only intended for the happy execution path.

[Return data in relayed message event](https://github.com/ethereum-optimism/optimism/pull/14599) could also potentially be used but requires a second transaction to query `CrossL2Inbox` and breaks atomicity.

## Risks & Uncertainties

- Currently any revert during `target.call()` will cause `L2ToL2CrossDomainMessenger` to also revert, changes in proposed solution will "swallow" the revert if `onRevert` is specified. Uncertain if this would cause any down stream problem.

TBD
