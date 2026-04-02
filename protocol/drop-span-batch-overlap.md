# Drop Overlapping Span Batches: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Sebastian Stammler (seb@oplabs.co)                 |
| Created at         | 2026-03-18                                         |
| Initial Reviewers  | @ajsutton, @theochap, @inphi                       |
| Need Approval From | @ajsutton                                          |
| Status             | Final (not implementing)                           |

## Purpose

This design doc explores dropping overlapping span batches as a derivation rule change and concludes
that **the change is not feasible** due to unresolvable implications for fault proofs, node restarts,
and interop. Overlap checks will instead be retained and implemented in any non-IO derivation
pipeline refactor.

The original motivation was:

1. **Simplify the derivation pipeline.** The current span batch overlap verification requires the
   `SafeBlockFetcher` interface to look up historical L2 blocks during batch validation. This
   complicates building a pure derivation pipeline where all L1 inputs are fetched in advance and IO
   is moved outside the pipeline. Removing the overlap check would simplify the derivation pipeline
   refactor drafted in [PR #19303](https://github.com/ethereum-optimism/optimism/pull/19303).

2. **Improve fault proof performance.** The current overlap verification in fault proofs requires
   K+1 preimage oracle reads (one parent block reference plus K full block payloads) and O(K×T)
   byte-by-byte transaction comparison, where K is the number of overlapping blocks and T is the
   average number of transactions per block. Eliminating the overlap check would also eliminate this
   cost, reducing on-chain dispute cost, off-chain FP VM execution, and ZK proving costs.

## Summary

Dropping overlapping span batches would simplify the derivation pipeline by removing the
`SafeBlockFetcher` dependency from batch validation. However, this introduces an implicit requirement
that **the safe head must always be at a span batch boundary** for derivation to work correctly. This
requirement cannot be guaranteed in practice — it is violated by fault proofs, node restarts, snap
sync, interop, and operational tools like `setDebugHead`.

Nine approaches to resolving these issues were explored. None are satisfactory: they either introduce
rule divergence between normal derivation and fault proofs, are too expensive for single-block fault
proof execution, require protocol changes disproportionate to the benefit, or retain the overlap
checking complexity that the rule change was meant to eliminate.

**Conclusion:** The overlap checks will be retained and implemented in the non-IO derivation pipeline
refactor using a deferred fetching pattern, where the pipeline returns the list of required L2 blocks
and the caller provides them.

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

This does not strictly prevent a non-IO pipeline — the pipeline can return a list of required L2
blocks and let the caller provide them. But this carries the overlap checking logic into the new
implementation, adding complexity that dropping overlapping batches would have eliminated entirely.

### The implicit span batch boundary requirement

Dropping overlapping span batches introduces an implicit requirement: **derivation can only start
from a safe head that is at a span batch boundary.** If the safe head is in the middle of a span
batch, the span batch that produced those blocks would overlap with the safe head and be dropped under
the new rule — breaking derivation.

This requirement is violated in multiple scenarios:

- **Fault proofs:** The dispute game bisects to arbitrary L2 blocks above the `SPLIT_DEPTH`
  (bisecting over output roots). The agreed prestate can place the safe head at any block, including
  mid-span-batch.
- **Node restarts:** If a node crashes while applying blocks from a span batch, the safe head may be
  left part way through it. The node restart process must find the L1 block to resume from and
  determine which safe L2 block to use. (This is mitigated by only updating the safe head once all
  blocks from a span batch are applied, but op-node currently sometimes rewinds the safe head on
  startup.)
- **Snap sync:** A node may snap sync to a peer that is in the middle of deriving from a span batch.
- **Interop:** The dispute game bisects over super roots at a particular timestamp. There is no way
  to guarantee that each individual chain is at a span batch boundary at that timestamp.
- **Operational tools:** `setDebugHead` can set the safe head to any block.

Fundamentally, the difficult question is how to know which safe block is actually valid to start
derivation from. Given an L2 block, you can iterate L1 to find the span batch it came from — but
then you need to know whether that span batch overlapped with a previous one. To determine that, you
find the previous span batch and check if it overlaps. But first you need to know if the previous
span batch was valid, so you have to find its parent. This recurses back through the entire batch
history, and with blobs, that history may not even be available.

### The fault proof's output root gap

The fault proof's agreed output root (`OutputV0`) captures L2 execution state (state root, block
hash) but **not** derivation state (which batch was in progress, the position within it). This gap
between execution state and derivation state means the fault proof has no way to determine the
correct derivation starting point within a span batch.

### Relation to Holocene

The [Holocene derivation design doc](holocene-derivation.md) introduced several relevant changes:

- **Partial span batch validity** with forward-invalidation only: if a singular batch within a span
  batch is invalid, only subsequent batches are invalidated, never prior ones. This principle of
  never backwards-invalidating batches would be preserved by any Karst rule change.
- **Strict batch ordering**: Batches within and across channels must be strictly ordered. This
  simplifies reasoning about which batch "owns" which block range.

## Explored Approaches

The following approaches were explored to resolve the fault proof (and broader mid-span-batch safe
head) problem. They are grouped by the depth of protocol change required.

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

**This is the approach we will take.** Since the overlap checks must be retained for fault proofs,
node restarts, snap sync, interop, and pre-Karst derivation, the simplification benefit of dropping
them is too limited to justify a protocol change.

#### 4. Re-derive from a span batch boundary

Ensure derivation always starts from a span batch boundary, either by re-deriving from the game's
anchor state or by constraining the dispute game to only bisect to span batch boundaries.

**Problem — too expensive.** Span batches can contain thousands of blocks, and fault proof block
execution must be over a single block. Re-deriving from a span batch boundary requires executing all
intermediate blocks up to the disputed block. Constraining the dispute game's bisection to span batch
boundaries does not help either — bisection above the split depth creates intermediate starting
points at arbitrary L2 blocks regardless of the anchor state.

Additionally, determining what the span batch boundary actually is requires checking every historic
span batch — the same recursion problem described above.

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

## Conclusion

Dropping overlapping span batches is not feasible at this time. The implicit requirement that the
safe head must be at a span batch boundary cannot be met in practice — fault proofs, node restarts,
snap sync, interop, and operational tools all create situations where the safe head is mid-span-batch.
No explored approach resolves this without either introducing soundness issues, requiring
disproportionate protocol changes, or retaining the very complexity the change was meant to eliminate.

**The overlap checks will be retained and implemented in the non-IO derivation pipeline refactor**
using a deferred fetching pattern: the pipeline returns the list of required L2 blocks for overlap
verification, and the caller is responsible for providing them. This achieves the goal of moving IO
outside the derivation pipeline without a protocol change.
