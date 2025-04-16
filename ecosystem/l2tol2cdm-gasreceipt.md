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

The general mental model for a transaction is that the sending account pays for the gas used. This mental model is the backbone for 4337 or 7702 sponsored transactions where tx.origin is decoupled from the calling account. By providing a mechanism for which the accumulated gas used for all cross domain calls with the initial `tx.origin` can be verifiably tracked, it provides the foundation for a single-tx experience.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Today, a single transaction cross-chain experience is achieved by pushing gas from the source -> destination chain with the message. Relayers are incentivized as long as the pushed funds for gas is sufficiently enough to cover the cost. This works in primitive usecases, where there's a single message from A -> B such as bridging funds. However even this has several drawbacks:

- **Gas Estimation**. Doing on-chain estimation is very inaccurate, resulting in overestimation for high odds of delivery. Some protocols/relayers even pocket this difference instead of refunding the account. And if the account is refunded, it is dust on the remote chain for the user.
- **Relayer Vendor lock-in**. Contract developers must lock into a crosschain provider (Hyperlane,LayerZero,Wormhole) in order to make use of these services. This tight coupling makes it nearly impossible to switch without introducing complexity (upgradable smart contracts and provider-agonstic interfaces).

Stepping outside this simple usecase, we can enumerate more scenarios in which pushing gas from source -> destination quickly falls apart.

1. **Transitive cross-domain calls**. A -> B -> C. Gas estmation from the source chain (A), is already inefficient in the single hop. This scheme must also additionally overestimate and provide enough gas for each cross-domain call, B -> C. If a cross-domain call was to fail in an intermediate hop, it's unclear how to fund that transaction through from A.

2. **Cross-Domain Composability**. A single transaction can fan out calls to multiple chains which then calls back to the originating contract, (A -> B -> A) & (A->C). For example, a Swap -> Bridge & Swap -> Bridge. Or two cross-chain DeFi protocols integrating each other. This type of cross-chain communication does not work in the world of relayers today.

   - For applications composing each other (A & B), we can not gaurantee the same relayer vendor is used. Hop A->B might leverage Hyperlane to send the message, while B might transitively use Wormhole to send a message to C. **This is why it's important to solve this problem natively**, such that as long as the providers (LayerZero/Wormhole/Hyperlane) use native message passing, everything works.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

    Providing a mechanism for which the accumulated gas used for all cross domain calls can be correlated with the original `tx.origin`.

By natively providing this in the messenger, anyone can charge tx senders exactly what was used for all incurred cross domain messages. It is important to note here that this design doc does not prescribe a solution for charging this gas. This can happen within shared block building, enshrined in a app-level relay bounties framework, or used the 3rdparty providers in their own accounting systems.

Support can be enumerated in 2 parts.

### 1. Propogated SentMessage "Headers" or "Context".

With nested cross domain calls, there's no way to correlate the subsequent messages with the original `tx.origin`, as each message has a unique message hash associated with it. By introducing a new field, "headers" or "context" that are appropriately propogated as nested cross domain calls are made, we can introduce contextual information unrelated to the cross domain message itself.

```solidity
event SentMessage(uint256 indexed destination, address indexed target, uint256 indexed messageNonce, address sender, bytes message, bytes context);
```

This added context should be versioned such that the propogated information can evolve over time to fit various needs. In this first iteration, we propose encoding the root-most `messageHash`, the `tx.origin` of that root call, and `call depth` of the cross domain message. This information is made availble in transient storage when relaying a message so that any further outbound messages simply forwards the appropriate context, rather than repopoulate an entirely new one.

Some pseudocode describing the above, the exact api will be fleshed out in a specs PR.

```solidity
function sendMessage(...) {
    (,,bytes ctx memory) = crossDomainMessageContext();
    if (ctx.length == 0) {
        // new "top-level" cross domain call (messageHash_ == outbound message)
        ctx = abi.encodePacked(version, abi.encode(messageHash_, tx.origin, 0));
    } else {
        // propogate, incrementing call dpeth
        (bytes32 rootMessageHash_, address txorigin, uint256 depth) = _decodeContext(ctx);
        ctx = abi.encodePacked(version, abi.encode(rootMessageHash_, txorigin, depth+1))
    }

    ...

    emit SentMessage(..., ctx)
}

function relayMessage(...) {
    (uint256 source, address sender, bytes memory ctx) = decodeSentMessage(...)
    _storeMessageMetaData(source, sender, ctx);
}
```

Although first use of this propogated context is for gas receipts, this versioned abstraction can prove useful for much more information in the future without breaking the signature of `SentMessage`, critical for not breaking the call flow between `sendMessage` and `relayMessage`.

### 2. Gas Receipt

All superchain interop cross domain messages are relayed via the `L2ToL2CrossDomainMessenger#relayMessage` function. We can leverage this single-entrypoint to track the gas consumption and emit relevant execution information.

We include the `block.basefee` and not the `tx.gasprice`, so that the priority fee (included in gasPrice) set by the relaying entity is not included and only the minimum execution costs are computed.

- **Question**: _Should we simply emit the computed cost here rather than the individual parts? Would emitting the gasPrice be useful for someone else building on top?_

New Event:

```solidity
event RelayedMessageGasReceipt(bytes32 indexed msgHash, bytes32 indexed rootMsgHash, uint256 indexed depth, address indexed txOrigin, uint256 baseFee, uint256 gasUsed)
```

```solidity
function relayMessage(...) {
    uint256 _gasLeft = gasLeft()
    _ = target.call{ value: msg.value }(message);
    uint256 gasUsed = _gasLeft - gasLeft();

    // there will always be populated context when relaying.
    (,,bytes ctx memory) = crossDomainMessageContext();
    (bytes32 rootMsgHash, address txorigin, uint256 depth) = _decodeContext(ctx);
    emit RelayedMessageGasReceipt(msgHash, rootMsgHash, depth, txorigin, block.basefee, gasUsed + RELAY_MESSAGE_OVERHEAD);
}
```

Thus this receipt can be used to reliably track `tx.origin` reponsible for all cross domain calls that occur from the first transaction. And with `rootMessageHash`, obtaining a unique identifier to tie all nested cross domain calls with their root-most outbound message.

- **Question**: _Might the parent message hash be useful here too? It could easily be fetched via the a gas receipt where the rootMsgHash matches and the depth is 1 behind. These are indexed topics making it an efficient getLogs query._

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

1. Rely on 3rdparty Bridge/Relay providers. As listed in the problem statement, this can work well for single cross-chain messages but quickly falls apart to any complex use case.
2. Offchain attribution. Schemes can be derived with web2-esque approaches such as API keys. With a special tx-submission endpoint, being able to tag cross-chain messages with off-chain accounts to thus charge for gas used. This might a good fallback mechanism to have in place.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

`RelayedMessageGasReceipt` does not end up used in any meaningful gas payment system. In this world, the event can be ignored and even removed in a future hardfork. The serialized context propogated with every SentMessage can also be versioned to empty bytes to reduced the overhead that was added. It is important to note here that serialized context is an added feature irrespective of the gas receipt that can be re-purposed to propogate _any_ useful context.
