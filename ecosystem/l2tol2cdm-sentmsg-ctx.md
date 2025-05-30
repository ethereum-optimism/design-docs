# [L2ToL2CrossDomainMessenger SentMessage context]: Design Doc

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

As more contracts are built that natively integrate with superchain interop, several single-action experiences can span up to N+1 op-stack chains to achieve their desired outcome. As multiple cross domain message are spawned from single transactions, being able to propogate contextual information unrelated to the core message, like headers in HTTP, is important to be able to tie subsequent cross domain messages together.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

We make space for internal contextual information by adding opaque bytes to each `SentMessage` event. The `L2ToL2CrossDomainMessenger` can internally take care of populating and propogating this context on nested cross domain messages.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

As is, the `SentMessage` event explictly emits some contextual information along with the core message, i.e `messageNonce`. If additional contextual information is required, a breaking abi change must be made in order to include it.

This leaves no room for flexibilty for including new contextual information without making a more complex breaking change to the messenger.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

Introudce opaque context bytes field in `SentMessage`. This field MUST be versioned such that it can evolve with new or removed fields.

```solidity
event SentMessage(uint256 indexed destination, address indexed target, uint256 indexed messageNonce, address sender, bytes message, bytes originContext);
```

This proposal simply includes this field without specifying the first versioned format. By leaving it as empty for now, we can populate the first version in a later
design that requires it.

- See [RelayedMessageGasReceipt](https://github.com/ethereum-optimism/design-docs/pull/282).
- See [incentivized message delivery](https://github.com/ethereum-optimism/design-docs/pull/272).

**Note**: The context itself **MUST** be included in the pre-image of the cross domain message hash. While `CrossL2Inbox` validation ensures log integrity, we must include the context in the preimage in order for `resendMessage` which re-emits stale `SentMessage` event to gaurantee that context hasn't been tampered with. The pre-image hashing process begins by computing the messagePayloadHash from the core message payload. The originContext is then constructed by encoding the ORIGIN_CONTEXT_ENCODING_VERSION alongside the messagePayloadHash. Finally, the messageHash is derived by hashing the messagePayloadHash and the originContext together.

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

There's a small added gas cost to emitting the `SentMessage` event with an extra field. In this proposal, the field is empty bytes so the overhead is most minimal. As further designs encode information into this context, resource usage must be bounded as the gas overhead applies to every outbound message and executing relay transaction.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

1. Resource Usage
2. Appropriate versioning on the context to not break in-flight messages

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

-

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

To avoid breaking the abi, the context could be encoded within the `message` field. However the `message` is not currently version and convolutes the purpose of the field (raw target calldata). Keeping these fields explictly seperate makes the overall cross domain message much more readable and decodable.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

No meaningful usage of any metadata in context. In the setting the context can be versioned such that it is always empty, minimizing gas overhead. This flexibility in this field is a strength, as the parameter can always be re-purposed or extended to new usecases.
