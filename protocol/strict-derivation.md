# Purpose

Holocene aims to introduce a set of changes to derivation that improve worst-case scenarios for
Fault Proofs and Interop. The goal of this design doc is to align on the changes to derivation that
will be part of the upcoming Holocene hardfork, and in particular to solve open questions to
finalize the specs.

The Holocene specs for these derivation changes are currently being written in [specs/#357](https://github.com/ethereum-optimism/specs/pull/357).

# Summary

The Holocene hardfork introduces several changes to batch derivation:

- _Partial Span Batch Validity_ determines the validity of singular batches within a span batch
individually, only invalidating the remaining span batch upon the first invalid batch.
- _Steady Batch Derivation_ derives invalid singular batches immediately as deposit-only
blocks.
- _Strict Batch Ordering_ requires batches within and across channels to be strictly ordered.

The design space and some proposed solutions will be discussed, together with practical implications
for batcher implementations that have to satisfy the stricter rules.

# Partial Span Batch Validity

## Problem Statement + Context

Before Holocene, Span Batch validity is handled atomically, i.e., either all singular batches that
can be derived from a span batch are valid, or if any of them is invalid (during batch queue
checks), the whole span batch is dropped. This has negative implications for Fault Proofs: in a
worst-case scenario, if the to-be-fault-proven block is the first batch in a span batch, to
determine its validity at the batch queue stage, the validity of the full span batch has to be
determined, as any later invalid singular batch would also invalidate all prior batches in the same
span batch (call this behavior "backwards invalidating"). So only invalidating the remaining span
batch upon hitting an invalid derived singular batch (call this behavior "forwards invalidating")
vastly improves this scenario, since batch queue validity is then final for any singular
batch, not just regular singular batches.

Note that forward-invalidation is natural for span batches because of its data structure: Singular
batches within span batches don't explicitly state the parent hash, so they have to be assumed to
implicitly reference the parent singular batch. So if that parent singular batch is invalid and
replaced by an empty batch, for consistency reasons, all future singular batches in a span batch
have to be considered invalid as well.

In late 2023 while Delta was still in development, the problem of partial span batch validity was
[already discussed](https://www.notion.so/oplabs/Handling-Invalid-Batches-48bb972368ce409c84bd9189eddd0577)
and for simplicity it was decided to use the atomic validity model. However, above discussion
focusses on implications of the atomic validity model for EL-sync. We're now concerned with behavior
around CL client restarts and reorgs.

## Proposed Design

To resolve open questions regarding the implementation of Partial Span Batch Validity, I propose to
apply the principle to consistently change the validation rules to be only forward-invalidating, and
thus treat a span batch like an additional (optional) stage in the derivation pipeline that sits
before the batch queue, so that the batch queue pulls singular batches from this previous span batch
stage. This way, we never backwards-invalidate the span batch, only forward.

This means that sequencer drift validations are applied individually to each singular batch derived
from the span batch. On a failed check, due to Steady Batch Derivation, the singular batch is
immediately derived as a deposit-only block, and the remaining span batch is invalidated.

There are still checks that naturally apply to the whole span batch due to its data structure:
- A span batch only has one parent reference, since all internal batch references are implicit.
Therefore, a failed parent check invalidates the whole span batch.
- A failed sequencing window check will always apply to the oldest batches in a span batch, so a
failed check of the first batch will invalidate the full span batch.
- If `span_end.timestamp < next_timestamp` (as defined in the Span Batch specs), the whole batch is
dropped, as it doesn't contain any new batches (this would also happen if applying timestamp checks
to each derived singular batch individually).

Those checks are done once per span batch before the batch queue derives any singular batches.

### Mid-way restart

One concern that was raised with partial span batch validity is how to deal with mid-way restarts,
i.e. when the CL client restarts, losing its internal derivation pipeline state while the safe head
of the EL client is in the middle of a span batch. In particular, the correct behavior needs to
cover the case where a L1 reorg happened in the meantime that reorged out the L1 block that
contained the span batch.

I fail to see currently how partial span batch validity makes this case worse or different. With
atomic span batch validity, there's the pending-safe state of derived blocks that might be reorged
out later by a backwards-invalidating singular batch derived from a span batch. So if we only
forwards-invalidate, this becomes easier.

And in any case, a restart or reorg needs to reconstruct the derivation pipeline state from the new
(sync-start) L1 block, so I don't understand how reconstructing span batch derivation state is
different from reconstructing the channel bank and batch queue in general. If a restart or reorg
happens in the middle of deriving singular batches from a channel, we're in a similar situation that
is solved by going back far enough for the sync-start.

# Strict Batch Ordering

Strict batch ordering requires batches within and across channels to be strictly ordered.
This removes the necessity to buffer more than one channel, and the channel bank effectively holds
at most one staging channel, for which it collects frames, that must arrive in order.

The following will explore the open design space, presenting options, and propose a solution.
In order to guide decisions on open design questions, we will consider the 
_principle of fastest derivation_: the new rules should lead to faster derivation progress where possible,
even if it means deriving empty batches earlier. 
Spoiler: Option 1 will always be the proposed option.

## Out of order frames

There's an open design space around how to handle some scenarios for missing or out of order frames:
- How do we handle "foreign first frames" -- namely, incoming _first_ frames which do not belong to the current channel?
  - Option 1: discard the current channel and open a new one for the given frame.
  - Option 2: discard the foreign frame and wait for the duration of the channel timeout for the
    next correct frame and only then discard the current staging channel and open a new staging
    channel for the next first frame.
- How do we handle the current staging channel if an out of order frame (that is not the first
frame) arrives? There's only one valid option, to drop the out-of-order frame, keeping the currently
staged channel. Note that this will give the staging channel a chance to be continued or closed
properly by the next in-order frame. If this doesn't happen, at the latest the next first frame
will drop the staging channel and start a new one.

### Proposed solution

Following the principle of fastest derivation, this means choosing option 1. This has the
additional advantage that the sync start algorithm simplifies significantly, because a fist frame
_always_ marks the start of a new channel, and guarantees that there won't be any buffered channels
in the channel bank.

## Out of order batches

The batch queue also becomes simpler as batches have to arrive in order. 

- How to treat _older_ batches is easy: they are dropped.
- Similarly to out of order frames, how to treat future batches (gaps) has two options:
  - Option 1: immediately fill the gap with deposit-only blocks, and then process the batch.
  It is noteworthy that the future batch can then only be valid if it built on top of a gap of empty batches.
  - Option 2: drop future batches, and wait for up to a full sequencing window for the next
  batch to arrive.
  - Note that the pre-Holocene behavior is to cache _future_ batches for later checking when the gap
  has been closed. It is one goal of Holocene to remove as much buffering as possible, so this
  behavior is not considered a valid option.

### Proposed solution

Following the principle of fastest derivation, option 1 is proposed to be chosen. Besides guaranteeing
faster derivation, it also avoids waiting for full sequencing windows on gaps to be resolved, which
is quite unlikely, unless the batcher implementation would be specially tuned to backfill gaps. If a
gap occurs, it is actually much more likely that the next batches will be even further in the
future, so option 2 would realistically almost always be a bad choice. Plus, the sync start
algorithm benefits from option 1 as well, as it leads to faster resolution of uncertainty caused
by gaps.

It should be noted that if the blocks after the gap actually come from a chain where the gaps
aren't deposit-only blocks, then these blocks would have mismatched parent hashes and be likely
discarded. This is intentional. This design allows for not having to submit empty batches to
derive deposit-only blocks, but instead just pick the next non-deposit-only block.

Writing this, I realize that this mechanism could even be used to encode large spans of empty
batches, as long as the sequencer is creating unsafe blocks that follow the same L1 origins as the
auto-derivation would for gaps. However, this would need to be investigated more deeply.

Note that the new strict ordering rules of the batch queue will always lead to an empty batch queue
when the origin of the derivation pipeline progresses to the next L1 block.
In order to guarantee this invariant fully, we might need to add an extra rule to the derivation
pipeline to only progress to the next L1 block for derivation if all `undecided` batches have been
decided upon by retrieving the previously temporarily missing L1 and L2 blocks.

## Batcher hardening

In a sense, Holocene shifts some complexity from derivation to the batching phase. Simpler but
stricter derivation rules need to be met by a more complex batcher implementation.

The batcher must be hardened to guarantee the strict ordering requirements. They are already mostly
met in practice by the current implementation, but more by accident than by design. There are edge
cases in which the batcher might violate the strict ordering rules. For example, if a channel fails
to submit within a set period, the blocks are requeued and some out of order batching might occur.
We also need to take extra care that dynamic blobs/calldata switching doesn't lead to out of order
or gapped batches in scenarios where blocks are requeued, while future channels are already waiting
in the mempool for inclusion.

I propose to let the batcher follow a fixed nonce to block-range assignment, once the first batcher
transaction (which is almost always the only batcher transaction for a channel) starts being
submitted. This should avoid out-of-order or gapped batches. It might require to implement some form
of persistence in the transaction manager, since we cannot reliably recover all globally pending
batcher transactions in the L1 network.

Furthermore, the batcher will need to be made aware of the Steady Batch Derivation rules, namely that
invalid batches will be derived as deposit-only blocks. So in case of a reorg, the batcher should
e.g. wait on the sequencer it is connected to until it has derived all blocks from L1 in order to
only start batching new blocks on top of the possibly derived deposit-only chain segment.

And finally, there must be some detection of unexpected deposit-only chain derivation, e.g. by
querying the sync-status repeatedly and matching it against the expected safe chain. As described
above, in case of any discrepancy, the batcher should then stop batching and wait for the sequencer
to fully derive up until the latest L1 batcher transactions, and only then continue batching.

## Sync Start Simplifications

Strict Batch Ordering leads to a simpler sync start algorithm. It may be sufficient to walk back
the L1 chain until finding a batcher transaction that starts a new channel, i.e., containing a first
frame. The strict ordering rules, as detailed above, would in such a case drop any pre-existing
staging channel, and the batch queue should always be empty right after the pipeline origin
progresses.

# Steady Batch Derivation

Steady Batch Derivation changes how invalid batches and payload attributes at different stages in
the derivation pipeline are treated.
- Batch queue: if a batch is found to be invalid at the batch queue stage (`BatchDrop`), it is immediately replaced by an empty
  batch, leading to deposit-only (first batch of an epoch) or empty payload attributes.
- Engine queue: if the payload attributes are found by the EL client to be
  [`INVALID`](https://github.com/ethereum/go-ethereum/blob/7fd7c1f7dd9ba8d90399df2f080e4101ae37a255/beacon/engine/errors.go#L63)
  they are replaced by deposit-only/empty payload attributes and reinserted into the EL.

## L1 origin selection

There's an open design space about how to select the L1 origin when deriving one or more
deposit-only blocks.

- A simple default rule could be to generate blocks in a way to maintain a steady L1/L2 block ratio,
e.g. bumping the L1 origin selection every 6 blocks in the case of mainnet.
- Edge case: we were already very near the sequencer drift limit, and need to select a new origin
faster.
- Edge case: L1 missed a slot, and the L1 origin cannot advance as expected.
- So maybe a better rule is to first eagerly advance the L1 origin as quickly as possible, and only
if a newer L1 origin isn't available, keep it. This solves for missed slots and will implicitly and
automatically maintain a good L1/L2 block ratio. We then just need a clear definition of "L1 origin
is/isn't available".
  - To mimic sequencer behavior, and avoid being hit by shallow L1 reorgs, we could add an
  in-protocol L1 validation depth. So eagerly advancing the L1 origin while maintaining a timestamp
  distance of this validation depth times the L1 bock time.

# Activation rules & actions

I propose to activate the new rules by L1 inclusion block timestamp, which is most convenient and clean
for derivation protocol upgrades. It's accessible in most parts of the derivation pipeline as the
stage's _origin_ (not to be confused with the _L1 origin of an L2 block_).
Note that this is in contrast to how Delta activated span batches, which was by first parsing the
span batch and then checking the span batch's L1 origin timestamp.

However, we need to specify how existing buffered frames, channels and batches are treated at
activation, since the stricter rules of Holocene take effect.
To start with a derivation pipeline state that is consistent with Holocene rules, I propose the
following special activation effects to take place when the pipeline's origin reaches the activation
timestamp, which is similar to applying Holocene rules retroactively.
- Channel Bank:
  1. All channels that don't consist of contiguous frames that include the first, are dropped. 
  2. All but the youngest (in terms of L1 inclusion) channel are dropped.
  3. This optional left-over channel
     will be the start staging channel for the Holocene channel bank.

The principle of eager derivation of later stages should guarantee at this point that all batches that could have
been applied directly on top of the current safe chain have already been exhausted from the batch
queue. So what is left should be `undecided` and `future` batches. (Note: I'm not sure on this one.)
I see two options on how to handle a non-empty batch queue at this point:
- Option 1: When the L1 origin reaches the Holocene activation block, discard all batches in the
batch queue. Could also be applied to the Channel Bank to give us a nice clean starting point.
  - The batcher would have to be aware of that rule and consider any blocks it submitted in channels
  that didn't close prior to the Holocene activation block as needing to be resubmitted.
- Option 2: Drop future batches, continue resolving undecided batches, if any are left, and apply
new Holocene rules.
- Option 3: The Batch Queue will just start applying Holocene rules from this moment onwards. This will then
automatically derive gaps as empty batches, discard out of order batches, etc. A problem with this
option is that the sync start for a while after Holocene would still need to apply the pre-Holocene
algorithm to restore the derivation pipeline state.

At this point I prefer Option 1. I'd like to pick the one that leads to a
simpler implementation, minimizing complexity where possible. Since almost always a healthy batcher
implementation during normal network conditions will already produce batches according to Holocene
rules, we don't need to optimize for the prettiest activation rules, as long as they work
consistently.

Another open question is how to handle span batches that come from pre-Holocene channels.
I propose that, for simplicity of implementation, if they are found to be valid as a span batch, to
just apply the new Partial Span Batch Validity rules even though those span batches were derived
from pre-Holocene L1 blocks.

# Alternatives Considered

We could implement only some of the three changes to derivation. However, the synergies among the
changes make it worthwhile to apply them at once. E.g. strict batch ordering simplifies several
stages in the derivation pipeline, thereby simplifying the reasoning about aspects like the right
sync start behavior, which helps in implementing partial span batch validity more easily.
Furthermore, the structural changes necessary in the derivation pipeline to apply multiple subsets
of changes over multiple hardforks would create more work that can be saved if applying all those
derivation changes in one go.

# Risks & Uncertainties

## Unsafe reorgs

Before Steady Batch Derivation, invalid batches got chances to be replaced by valid future batches.
Because they will now be immediately derived as deposit-only blocks, there is a theoretical
heightened risk for unsafe head reorgs. To the best of our knowledge, we haven't
experienced this on OP Mainnet or other mainnet OP Stack chains yet.

The current rules of Steady Batch Derivation can indeed lead to a previously valid submitted batch
to now be invalid, and then immediately be derived as deposit-only blocks, causing a long L2 unsafe
reorg. The same batcher tx may still be included on L1. However, given that span batches are only
forward-invalidated with Holocene, I think the L2 unsafe reorg would be limited to the L2 section
that references the reorged-out L1 section. However, more batcher txs that might already have landed
on L1 as well would cause more deposit-only blocks to be derived.

I think this is the trade-off of Steady Batch Derivation.

### Last L1 origin of channel

One solution that I can think of that may alleviate this problem is to reference the last L1 origin
in the channel metadata (in a new channel format), and then drop the channel directly in the channel
bank before even decoding any batches from it, that would then at some point be derived as
deposit-only blocks. This way, the batcher could get a "second chance" to submit a channel that
includes the correct reorged-to L1 origin chain. This is very similar to how span batches contain
the last L1 origin as `l1_origin_check` in their prefix, just moved one layer up to the channel
container. Having the channel contain such an L1 origin check has the advantage that the derivation
pipeline wouldn't too eagerly derive deposit-only blocks, and to recover from L1 reorgs. I think
this solution would also still maintain the nice properties of Strict Batch Ordering, that there's
only one staging channel, and that we don't buffer out of order frames or batches. However, this is
also the disadvantage that it violates the above stated principle of fastest derivation.

If Proofs and Interop experts can confirm that such a solution would still lead to the sought after
improvements for Proofs and Interop, resp., we could consider including it as part of Holocene.

## Multiple batchers & strict ordering

There are ideas to allow multiple batchers to send batcher transactions, like
[Event Based Batch Inclusion](https://github.com/ethereum-optimism/specs/issues/221).
Strict batcher ordering actually benefits from having only a single batcher address, since nonce
ordering guarantees transaction inclusion order. When we add some form of multi-batcher support, we
need to solve correct transaction inclusion ordering from multiple batcher addresses. Note that
during phases of high throughput, the batcher may send multiple transactions in parallel and then
relies on the nonce ordering for correct inclusion order of multiple transaction in the mempool.
