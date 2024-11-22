## Summary

This design document introduces `Entrypoint` contracts as a new primitive that allows anyone to add custom logic on top of the `L2ToL2CrossDomainMessenger`.
It generalizes the `L2ToL2CrossDomainMessenger` design and unlocks other interop primitives such as message batching and expiring.
To do so, the `L2ToL2CrossDomainMessenger` will allow to authorize a single address to relay a message. The authorized address can then add custom logic before executing the call.

## Problem Statement + Context

Building functionality on top of the `L2ToL2CrossDomainMessenger` is not simple. For example, to allow message recovery, the contract would need to track expired/failed messages to prevent someone from relaying them after recovery. This would require adding code on the `L2ToL2CrossDomainMessenger`. If another feature were required, we would need to add more code, which doesn’t scale.

The `Entrypoint` contract primitive solves this issue and unlocks new powerful cross-chain actions. The integration with `Entrypoint` contracts is simple, and the code difference is small.

## `Entrypoint` Contracts

### Main idea

Anyone can deploy `Entrypoint` contracts with any logic. Users can indicate, explicitly or through a contract, which `Entrypoint` contract can process their messages in the destination chain. `relayMessage` will check if the message has an `Entrypoint` associated, and if it does, it will validate that this contract is the `msg.sender`; otherwise, it will revert.

The key modification is the `msg.sender` binding feature on the `relayMessage` call. If `msg.sender` has logic, the design is equivalent to binding `relayMessage` to include a pre-hook with that logic.

Notice that post-hook design is already possible with the current design using logic on `target`.

The single authorized address design can also work to permission relaying for an EOA. There is no check that the calling address is a contract or not.

### Intuition Example

This section will describe a simple example to showcase the power of `Entrypoint` contracts. Let’s consider the situation where Alice wants to send `AliceSuperToken` from Chain A to Chain B and then swap them.

1. Alice (or anyone) deploys a `SwapperEntrypoint` as an `Entrypoint` contract in Chain B that can:
    - Receive `AliceSuperToken` tokens.
    - Swap for a target token.
2. Alice (or anyone) deploys a `CrosschainSwapper` contract in chain A that will have a `sendAndSwap()` function.
    - We assume the `CrosschainSwapper` is allowed to call `crosschainBurn()` on `AliceSuperToken`.
3. Alice calls `sendAndSwap()` . The `CrosschainSwapper` burns tokens on behalf of Alice, and calls `sendMessage()` in `L2ToL2CrossDomainMessenger` with the `SwapperEntrypoint` in destination as `entrypoint`.
4. `Alice` (or anyone) goes to the `SwapperEntrypoint` in Chain B, and calls `relaySendAndSwap()` with her message. The contract will a. Call `relayMessage()` in the `L2ToL2CrossDomainMessenger` and process her message, which relays the `AliceSuperToken` to the `SwapperEntrypoint` contract. b. Perform a swap in a Uniswap pool, for a target token. c. Optionally, the `SwapperEntrypoint` could bridge the target token back to Alice on chain A.

The message cannot be relayed by any other `msg.sender` besides the `SwapperEntrypoint`, or it will revert. This is what we mean by binding.

This will allow Alice to perform complex but safe cross-chain actions with a simple interaction in the source chain.

## `entrypointContext`

### Main idea

Following the intuition example above, we could notice a problem in 4b: the `SwapperEntrypoint` doesn’t know what pool to use, how much of the total tokens given to it to swap, or who to give the tokens to after the swap. Information is missing. `SwapperEntrypoint` could ask for these values in the `swapTokens` function, but anyone could call it with Alice’s event and provide bad values.

To solve this, `Entrypoint` contracts can define a decoding method and expect a second event. Then, `Entrypoint` can recover the required parameters from a second validated message, and then perform its logic as expected.

In origin, Alice would emit the `EntrypointContext` as a separate event, and send her message with her designed `Entrypoint` contract. This way, the `Entrypoint` contract can validate the `EntrypointContext` event from origin, decode it, and perform the required actions.

Therefore, the processing message function in the `Entrypoint` context will validate and consume two messages: the regular `SentMessage` event and the `EntrypointContext` event.

An alternative design could be to enshrine the `entrypointContext` as a `bytes` field from within the `SentMessage` event. We discuss this approach in [Future Considerations](#enshrined-entrypointcontext).

Notice that, in the intuition example, the `SwapperEntrypoint` is used as both pre and post-hook for the message relay. The `EntrypointContext` is used here in the post-action, but it would be needed beforehand in other cases (like batching).

### Context Binding

It is important to note that `EntrypointContext` is not binding to its connected initiating message. This means that, in theory, anyone could execute the message alongside any other valid `EntrypointContext`. 
Each `Entrypoint` and initiating message contract will be responsible for handling authentication logic.

`Entrypoints` can follow the model of the `L2ToL2CrossDomainMessenger` and decode additional parameters.
Of course, this would require the `EntrypointContext` event to encode this information.
Some parameters that can be used for binding are
- `L2ToL2CrossDomainMessenger` current `nonce`. The `Entrypoint` can then check that the `EntrypointContext` has the same `nonce` than the `SentMessage`.
- Identifier's `origin` from `EntrypointContext` event matches `sender` from `SentMessage`. This could show that the context was created on the same contract than the connected message.
- Identifier's `chainId` from `EntrypointContext` event matches `source` from `SentMessage`. Should match with the `SentMessage`.

However, the most efficient way of binding both events is to check who emitted the `EntrypointContext` event against an allowed-list. This might seem restricting, but will be more than enough for many applications.

### Summary of required changes

The following changes are required:

- `sendMessage` includes `address entrypoint`.
- `SentMessage` event includes `address entrypoint`.
- `relayMessage` checks if `entrypoint!=0`. If so, it checks if `msg.sender == entrypoint` and reverts if not.
- `_decodeSentMessagePayload` will decode the `entrypoint`.
- `hashL2toL2CrossDomainMessage` will include `entrypoint`.

## Full example: Expire messages

Here’s an example of how `Entrypoint` contracts with `EntrypointContext` events can enable powerful features like the expiration of messages:

Suppose an `ExpirableTokenBridge` contract exists, that uses an `Entrypoint` with the added functionality of expiring a failed message in destination. `ExpirableTokenBridge` has minting and burning rights over `SuperchainERC20`.

- Alice will transfer `AliceSuperToken` from Chain A to Chain B through the `ExpirableTokenBridge`.
- If `AliceSuperToken` has not yet been deployed, relaying the message will fail.
- The user will then use the `Entrypoint` functionality to expire the message, making it non-processable by adding it to a mapping. Any call to the `Entrypoint` to relay the expired message will revert. As the `Entrypoint` is the only contract that can process this message in the `L2ToL2CrossDomainMessenger`, the expired message is effectively non-processable in Chain B.
	- (Optional) The expiring function can also check `EntrypointContext` parameters, such as time window.
- Lastly, after the relaying failure, the user will call the `Entrypoint` again, on chain B, to expire a message. This will emit a message that the contract in Chain A can consume to handle the expired message.

The sequence diagrams will be divided in three:

1. Shows the initiating interaction of the user with the `ExpirableTokenBridge` to send the message to Chain B.
2. Shows the interaction of the user in Chain B with the `Entrypoint` to process the expirable message, and shows how it fails to be relayed, and how the `Entrypoint` expires it.
3. Shows the interaction of the user back with the `ExpirableTokenBridge` in Chain A to recover the lost tokens.

```mermaid
sequenceDiagram

title Message creation (Chain A)

Alice->>ExpirableTokenBridge_A: sendERC20(to)
ExpirableTokenBridge_A->>L2ToL2CrossDomainMessenger: sendMessage(..., entrypointAddr)
L2ToL2CrossDomainMessenger-->L2ToL2CrossDomainMessenger: emit SentMessage(...params, entrypointAddr, message)
ExpirableTokenBridge_A-->ExpirableTokenBridge_A: emit EntrypointContext(expirerAddr, token, deadline, receiverOfExpiredMessage, receiverOfExpiredTokens)
```

```mermaid
sequenceDiagram

title Message failed and expired (Chain B)

Anyone->>Entrypoint: processMessage(Identifier messageId, Message message, Identifier contextId, Message context)
Entrypoint->>CrossL2Inbox: Validate message and EntrypointContext event
Entrypoint->>L2ToL2CrossDomainMessenger: relayMessage()
L2ToL2CrossDomainMessenger->>ExpirableTokenBridge_B: relayERC2O(to, amount)
ExpirableTokenBridge_B->>SuperchainERC20_B: FAILS
Anyone->>Entrypoint: expireMessage(messageHash)
Entrypoint-->Entrypoint: checks if the deadline of the message is due
Entrypoint-->L2ToL2CrossDomainMessenger: checks if the message has been successfully relayed before
Entrypoint-->Entrypoint: adds it to expiredMessages mapping to avoid future processing
Entrypoint-->Entrypoint: emit ExpiredMessage(destination, messageHash, receiverOfExpiredMessage)
```

```mermaid
sequenceDiagram

title Expired message handling and funds recovery (Chain A)

Anyone-->ExpirableTokenBridge_A: handleExpired(Identifier id, ExpiredMessage expiredMessage)
ExpirableTokenBridge_A->>CrossL2Inbox: validateMessage
ExpirableTokenBridge_A-->ExpirableTokenBridge_A: adds to expired messages mapping
ExpirableTokenBridge_A-->SuperchainERC20_A: crosschainMint(receiverOfExpiredTokens)
```

## Notes and Open Questions

- Notice the modifications basically unlock permissioned relaying. `Entrypoints` are the most clear and useful way to take benefit of this permission control, but it could potentially be used for other actions.
- Names are all up for debate and change.
- For `EntrypointContext`, should we emit the full struct, an encoded bytes or a hash?

## Future Considerations

### Enshrined `entrypointContext`

This second design binds the context to the message and makes the `L2ToL2CrossDomainMessenger` perform a callback to the entrypoint with this data. For the moment, we decided to let this feature outside of the `L2ToL2CrossDomainMessenger` to keep it minimal, but it can be considered. It is also possible to implement into into a second `L2ToL2CrossDomainMessengerWithContext` that extends from `L2ToL2CrossDomainMessenger`.

The context is binded into the message, meaning that its not possible to frontrun the `SentMessage` event with a different context.

Entrypoints that wish to use enshrined `entrypointContext` would need to implement the `onRelayMessage` function. The `L2ToL2CrossDomainMessenger` will perform a callback to `onRelayMessage` inside the `relayMessage` call and before calling the target.  

`Entrypoint` can use the context however they want. It’s important to notice the `relayMessage()` is non reentrant, so the callback cannot call it back. Some possible ways to use the context are the following:


**Optimistic**

Context is assumed to be true, but revert on callback if not. 

1. In origin, Alice will encode the `entrypointContext` as a single `bytes` parameter into the `sendMessage` call. 
2. On destination, the `Entrypoint` will ask for the context as a user input, in addition to the message and id. `Entrypoint` will assume this input context is true (optimistic).
    1. `Entrypoint` will include an `onRelayMessage` callback.
3. The `Entrypoint` will check that the context is valid.
4. If so, it will call `relayMessage`, which will 
    1. validate usual properties of the message (domain binding, non-replayability).
    2. check `entrypoint == msg.sender`.
    3. do a callback to the `Entrypoint` for `onRelayMessage`, passing the encoded `entrypointContext`. `onRelayMessage` will check that the context that was inputed matches the `entrypointContext`, and revert if not.
    4. finally, call `target` with the `message`.

**Store and use later**

Some applications might not require the context at the time of the message (expire messages for instance). In that case, the `Entrypoint` can ignore the context parameter and just store it on the callback for later use.

1. In origin, Alice will encode the `entrypointContext` as a single `bytes` parameter into the `sendMessage` call.
2. On destination, the `Entrypoint` will ask for the message and id.
    1. `Entrypoint` will include an `onRelayMessage` callback.
3. `Entrypoint` will call `relayMessage`, which will
    1. validate usual properties of the message (domain binding, non-replayability).
    2. check `entrypoint == msg.sender`.
    3. do a callback to the `Entrypoint` for `onRelayMessage`, passing the encoded `entrypointContext`. `onRelayMessage` the `Entrypoint` will store the context for later use.
    4. finally, call `target` with the `message`.

### Changes for enshrinement

The following changes are required:

- Overload `sendMessage` to
    - include `address entrypoint` and `bytes entrypointContext`.
    - modify `SentMessage` event to include `address entrypoint` and `bytes entrypointContext`.
- Modify `relayMessage` to
    - check if `msg.sender != address(0)` to check if an entrypoint has been defined.
    - if entrypoint has been defined, check if `msg.sender == entrypoint` and reverts if not.
    - if entrypoint has been defined and if `entrypointContext.length > 0` , performs a callback to the `entrypoint` address with `entrypointContext` data to the `onRelayMessage` function.
- `_decodeSentMessagePayload` will decode the `entrypoint` and `entrypointContext`.
