# [L2ToL2CrossDomainMessenger Gas Receipt]: Design Doc

|                    |                                                          |
| ------------------ | -------------------------------------------------------- |
| Author             | _Hamdi Allam_                                            |
| Created at         | _2025-04-14_                                             |
| Initial Reviewers  | _Wonderland, Mark Tyneway, Harry Markley, Karl Floersch_ |
| Need Approval From | _Wonderland, Ben Clabby, Mark Tyneway_                   |
| Status             | _Draft / In Review / Implementing Actions / Final_       |

## Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

As more contracts are built that natively integrate with superchain interop, several single-action experiences can span up to N+1 op-stack chains to achieve their desired outcome. It is important we include the needed features & data to preserve a single-tx experience in the Superchain to not lose developers or users due to either a poor developer or user experience.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

The general mental model for a transaction is that the sending account pays for the gas used. This mental model is the backbone for 4337 or 7702 sponsored transactions where tx.origin is decoupled from the calling account. By emitted a receipt of execution that emits the cost of gas consumped for all relayed cross domain messages, as well as the initial `tx.origin` responsible for all nested cross domain calls, it provides the foundation where the sending account is charged for all computation associated with the first transaction.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Today, a single transaction cross-chain experience is achieved by pushing gas from the source -> destination chain with the message. Relayers are incentivized as long as the pushed funds for gas is sufficiently enough to cover the cost. This works in primitive usecases, where there's a single message from A -> B such as bridging funds. However even this has several drawbacks:

- **Gas Estimation**. Doing on-chain estimation is very inaccurate, resulting in overestimation for high odds of delivery. Some protocols/relayers even pocket this difference instead of refunding the account. And if the account is refunded, it is likely dust on the remote chain for the user.
- **Vendor lock-in**. Contract developers must lock into a crosschain provider (Hyperlane,LayerZero,Wormhole) in order to make use of these services. This tight coupling introduces complexity in order to retain flexibility -- upgradable smart contracts and provider-agonstic interfaces.

Stepping outside this simple usecase, we can enumerate more scenarios in which pushing gas from source -> destination quickly falls apart.

1. **Transitive cross-domain calls**. A -> B -> C. Gas estmation from the source chain (A), is already inefficient in the single hop. This scheme must also additionally overestimate and provide enough gas for each cross-domain call, B -> C. If a cross-domain call was to fail in an intermediate hop, it's unclear how to fund that transaction through from A.

2. **Cross-Domain Composability**. Transitive call domain calls may be between unrelated contracts such as different DeFi protocols. We can not assume to applications are using the same relaying vendor. Hop A->B might leverage Hyperlane to send the message, while B might transitively use Wormhole to send a message to C. **This is why it's important to solve this problem natively**, such that as long as the providers (LayerZero/Wormhole/Hyperlane) use native message passing, everything works.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

    Providing a mechanism for which the accumulated gas used for all cross domain calls can be correlated with the original `tx.origin`.

By natively providing this in the messenger, anyone can charge tx senders exactly what was used for all incurred cross domain messages. It is important to note here that this design doc does not prescribe a solution for charging this gas. This can happen within shared block building, enshrined in a app-level relay bounties framework, or used the 3rdparty providers in their own accounting systems.

Support is enumerated in 2 parts.

### 1. Populated SentMessage "Context".

[#266](https://github.com/ethereum-optimism/design-docs/pull/266) introduces the ability to embed context propogated with cross domain messages. We propose a first version format for this context

- `abi.encodePacked(version0, abi.encode(rootMsgHash, txOrigin))`

The root-most cross domain call is responsible to populating the responsible tx.origin and root message hash that will be propogated to all nested transactions.

```solidity
function sendMessage(...) {
    (,,bytes ctx memory) = crossDomainMessageContext();
    if (ctx.length == 0) {
        // new "top-level" cross domain call (messageHash_ == outbound message)
        ctx = abi.encodePacked(version, abi.encode(messageHash_, tx.origin));
    }

    // keep the ctx as-is for nested messages, this format does not require re-encoding

    ...

    emit SentMessage(..., ctx)
}
```

### 2. Gas Receipt

All superchain interop cross domain messages are relayed with `L2ToL2CrossDomainMessenger#relayMessage` as the single entrypoint. We can leverage this to track gas consumption and emit a relevant execution trace.

New Event:

```solidity
event RelayedMessageGasReceipt(bytes32 indexed msgHash, bytes32 indexed rootMsgHash, address relayer, address txOrigin, uint256 cost)
```

When computing the total cost used for gas, the block's basefee is used and not the gasPrice of the transaction. This ensures the cost is a pure representation of the true cost of execution parameterized by the block and not skewed by any priority fee set to influence ordering.

```solidity
function relayMessage(...) {
    uint256 _gasLeft = gasLeft()
    _ = target.call{ value: msg.value }(message);
    uint256 gasUsed = (_gasLeft - gasLeft()) + RELAY_MESSAGE_OVERHEAD;

    uint256 cost = block.basefee * gasUsed;

    // there will always be populated context when relaying.
    (,,bytes ctx memory) = crossDomainMessageContext();
    (bytes32 rootMsgHash, address txorigin) = _decodeContext(ctx);

    emit RelayedMessageGasReceipt(msgHash, rootMsgHash, msg.sender, txorigin, cost);
}
```

Thus this receipt can be used to reliably track `tx.origin` reponsible for all cross domain calls that occur from the first transaction. With the `rootMsgHash` serving as a unique identifier to tie all nested cross domain calls with their root-most outbound message.

- **Question**: \_Might the parent message hash be useful here too?

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

There is an increase in gas cost to the `relayMessage` function:

1. Additional encoded bytes for the `SentMessage` event, used to relay the message.
2. Emission of a new event, `RelayedMessageGasReceipt`.

However the added event fields to `SentMessage` and fields of the new event, `RelayedMessageGasReceipt`, are encodings of fixed sized data fields bounding the added overhead.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

The added gas cost, if unbounded, could be a single point of failure to relaying messages since we are attaching auxilliary data to each outbound message. However as noted in the [Resource Usage](#resource-usage) section, the encoding of this data is fixed is size and cannot be exploited to include more.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

-

## Impact on Developer Experience

<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? --->

The contract API of the L2ToL2CrossDomainMessenger does not change with this introduction. Although any relayers operating on the devnet must update the their abi for the `SentMessage` event to relay.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

1. Rely on 3rdparty Bridge/Relay providers. As listed in the problem statement, this can work well for single cross-chain messages but quickly falls apart in minimally more complex use case.
2. Offchain attribution. Schemes can be derived with web2-esque approaches such as API keys. With a special tx-submission endpoint, being able to tag cross-chain messages with off-chain accounts to thus charge for gas used. This might a good fallback mechanism to have in place.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

`RelayedMessageGasReceipt` does not end up used in any meaningful gas payment system. In this world, the event can be simple be ignored. In hardfork, the event can be removed and the contexted versioned such that the added fields are removed to save on the gas overhead added from this feature.
