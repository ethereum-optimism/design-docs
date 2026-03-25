# Drop Overlapping Span Batches: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Sebastian Stammler (seb@oplabs.co)                 |
| Created at         | 2026-03-18                                         |
| Initial Reviewers  | @ajsutton, @theochap, @inphi                       |
| Need Approval From | @ajsutton                                          |
| Status             | Draft                                              |

## Purpose

This design doc proposes dropping overlapping span batches at the Karst hardfork and explores how to
maintain fault proof soundness under this new rule.

The motivations are:

1. **Simplify the derivation pipeline.** The current span batch overlap verification requires the
   `SafeBlockFetcher` interface to look up historical L2 blocks during batch validation. This
   complicates building a pure derivation pipeline where all L1 inputs are fetched in advance and IO
   is moved outside the pipeline. Removing the overlap check simplifies the derivation pipeline
   refactor drafted in [PR #19303](https://github.com/ethereum-optimism/optimism/pull/19303).
   Alternatively, the overlap check could be retained with a two-pass approach where the pipeline
   returns the required L2 blocks and the caller provides them (see approach 3 below), but this
   carries the overlap checking complexity into the new implementation.

2. **Improve fault proof performance.** The current overlap verification in fault proofs requires
   K+1 preimage oracle reads (one parent block reference plus K full block payloads) and O(K×T)
   byte-by-byte transaction comparison, where K is the number of overlapping blocks and T is the
   average number of transactions per block. Any solution that eliminates the overlap check also
   eliminates this cost, reducing on-chain dispute cost, off-chain FP VM execution, and ZK proving
   costs.

## Summary

**The derivation rule change is straightforward:** overlapping span batches are dropped (`BatchDrop`)
instead of verified-and-accepted post-Karst. The `SafeBlockFetcher` interface is removed from batch
checking entirely.

Overlapping span batches exist to handle a race condition during sequencing window elapse, where
auto-derivation races with submitted batches. This is an unlikely scenario, and the batcher can
temporarily switch to singular batches as a fallback. This makes removal of overlapping span batch
acceptance safe from a liveness perspective.

**The fault proof interaction is an open design question.** The dispute game bisects to arbitrary L2
blocks, which can place the safe head in the middle of a span batch. Under the new drop rule, the
span batch that originally produced those blocks would be dropped — breaking derivation. This doc
explores nine approaches to solving this, grouped by the depth of protocol change required. None are
fully satisfactory; the trade-offs are presented for discussion.

## Problem Statement + Context

### Current overlap mechanism

When a span batch's first block timestamp is earlier than the next expected L2 timestamp
(`batch.GetTimestamp() < nextTimestamp`), the span batch overlaps with the safe head. The derivation
pipeline currently handles this with a two-stage verification:

1. **Prefix check** (`checkSpanBatchPrefix` in `op-node/rollup/derive/batches.go`): Calculates the
   parent block number at the overlap boundary and calls `l2Fetcher.L2BlockRefByNumber()` to fetch
   the parent block. Validates that the span batch's parent hash matches.

2. **Full overlap check** (`checkSpanBatch`): For each overlapping block, calls
   `l2Fetcher.PayloadByNumber()` to fetch the full execution payload. Compares:
   - Non-deposit transaction count (must match)
   - Each transaction byte-for-byte (must match)
   - L1 origin number (must match)

The `SafeBlockFetcher` interface provides two methods: `L2BlockRefByNumber` and `PayloadByNumber`.
This interface is threaded through the batch checking call chain from the derivation pipeline.

### Why the overlap check complicates the pure derivation pipeline

The derivation pipeline should ideally depend only on L1 data and the current safe head, with all IO
moved outside the pipeline. The overlap check complicates this by requiring lookback up to a
sequencing window into already-derived L2 history — fetching block references and full payloads for
blocks that are behind the safe head.

This does not strictly prevent a non-IO pipeline — the pipeline could return a list of required L2
blocks and let the caller provide them (approach 3 below). But this carries the overlap checking
logic and `SafeBlockFetcher` interface into the new implementation, adding complexity that dropping
overlapping batches would eliminate entirely.

### The fault proof complication

Simply dropping overlapping span batches in the fault proof program does not work. The dispute game
bisects to arbitrary L2 blocks above the `SPLIT_DEPTH` (bisecting over output roots). The agreed
prestate can place the safe head at any block, including in the middle of a span batch.

If the safe head is mid-span-batch and the overlapping span batch is dropped, the batch that
originally produced the safe head's blocks would be dropped — and derivation cannot continue.

The fault proof's agreed output root (`OutputV0`) captures L2 execution state (state root, block
hash) but **not** derivation state (which batch was in progress, the position within it). This gap
between execution state and derivation state is the core challenge.

### Relation to Holocene

The [Holocene derivation design doc](holocene-derivation.md) introduced several relevant changes:

- **Partial span batch validity** with forward-invalidation only: if a singular batch within a span
  batch is invalid, only subsequent batches are invalidated, never prior ones. This principle of
  never backwards-invalidating batches is preserved by the Karst change.
- **Strict batch ordering**: Batches within and across channels must be strictly ordered. This
  simplifies reasoning about which batch "owns" which block range.

## Proposed Derivation Rule Change

Post-Karst, in `checkSpanBatchPrefix` / `checkSpanBatch`:

- If `batch.GetTimestamp() < nextTimestamp`, return `BatchDrop`.
- The entire overlap verification codepath is removed.
- The `SafeBlockFetcher` parameter is removed from the batch checking functions.
- Gated by `cfg.IsKarst(l1InclusionBlock.Time)`, following the same pattern as Holocene checks in
  `batches.go`.

**Overlap semantics:** This applies to partially overlapping batches (start before safe head, end
after) and fully overlapping batches (entirely before safe head). The existing Holocene rule for
`span_end.timestamp < next_timestamp` (the whole batch is in the past) remains unchanged — Karst
changes the handling of the partial overlap case where
`span_start.timestamp < next_timestamp <= span_end.timestamp`.

**Activation semantics:** A span batch is processed under pre-Karst or post-Karst rules based on its
L1 inclusion block timestamp, not on individual L2 block timestamps. Channels that straddle the Karst
activation boundary: the L1 inclusion block of the batch (not the channel's first frame) determines
which rules apply.

Batchers that submit batches before Karst that get included after Karst will have their overlapping
batches dropped. Sane batchers never produce overlapping span batches anyways.

### Resource usage

| Scenario | Current (overlap verification) | Post-Karst (drop) |
|----------|-------------------------------|-------------------|
| Normal derivation | O(K) network IO + O(K×T) comparison | O(1) drop |
| Fault proof VM | K+1 preimage reads + O(K×T) comparison | Depends on approach (see below) |

### Multi-client considerations

The derivation rule change applies to all clients that implement the derivation pipeline:

| Component | Change Required |
|-----------|----------------|
| op-node (Go) | Drop overlapping span batches post-Karst, remove `SafeBlockFetcher` from batch checking |
| kona (Rust) | Same derivation rule change |
| op-program | Same derivation rule change (shares op-node derivation code) |
| Derivation spec | Document new Karst rule |

The fault proof impact depends on which approach is chosen for the open question below.

## Open Question: Fault Proof Soundness

The derivation rule change creates a problem for fault proofs: how does the fault proof program
derive blocks when the safe head is mid-span-batch and the overlapping span batch is dropped?

The following approaches were explored, grouped by the depth of protocol change required. Each
includes the specific attack scenario or limitation that prevents it from being a complete solution.

### Group 1: No protocol change (derivation-only)

#### 1. Keep overlap checks in fault proofs only

Normal derivation drops overlapping batches (Karst rule). Fault proofs keep the old verification
logic via the preimage oracle.

**Problem — rule divergence.** The fault proof would accept an overlapping span batch (after
verification) that normal derivation dropped. If another batch was accepted in normal derivation for
the same block range, the fault proof derives different blocks.

**Attack scenario:** Safe head at block N. Batch A covers blocks N to N+K (overlaps at N, dropped by
Karst rule). Batch B covers blocks N+1 to N+K (no overlap, accepted — canonical chain uses B). In
the fault proof, batch A passes overlap verification and is accepted instead of B. Different
transactions, soundness violation.

#### 2. Trim overlapping span batches without verification

Strip the overlapping prefix from span batches and accept the non-overlapping suffix without
verifying the overlap portion against the safe chain.

**Problem — unsafe.** A malicious batcher can post a span batch that overlaps with the safe head but
was actually dropped during normal derivation (because a different batch was accepted for those
blocks). Trimming would promote the dropped batch.

**Attack scenario:** Same setup as above. In a fault proof with safe head at N+K-1, blind trimming
of batch A would accept it and derive block N+K from batch A. But normal derivation used batch B for
block N+K — different transactions, soundness violation.

#### 3. Deferred L2 block fetching (two-pass derivation)

The pure derivation pipeline returns a list of L2 blocks it needs for overlap verification. The
caller fetches them, then derivation re-runs with those inputs provided. This preserves the overlap
check without coupling the pipeline to network IO.

**Trade-offs.** This is a viable approach — it achieves the goal of moving IO outside the derivation
pipeline. The pipeline itself would not implement the `SafeBlockFetcher` interface; instead, it would
return a list of required L2 blocks for overlap verification, and the caller would be responsible for
fetching them. However, the overlap checking logic must still be implemented in the new pipeline,
adding complexity. The fault proof would also still need K+1 preimage oracle reads for overlap
verification, forgoing the performance improvement.

**This raises the question of whether the derivation rule change is worth pursuing at all: if the
overlap checks must be retained for fault proofs and for pre-Karst derivation, the simplification
benefit is limited to normal post-Karst derivation, while the protocol gains a new hardfork-gated
rule and the associated cross-client coordination cost.**

#### 4. Re-derive from a span batch boundary

Ensure derivation always starts from a span batch boundary, either by re-deriving from the game's
anchor state or by constraining the dispute game to only bisect to span batch boundaries.

**Problem — too expensive.** Span batches can contain thousands of blocks, and fault proof block
execution must be over a single block. Re-deriving from a span batch boundary requires executing all
intermediate blocks up to the disputed block. Constraining the dispute game's bisection to span batch
boundaries does not help either — bisection above the split depth creates intermediate starting
points at arbitrary L2 blocks regardless of the anchor state.

### Group 2: Span batch format change

#### 5. Merkle root of block hashes in span batch header

Include a merkle root over all L2 block hashes produced by the span batch. During fault proofs,
verify via merkle proof that the safe head block hash is at position K within the batch.

**Problem — violates Holocene principle.** The merkle root must be consensus-critical (verified during
normal derivation) to prevent a malicious batcher from posting wrong merkle roots. But verification
requires comparing the merkle root against block hashes computed *after* execution — this is
post-execution validation that could backwards-invalidate a batch, violating the Holocene principle
of never backwards-invalidating batches.

### Group 3: Output root format change

#### 6. Streaming hash in the output root

Track a rolling hash of the span batch encoding during derivation and include it in the output root
(`OutputV0`). The fault proof extracts it from the agreed output root and matches it against span
batches.

**Problem — too heavy a protocol change.** Modifying the output root format affects every component
that computes or verifies output roots: dispute games, proposers, challengers, bridges, and all
clients. Disproportionate to the problem being solved.

### Group 4: Dispute game change

#### 7. Host-provided cursor without new local preimage

The cannon host computes a "span batch cursor" (identifying which batch produced the safe head and
the position within it) and provides it via preimage hints.

**Problem — not verifiable on-chain.** Preimage hints are not committed to in the dispute game
contract. Local preimages must be verifiable on-chain during dispute resolution. Without a new local
preimage index, a malicious host could provide a wrong cursor with no on-chain recourse.

#### 8. Span batch cursor as a new local preimage

Add a new local preimage index to the dispute game — the "span batch cursor" — that identifies which
span batch produced the safe head and the position within it. The fault proof program reads the
cursor and uses it to skip the overlapping prefix of the identified batch.

The cursor would encode the span batch's L1 position (L1 block number, channel index, batch index)
and the block offset within the batch. Both dispute parties compute the cursor deterministically from
the agreed L1 head and L2 output root.

**Problem — local preimages are contract-computed.** Existing local preimages (`L1_HEAD`,
`STARTING_OUTPUT_ROOT`, `DISPUTED_L2_BLOCK_NUMBER`, `CHAIN_ID`) are all computed by the
`FaultDisputeGame` contract itself in `addLocalData()` from on-chain state. The contract does not
trust either party — it derives the values. The span batch cursor requires derivation state that
does not exist on-chain, so the contract cannot compute it. This breaks the trust model of local
preimages.

#### 9. Canonical span batch hash as a dispute game input

Hash the span batch encoding and add it as a new dispute game input to identify the correct batch.

**Problem — same as cursor (approach 8) but more rigid.** This effectively requires the same dispute
protocol change (adding a new input) with the same trust model issue: the contract cannot compute the
hash from on-chain state. The encoding is also less flexible than a cursor.

## Impact on Developer Experience

This is a protocol-internal derivation rule change. Application developers and Superchain developer
tools (Supersim, templates, etc.) are not affected. The only external-facing change is that chain
operators and batchers must ensure they do not produce overlapping span batches post-Karst.

## Risks & Uncertainties

### Fault proof soundness remains an open question

The core risk is that no explored approach fully solves the fault proof interaction without
significant trade-offs. The most viable approach — deferred L2 block fetching (approach 3) — works
but retains the overlap checking complexity in the new pipeline, limiting the simplification benefit
and raising the question of whether the derivation rule change is worth the protocol change cost.

### Sequencing window elapse migration

Existing batchers that produce overlapping span batches must be updated before Karst activation. The
fallback — singular batches for the sequencing window elapse race condition — must be implemented
and tested in the batcher. Operators should plan batcher upgrades ahead of the Karst activation time.

### Cross-client consistency

All derivation clients (op-node, kona, op-program) must implement the new drop rule identically.
Additionally, whatever approach is chosen for fault proof soundness must be implemented consistently
across all clients.

### Backwards compatibility

Pre-Karst span batches with overlaps already on L1 remain processable — the Karst activation gate
(based on L1 inclusion block timestamp) ensures old rules apply to old batches. Batches submitted
before Karst but included after Karst are subject to the new rules; batchers must account for this
during the transition.
