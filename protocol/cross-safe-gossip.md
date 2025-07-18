# [interop]: Following Mode

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _@clabby_                                          |
| Created at         | _2025-06-24_                                       |
| Initial Reviewers  | _Protolambda, Adrian Sutton_                       |
| Need Approval From | _Mark Tyneway_                                     |
| Status             | _Draft_                                            |

## Purpose

The purpose of this design doc is to specify a potential new "mode" for the [rollup node][rollup-node] to operate in a
post-interop world, that alleviates the need to reach out to an `op-supervisor` to determine [cross-safety][cross-safe].

## Summary

This design doc introduces an alternative design for "following mode," where a [rollup node][rollup-node] operator
trusts a sequencer's determination of
[message validity](https://specs.optimism.io/interop/messaging.html#invalid-messages) in order to advance its
[`cross-safe`][cross-safe] chain.

## Problem Statement + Context

For the vast majority of [Rollup Node][rollup-node] operators, operating a [supervisor][supervisor] is costly and
undesirable. The benefit of operating one's own [supervisor][supervisor] is primarily to _ensure_ that the messages
included in blocks by the sequencer are indeed valid. While this is a necessary step to _ensure_ an L2 chain's integrity
post-interop, there are valid user stories that don't involve the costly validation of message validity, beyond trusting
an _attestation_ from the sequencer that it has already been done.

For operators that are willing to trust the sequencer's attestations, an alternative mode can be introduced which
bypasses the need for a [rollup node][rollup-node] to directly consult a [supervisor][supervisor] when determining
[cross-safety][cross-safe].

## Proposed Solution

"Following Mode," at a high level, consists of [rollup nodes][rollup-node] subscribing to a new gossip topic that
receives _signed [`cross-safe`][cross-safe] blocks_.

1. When the sequencing node consults with its own [supervisor][supervisor] and determines that a block can be promoted
to [`cross-safe`][cross-safe], it will sign the payload hash of the new [cross-safe][cross-safe] block as specified in
["Rollup Node P2P"](https://specs.optimism.io/protocol/rollup-node-p2p.html#block-signatures) and gossip the
[signed payload envelope](https://specs.optimism.io/protocol/rollup-node-p2p.html#block-encoding) to peers
over the [`cross-safe-blocksv1` topic](#new-gossip-topic).
1. Peers observing the topic that are operating in "Following Mode" will then validate the block as specified in
["Rollup Node P2P"](https://specs.optimism.io/protocol/rollup-node-p2p.html#block-validation), and if validation succeeds:
    1. Check that the block has been derived as `local-safe`. If not, ignore the block.
    1. Perform [L1 consolidation](https://specs.optimism.io/protocol/derivation.html#l1-consolidation-payload-attributes-matching)
    on the derived attributes, relative to the received block.
        1. If L1 consolidation fails, reduce the `transactions` array in the derived attributes to only deposits
        following the [interop block replacement rules](https://specs.optimism.io/interop/derivation.html#replacing-invalid-blocks)
        , and try again.
            1. If finally, L1 consolidation fails after reducing the attributes to deposits-only, discard the block.

### New Gossip Topic

The new gossip topic that is introduced is `cross-safe-blocksv1`, broadcasted on
`/optimism/<chainId>/0/cross-safe-blocks`. Block encoding for the `cross-safe-blocksv1` topic is as specified within
["Rollup Node P2P - blocksv4"](https://specs.optimism.io/protocol/rollup-node-p2p.html#block-encoding).

Peers that broadcast invalid blocks on this topic, per the block validity rules mentioned above, should be downscored
to mitigate possible DoS.

### Resource Usage

Resource utilization for this new mode should be minimal. A new gossip topic will need to be subscribed to among
rollup node participants operating as a sequencer or in "Following Mode," and the sequencer will need to both sign
and broadcast blocks as it promotes them to [`cross-safe`][cross-safe].

### Single Point of Failure and Multi Client Considerations

This change affects only the [rollup node][rollup-node], and will need to be implemented in both
[`op-node`](https://github.com/ethereum-optimism/optimism/tree/develop/op-node) and
[`kona-node`](https://github.com/op-rs/kona). It is invisible to the execution layer.

## Failure Mode Analysis

_TODO_

## Impact on Developer Experience

n/a - change affects node operators.

## Alternatives Considered

_TODO_

## Risks & Uncertainties

_TODO_

[cross-safe]: https://specs.optimism.io/interop/verifier.html#safe-inputs
[supervisor]: https://specs.optimism.io/interop/supervisor.html#supervisor
[rollup-node]: https://specs.optimism.io/protocol/rollup-node.html
