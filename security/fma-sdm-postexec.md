# Sequencer Defined Metering (SDM) & PostExec Transactions: Failure Modes and Recovery Path Analysis

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [A. Consensus & State-Root Divergence](#a-consensus--state-root-divergence)
    - [FM1: Divergence in applying or validating sequencer-set refunds → chain split or unprovable output root](#fm1-divergence-in-applying-or-validating-sequencer-set-refunds--chain-split-or-unprovable-output-root)
    - [FM2: Refund mis-metering — over-refund, fee-vault drain, value non-conservation](#fm2-refund-mis-metering--over-refund-fee-vault-drain-value-non-conservation)
    - [FM3: Refunds hide real compute → block-gas rate-limit bypass (no aggregate cap — superseded by pre-refund enforcement)](#fm3-refunds-hide-real-compute--block-gas-rate-limit-bypass-no-aggregate-cap--superseded-by-pre-refund-enforcement)
    - [FM4: Malformed / malicious PostExec payload accepted (structural validation gaps)](#fm4-malformed--malicious-postexec-payload-accepted-structural-validation-gaps)
  - [B. Block Building & Derivation](#b-block-building--derivation)
    - [FM5: Flashblock materialization diverges from sealed canonical block](#fm5-flashblock-materialization-diverges-from-sealed-canonical-block)
    - [FM6: Span-batch derivation has no PostExec structural gate](#fm6-span-batch-derivation-has-no-postexec-structural-gate)
    - [FM7: Disabled / flaky flashblocks SDM test leaves the divergence detector dark](#fm7-disabled--flaky-flashblocks-sdm-test-leaves-the-divergence-detector-dark)
  - [C. Fault Proofs](#c-fault-proofs)
    - [FM8: SDM unproven through the FPVM/Cannon path; no non-zero refund fixture](#fm8-sdm-unproven-through-the-fpvmcannon-path-no-non-zero-refund-fixture)
  - [D. Activation & Configuration](#d-activation--configuration)
    - [FM9: Inconsistent Lagoon activation across chains/components → fragmentation & unprovable blocks](#fm9-inconsistent-lagoon-activation-across-chainscomponents--fragmentation--unprovable-blocks)
    - [FM10: Latent SDM↔Interop coupling (decoupling rename not applied)](#fm10-latent-sdminterop-coupling-decoupling-rename-not-applied)
    - [FM11: Operator opt-in flag in wrong / split state (non-persistent, multi-endpoint)](#fm11-operator-opt-in-flag-in-wrong--split-state-non-persistent-multi-endpoint)
  - [E. User-Facing & Operational](#e-user-facing--operational)
    - [FM12: Synthetic 0x7D tx breaks standard user-facing RPC consumers (expose, don't filter)](#fm12-synthetic-0x7d-tx-breaks-standard-user-facing-rpc-consumers-expose-dont-filter)
    - [FM13: Receipt refund mis-attribution (index anchoring)](#fm13-receipt-refund-mis-attribution-index-anchoring)
    - [FM14: Admin RPC abuse / exposure](#fm14-admin-rpc-abuse--exposure)
    - [FM15: Policy-abstraction misconfiguration (silent economic error)](#fm15-policy-abstraction-misconfiguration-silent-economic-error)
  - [Generic items we need to take into account](#generic-items-we-need-to-take-into-account)
- [Action Items](#action-items)
- [Audit Requirements](#audit-requirements)
- [Appendix](#appendix)
  - [Appendix A: Verification environment & method](#appendix-a-verification-environment--method)
  - [Appendix B: Doc-vs-code discrepancies found while writing this FMA](#appendix-b-doc-vs-code-discrepancies-found-while-writing-this-fma)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                    |                       |
| ------------------ | --------------------- |
| Author             | Anton Evangelatov     |
| Created at         | 2026-06-16            |
| Initial Reviewers  | _TBD_                 |
| Need Approval From | _TBD (Security team)_ |
| Status             | Draft                 |

> [!NOTE]
> First draft. Anything I couldn't confirm in code is flagged unverified.

## Introduction

This document covers **Sequencer Defined Metering (SDM)** and the **PostExec transaction** it's built on.

**What it is.** SDM lets the sequencer reduce the gas a user is metered for — chiefly by refunding the EIP-2929
"cold→warm" access surcharge the protocol charges but the sequencer chooses not to pass on. The block builder appends
**exactly one synthetic trailing transaction of type `0x7D` (`TxPostExec`)** to each L2 block. That tx carries a
`PostExecPayload`: a list of per-transaction refund entries `SDMGasEntry{index, gas_refund}`. Execution clients
re-execute the block in PostExec-aware mode, apply the refunds (adjusting `gasUsed`, receipts, fee-vault settlement, and
the state root), and verifiers re-check the embedded payload rather than recomputing it.

**Where the code lives (scope of this analysis):**

- **EVM / refund math:** `rust/alloy-op-evm/src/block/mod.rs`, `rust/alloy-op-evm/src/inspector.rs`, `rust/op-revm/src/spec.rs`.
- **Block building:** op-reth payload builder; op-rbuilder flashblocks (`rust/op-rbuilder/.../builders/flashblocks/payload.rs`); rollup-boost (`crates/rollup-boost/src/flashblocks/service.rs`).
- **Consensus / derivation:** op-node batch derivation (`op-node/rollup/derive/batches.go`), span-batch codec; `op-chain-ops/pkg/sdm/`.
- **Fault proofs:** kona proof executor (`rust/kona/crates/proof/executor/...`), FPVM/Cannon path, absolute prestates.
- **Activation & config:** `op-node/rollup/toggles.go`, rollup-config Lagoon time, EL operator opt-in (`admin_setSdmPostExecOptIn`).
- **User-facing:** op-reth eth RPC receipts/blocks; flashblocks-rpc.

**Activation vehicle (verified).** SDM rides the **Lagoon** hardfork (the renamed Interop fork; rollup-config field
`LagoonTime`, JSON `lagoon_time`, serde-aliased from `interop_time`). In all three clients `is_sdm_active` currently
delegates through Interop → Lagoon (op-node `toggles.go:28`; op-reth `evm/src/lib.rs:138`; kona `rollup.rs:308`). The
revm spec for the SDM era is `OpSpecId::INTEROP` (eth base `OSAKA`, `op-revm/src/spec.rs:32-46`). Producing `0x7D` also
requires a per-node **operator opt-in** flag (`admin_setSdmPostExecOptIn`, in-memory, default OFF). **SDM is not active
on any production chain or default devnet today** — it runs only under test overrides and forced-activation acceptance
tests.

**References:**

- FMA template & process: `design-docs/assets/fma-template.md`, [FMAs in the SDLC](https://github.com/ethereum-optimism/pm/blob/main/src/fmas.md).
- Generic failure modes to inherit: `design-docs/security/fma-generic-hardfork.md`, `fma-generic-contracts.md`.
- SDM working notes: `~/sdm-docs/` — esp. `README.md`, `sdm-fault-proofs.md`, `sdm-infinite-mint-guards-and-intrinsic-warm-bug.md`, `max-gas-limit-enforcement.md`, `sdm-refund-cap-plan.md`, `flashblocks-sdm-postexec-divergence.md`, `postexec-derivation-gating-plan.md`, `sdm-hardfork-estimate.md`, `2026-06-15-sdm-toggle-lagoon-rename-plan.md`, `sdm-admin-rpc-plan.md`, `sdm-policy-abstraction-plan.md`.
- Key PRs/issues: #20738 (replay-cost fix), #21034, #908 (post-exec), #21105 (Lagoon rename), #21110 (explorer support), #21324 (proof test coverage), #21354 (Osaka flashblocks divergence).

## Failure Modes and Recovery Paths

### A. Consensus & State-Root Divergence

#### FM1: Divergence in applying or validating sequencer-set refunds → chain split or unprovable output root

- **Description:** Refunds are **sequencer-defined**: the block producer computes them and embeds the values in the
  `0x7D` payload. Verifiers, replicas, and the fault-proof program **do not recompute refunds** — they take the claimed
  values as authoritative and only (i) check structural validity plus the per-tx bound `refund ≤ evm_gas_used`,
  (ii) apply the post-exec settlement and set `gas_used = evm_gas_used − refund`, and (iii) require the resulting
  receipts and state root to match. So the split surface is **not** "different clients compute different refunds" but the
  steps every client must perform **identically on the same payload**: computing `evm_gas_used`, applying the settlement,
  and enforcing the structural + per-tx-bound checks. If two verifiers apply the same payload to different state roots,
  or one accepts what another rejects, that's a **chain split**; the same divergence between sequencer and proof program
  is an **unprovable (or wrongly-provable) output root** that stalls withdrawals. A _wrong refund value_ is a separate,
  non-split failure (FM2) — every verifier faithfully applies the same wrong value. The Osaka entry-count incident
  (#21354) shows the surface is live even though that specific case was a test artifact (FM7).
- **Risk Assessment:** **High impact, low likelihood.** Because verifiers apply rather than recompute, the whole
  "recompute-and-disagree" class is out of scope by design, and the apply/validate path is a single shared Rust crate
  (`alloy-op-evm` Verify mode) used by both op-reth and the kona proof program. **op-geth is end-of-life and won't
  support SDM**, so there's no second independent EVM re-implementation. The residual differential surface is narrow:
  drift between op-reth Verify mode and the kona proof program (incl. MIPS-vs-native, FM8), and between the Rust
  validators and the **Go-side** structural/derivation checks (op-node, `op-chain-ops/pkg/sdm`), which independently
  reimplement the _validation_ (not the refund math).
- **Mitigations:**
  1. _Reduce chance:_ keep one shared apply/validate path (`block/mod.rs` Verify). The consensus-critical invariants —
     the per-tx bound `refund ≤ evm_gas_used` (`:554,933`), the structural checks (`:130,528-556,1080-1087`), and the
     conservation/underflow guards (`:634,663`) — must be identical across **every verifier and the proof program**.
     kona enforces the same bound and structural checks and doesn't recompute warming (`sdm-fault-proofs.md`).
  2. _Add:_ a **canonical spec for the application & validation semantics** — settlement math,
     `gas_used = evm_gas_used − refund`, receipt adjustment, the per-tx bound, and the structural rules (one `0x7D`,
     last-in-block, valid indices, block-number anchor) — so the Go derivation/decode checks (op-node,
     `op-chain-ops/pkg/sdm`) and the kona proof program can't drift from op-reth Verify mode. Add a **differential test**
     asserting op-reth Verify mode and the kona proof program apply the same `0x7D` payload to the same block hash /
     state root, and that the Go structural checks accept/reject exactly the payloads the Rust verifier does. (The
     warming **exclusion set** and refund constants at `inspector.rs:19-21` are _sequencer policy_, not consensus — their
     cross-builder consistency is a builder/economic concern, FM2/FM5.)
- **Detection:** `debug_replaySDMBlock` (synthesized-vs-embedded payload equality); kona native + FPVM output-root match
  vs op-reth; acceptance tests `TestSDMEnabledPayloadAndReplayMatch`, `TestSDMPostExecBlockDerivesAndChainProgresses`;
  in production, replica state-root disagreement or op-challenger unable to counter a claim.
- **Recovery Path(s):** A live post-activation split needs an **emergency coordinated hardfork** to pin the canonical
  math. If the divergence is only in the proof path (kona), a kona-client hotfix plus a **new absolute prestate release**
  (governance-approved swap in superchain-registry + op-challenger-runner config). Both high-effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — shared apply/validate path (`rust/alloy-op-evm/src/block/tests.rs`):_ `test_post_exec_producer_verifier_roundtrip`, `test_post_exec_settlement_deltas_conserve_value`, `test_post_exec_settlement_deltas_skip_non_refunding_txs`, `test_settlement_state_account_preserves_original_info`, `test_post_exec_settlement_conserves_total_eth_supply`, `test_post_exec_settlement_underflow_is_rejected`.
  - _Unit — kona proof-program parity (`rust/kona/crates/proof/executor/src/builder/core/tests.rs`):_ `post_exec_valid_empty_payload_executes_without_state_or_gas_change` plus the reject-matrix (FM4), all through `StatelessL2Builder`.
  - _Unit — Go-side structural/derivation reimplementation (`op-node/rollup/derive/`):_ `TestDeriveSpanBatchSDMGate`, `TestSpanBatchReadPostExecTxData`; the `batches_test.go` cases `"postExec tx included pre-SDM"` / `"postExec tx included at SDM"`; cross-encode round-trips `TestPostExecTxMarshalBinaryDifferential`, `TestUnmarshalPostExecTxRoundTrip` (`op-core/types/post_exec_tx_test.go`).
  - _Acceptance:_ `TestSDMEnabledPayloadAndReplayMatch`, `TestSDMPostExecBlockDerivesAndChainProgresses` (span + singular), `TestSDMPostExecBlockDerivesOnIsolatedVerifier` (tests/sdm/block_test.go); `TestInteropSingleChainFaultProofsWithSDM` (tests/interop/proofs-singlechain/).
  - _Gap (BLOCKING):_ no **differential test** asserting op-reth Verify mode and the kona proof program apply the _same non-empty_ `0x7D` payload to the same block hash, nor that the Go structural checks accept/reject exactly the same payloads as the Rust verifier — the cross-implementation drift surface this FM is about is untested.

#### FM2: Refund mis-metering — over-refund, fee-vault drain, value non-conservation

- **Description:** SDM moves real value — refunds reduce what users pay into the fee vaults. Two historical/latent bug
  classes: (a) the **intrinsic-warm over-refund** — granting the 2500 warm rebate for a tx's own intrinsically-warm
  accounts (sender/`to`/created-address) and for fee-vault settlement touches, which over-credits senders and **drains
  OP fee vaults** (Immunefi #81847); and (b) **value non-conservation** — settlement deltas that don't net to zero, in
  the limit minting ETH. The intrinsic-warm fix has landed on this branch, but the regression surface is large: any
  future change to `collect_intrinsic_warmth` / `note_account_touch` could silently reintroduce it (the buggy behavior
  was previously locked in by green tests).
- **Risk Assessment:** **Medium impact, low likelihood** now the fix has landed. It was _systematic_ when present (fires
  on sequential same-EOA txs and popular `to` contracts), so a regression would be high-impact again.
- **Mitigations:**
  1. _Reduce chance:_ three-layer defense — per-tx bound `refund ≤ evm_gas_used` (`block/mod.rs:554,933`); settlement
     **underflow guard** `checked_sub` → `PostExecSettlementUnderflow` (`:634`); value conservation via
     `post_exec_settlement_deltas` (`:663`). Intrinsic-warm fix at `inspector.rs:208,259,261,415`. Regression tests:
     `test_intrinsic_warm_{to,sender,created}_address_is_not_rebated`, `test_fee_recipient_settlement_touch_is_not_rebated`,
     `test_post_exec_settlement_{conserves_total_eth_supply,underflow_is_rejected}`.
  2. _Add:_ a spec/reference cross-check of the exclusion set (FM1); a test asserting saturating arithmetic
     (`inspector.rs:134`, `block/mod.rs:688-708`) can't silently truncate a settlement delta under max realistic inputs
     (today saturation is defense-in-depth behind the underflow guard, but nothing pins that it can't mask an accounting
     bug); re-baseline the downstream Go acceptance test (`TestSDMStorageRefundBreakdown`) on devstack CI.
- **Detection:** fee-vault balance monitor; acceptance tests asserting zero refund for plain transfers;
  conservation/underflow unit tests; replay parity.
- **Recovery Path(s):** A code fix (already done for the known bug). **Note:** since all clients compute the same wrong
  value, an over-refund that shipped would _not_ be a chain split — it's value misrouting needing treasury
  reconciliation / governance, not a fork. A true ETH-creation bug would be critical and need an emergency upgrade.
- **Tests covering this FM (this monorepo):**
  - _Unit — intrinsic-warm exclusion regression set (`block/tests.rs`):_ `test_intrinsic_warm_{to,sender,created,beneficiary,precompile}_..._is_not_rebated`, `test_intrinsic_warm_7702_authority_is_not_rebated`, `test_fee_recipient_settlement_touch_is_not_rebated` (the Immunefi #81847 fee-vault-drain class).
  - _Unit — value conservation / underflow (`block/tests.rs`):_ `test_post_exec_settlement_conserves_total_eth_supply`, `test_post_exec_settlement_underflow_is_rejected`, `test_post_exec_settlement_deltas_conserve_value`.
  - _Unit — inspector accounting (`rust/alloy-op-evm/src/post_exec/inspector/tests.rs`):_ `repeated_account_touch_refunds_once`, `repeated_storage_refunds_without_account_double_count`, `deposit_warms_but_does_not_claim`, `settlement_note_warms_without_claiming_but_opcode_access_still_rebates`, `post_exec_tx_never_claims_refunds`, `intrinsic_access_list_warmth_does_not_claim`.
  - _Acceptance:_ `TestSDMStorageRefundBreakdown`, `TestSDMMixedWorkloadSmoke`, `TestSDMDisabledNoRefunds` (tests/sdm/); `TestFlashblocksSDMPhantomWarmingDivergence` (tests/flashblocks/).
  - _Gap (NON-BLOCKING):_ no test pins that **saturating** settlement arithmetic can't silently truncate a delta under max realistic inputs; the `TestSDMStorageRefundBreakdown` devstack-CI re-baseline is still pending.

#### FM3: Refunds hide real compute → block-gas rate-limit bypass (no aggregate cap — superseded by pre-refund enforcement)

- **Description:** Refunds lower canonical `gasUsed`. With only a _per-tx_ bound, a sequencer could refund ~100% of every
  tx, driving canonical `gasUsed → 0` while the real pre-refund EVM work stays large — so the block-gas-limit rate limit
  no longer bounds real compute, and replicas/challengers fall behind the tip, threatening the fault-proof
  liveness/Stage-1 guarantee. **Design decision (2026-06-19): no percentage-based aggregate refund cap. The cap is
  `evm_gas_used`** — the per-tx bound `refund ≤ evm_gas_used`, combined with pre-refund gas-limit enforcement:
  - **Pre-refund gas-limit enforcement** (the `evm_gas_used` accumulator + admission check, `block/mod.rs:350,847-848,1025`)
    — **landed**; bounds _real compute_ per block (`sum(evm_gas_used) ≤ block_gas_limit`), the Stage-1 concern. This is
    the layer that neutralizes the compute-blowup attack.
  - **Per-tx bound `refund ≤ evm_gas_used`** (`block/mod.rs:554,933`) — **landed**; the only refund cap, by design.
  - **Aggregate refund cap** (`MAX_REFUND_BPS`, designed in `sdm-refund-cap-plan.md`) — **will not be implemented**; that
    plan is closed (it was never implemented — only a stray comment at `tests.rs:1066`).
- **Risk Assessment:** **Low residual (post-decision).** Pre-refund enforcement already bounds real compute, so the
  Stage-1 violation the cap was meant to prevent is handled without it. Accepted residual: a sequencer may still refund
  up to 100% of every tx, so canonical `gasUsed` can legitimately approach 0 while real compute stays bounded — and the
  magnitude of any value misrouting is bounded only per-tx by `evm_gas_used`, not by an aggregate percentage. That's an
  economic/policy property, **not** a consensus or Stage-1 concern, and is accepted by the decision above.
- **Mitigations:**
  1. _Reduce chance:_ pre-refund enforcement bounds `sum(evm_gas_used) ≤ block_gas_limit` at the executor level (op-reth
     + kona both route through `OpBlockExecutor`). Tests: `test_pre_refund_gas_limit_counts_sdm_refunded_gas`,
     `test_verifier_rejects_malicious_payload_whose_refunds_hide_pre_refund_overuse`.
  2. _DECIDED — no aggregate cap:_ the per-tx bound plus pre-refund enforcement supersede `MAX_REFUND_BPS`; no shared
     constant / verifier `finish()` cap / producer clamp / kona cap-parity work is needed. Still open: confirm the
     **`finish()` pre-refund debug-assert** the `max-gas-limit` notes describe — **absent** today (`block/mod.rs:1077-1106`
     only checks unconsumed entries); add an e2e malicious-sequencer test and a **cross-flashblock** cumulative-overrun
     test (the per-flashblock executor accumulator resets; the cross-flashblock bound relies on the producer's persistent
     `ExecutionInfo.cumulative_evm_gas_used`).
- **Detection:** verifier rejects with `TransactionGasLimitMoreThanAvailableBlockGas`; a per-block
  `refund / refundable_evm_gas` ratio metric (planned in the cap plan §6.5, **not yet added**); an explorer showing a
  full block with `gasUsed ≈ 0`.
- **Recovery Path(s):** A consensus-rule addition — trivial if it lands before activation, a coordinated
  upgrade/hardfork if SDM is already live.
- **Tests covering this FM (this monorepo):**
  - _Unit — pre-refund enforcement (`block/tests.rs`):_ `test_pre_refund_gas_limit_counts_sdm_refunded_gas` (tx fitting canonical `gas_used` but exceeding pre-refund `evm_gas_used` is rejected); `test_pre_refund_gas_limit_never_binds_with_sdm_off`; `test_evm_gas_used_tracks_pre_refund_gas_under_sdm` (`evm_gas_used − gas_used == total refund`); `test_verifier_rejects_malicious_payload_whose_refunds_hide_pre_refund_overuse`; `test_verifier_accepts_payload_when_pre_refund_stays_below_limit`.
  - _Decided — not a gap:_ the **aggregate refund cap** won't be implemented, so its absence needs no test. _Still-open test gaps:_ no **cross-flashblock cumulative pre-refund-overrun** test; no **e2e malicious-sequencer** test; the `finish()` pre-refund debug-assert is absent (`block/mod.rs:1077-1106`).

#### FM4: Malformed / malicious PostExec payload accepted (structural validation gaps)

- **Description:** The embedded `0x7D` payload is attacker-controllable by a malicious sequencer. Incomplete structural
  validation could let a payload target a deposit tx or the `0x7D` tx itself, reference an out-of-range/duplicate index,
  leave entries unconsumed, or place a second/non-final PostExec tx — any of which mis-applies refunds and diverges
  state. A related risk is **Go/Rust validation drift**: the Go decoder
  `op-chain-ops/pkg/sdm/payload.go::DecodePayload` validates **only `Version`** (`:41`) — it doesn't bound `refund ≤ gas`,
  reject duplicate/out-of-range indexes, or enforce the one-per-block / last-in-block / block-number-anchor invariants.
  Those are Rust-side only.
- **Risk Assessment:** **High impact, low likelihood.** The Rust verifier checks look complete and sit on the
  consensus-critical path; the Go decoder is currently diagnostic/replay tooling, not a consensus verifier.
- **Mitigations:**
  1. _Reduce chance:_ verifier `verifier_post_exec_refund_for_tx` rejects deposit/post-exec targets and
     `refund > evm_gas_used` (`block/mod.rs:528-556`); duplicate index rejected at map construction (`:130`); `finish()`
     rejects unconsumed entries (`:1080-1087`); the `0x7D` tx is bounded with `evm_gas_used = 0` (`:861`). Tests:
     `test_verifier_rejects_refund_exceeding_evm_gas` + the kona reject-matrix.
  2. _Add:_ either add full invariant validation to the Go `DecodePayload`, or explicitly mark it non-authoritative so no
     Go consumer treats decode-success as "valid". (The **at-most-one-0x7D** and **last-in-block** checks the FMA marked
     unverified are now confirmed present in the op-alloy parser — see Tests.)
- **Detection:** verifier rejects with `InvalidPostExecPayload`; kona reject-matrix tests; replay-parity comparing
  Go-decoded payload to Rust-applied refunds.
- **Recovery Path(s):** The verifier already rejects bad payloads (consensus-safe). Adding Go-side validation is
  tooling-only unless Go ever sits on a consensus path.
- **Tests covering this FM (this monorepo):**
  - _Unit — Rust verifier reject-matrix (`block/tests.rs`):_ `test_verifier_rejects_refund_exceeding_evm_gas` (boundary `refund == evm_gas_used` permitted), `test_mismatched_payload_block_number_fails_pre_execution`, `test_duplicate_payload_index_fails_pre_execution`, `test_verifier_rejects_payload_targeting_non_normal_tx`, `test_verifier_returns_zero_when_no_entry_for_tx`, `test_finish_reports_all_unconsumed_post_exec_entries`, `test_disabled_mode_rejects_post_exec_tx`.
  - _Unit — op-alloy parser invariants (`rust/op-alloy/crates/consensus/src/post_exec/tests.rs`):_ `parse_accepts_trailing_post_exec_tx`, `parse_returns_none_without_post_exec_tx`, `parse_rejects_post_exec_tx_when_sdm_inactive`, `parse_rejects_multiple_post_exec_txs` (at-most-one), `parse_rejects_post_exec_tx_not_last` (last-in-block), `parse_rejects_payload_anchored_to_wrong_block`, plus version/trailing-byte decode rejects. _Confirms the "at-most-one" / "last-in-block" checks the FMA marked unverified._
  - _Unit — kona proof-executor reject-matrix (`.../builder/core/tests.rs`):_ `post_exec_payload_rejects_{deposit_target,post_exec_target,duplicate_entries,unconsumed_entry,refund_exceeding_gas_used}`, `post_exec_sdm_enabled_rejects_{wrong_block_number,duplicate_post_exec_txs}`; span-batch validity `test_check_batch_with_post_exec_tx_pre_sdm`.
  - _Gap (NON-BLOCKING):_ the Go decoder `DecodePayload` validates **only `Version`**, has **no unit test** in `op-chain-ops/pkg/sdm/`, and isn't marked non-authoritative.

### B. Block Building & Derivation

#### FM5: Flashblock materialization diverges from sealed canonical block

- **Description:** Flashblocks materialize as `base + concat(diff.transactions) + latest(diff.post_exec_tx)`. Three code
  paths can produce a body whose trailing `0x7D` (and therefore roots) diverges from the sealed canonical block:
  - **Builder/EL fallback (rollup-boost):** if op-rbuilder's streamed view ≠ what op-reth seals, or rollup-boost serves
    an EL-fallback block, the canonical settlement differs from what was preconfirmed (the "(B)" possibility the
    divergence doc couldn't rule out locally).
  - **op-node/CL overlay:** `OpExecutionData::from_flashblocks_unchecked` (`op-alloy envelope.rs:217-269`) folds only
    `diff.transactions` and **never appends `post_exec_tx`**, while taking hash/root from the last flashblock → a
    deterministic hash mismatch for any node self-driving from flashblocks. The SDM-aware fix exists only on an
    **unmerged** branch.
  - **latest-vs-last selection:** rollup-boost uses `flashblocks.last().post_exec_tx` (`service.rs:119`) and relies on an
    _asserted, not enforced_ invariant ("once a flashblock carries `post_exec_tx`, all later ones do too"). op-rbuilder
    emits `None` when a flashblock has no new entries; the op-alloy overlay uses `rev().find_map` (latest `Some`). The
    two materializers can therefore disagree (drop vs keep the tx).
- **Risk Assessment:** **High impact, low–medium likelihood.** A genuine builder/EL divergence is a silent
  preconfirmation/consensus break; op-rbuilder routes real execution through the same `materialize_post_exec` path and
  sibling EL-build tests pass, which lowers likelihood. The overlay path only bites once SDM is live and a node
  self-drives from flashblocks.
- **Mitigations:**
  1. _Reduce chance:_ op-rbuilder uses the O(1) `materialize_post_exec` (the #20738 follow-up), re-deriving entries from
     monotonic accumulated `info.extra.post_exec_entries`; `debug_replaySDMBlock` agreement is recorded in the fix
     checklist.
  2. _Add:_ merge the overlay-SDM-aware fix so **both** materializers re-attach `0x7D` via a single shared
     `post_exec_source` helper using identical "latest `Some`" selection; add a builder-side assertion that the final
     flashblock is non-`None` once any earlier one carried the tx; add a production metric alarming on builder-view ≠
     seal.
- **Detection:** the `observed.blockHash == canonical block.Hash` assertion (`flashblocks_sdm_test.go:215`);
  `debug_replaySDMBlock` mismatch; preconfirmation-vs-final `opGasRefund` diff; state-root mismatch on the overlay's
  `pending` rebuild.
- **Recovery Path(s):** rollup-boost EL fallback yields a valid (non-builder) block, so chain liveness holds; root-cause
  and redeploy the builder. Merging the overlay fix is required before exposing the flashblocks overlay on an SDM chain.
  Medium effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — rollup-boost materializer (`.../flashblocks/service.rs`):_ `flashblock_builder_leaves_transactions_unchanged_without_post_exec_tx`, `flashblock_builder_appends_single_delta_post_exec_tx_at_end`, `flashblock_builder_uses_latest_post_exec_tx_once`, `test_flashblocks_builder`, `test_flashblocks_fallback_to_get_payload`, `test_flashblocks_different_payload_id`.
  - _Unit — op-rbuilder:_ `offset_post_exec_entries_preserves_previous_and_offsets_current` (`builders/flashblocks/payload.rs`); `post_exec_tx_follows_operator_opt_in` (`src/tests/sdm.rs`).
  - _Acceptance:_ `TestFlashblocksSDMMaterializesPostExecBlock`, `TestFlashblocksSDMPhantomWarmingDivergence` (tests/flashblocks/) — **both skipped on the kona-node and op-geth presets** (`SkipOnKonaNode` / `SkipOnOpGeth`), i.e. exercised only on the op-reth flashblocks preset.
  - _Gap (BLOCKING):_ the **op-node/CL overlay** path (`from_flashblocks_unchecked`) that drops `post_exec_tx` has its SDM-aware fix on an **unmerged branch** and is **untested** here; no builder-view ≠ seal production metric.

#### FM6: Span-batch derivation has no PostExec structural gate

- **Description:** The **span-batch** per-block validation loop (`batches.go:399-413`) filters only Deposit and
  pre-Isthmus SetCode txs — it has **no `isSDM` check and no PostExec handling at all**, unlike the singular-batch path
  (`checkSequencerTxData`, `batches.go:200`). Span batch is the **default** batcher path. A span batch carrying a `0x7D`
  tx **before SDM activation**, or a malformed one (not last, duplicated, wrong block-number anchor), would be accepted
  by derivation and then hard-rejected at the EL → permanent L1 data that **stalls every follower**. The span-batch codec
  changes that make `0x7D` round-trip (`span_batch_txs.go:446`) widen the surface.
- **Risk Assessment:** **High impact, medium likelihood.** A bad `0x7D` in a span batch is permanent on L1 and stalls all
  followers; the default path makes this the common case once SDM ships.
- **Mitigations:**
  1. _Reduce chance:_ the singular-batch gate exists (`batches.go:200`); the EL is a backstop (`alloy-op-evm` rejects on
     SDM-config mismatch).
  2. _Add (BLOCKING before activation):_ port the `isSDM` activation gate plus positional ("must be last"), count
     ("exactly one"), and block-number-anchor checks into the **span-batch** loop; add a defensive pre-encode check in
     `BlockToSingularBatch` / the span equivalent (gating-plan Stages 2–3 are **not implemented**); add negative
     acceptance tests (`0x7D` not-last, pre-activation `0x7D` in a span batch).
- **Detection:** today only surfaces as follower `BatchDrop` / EL reject _after_ L1 inclusion (too late); the gates above
  move detection to derivation time and CI.
- **Recovery Path(s):** A derivation code change — must land before any chain schedules Lagoon. If a bad batch is already
  on L1 post-activation, recovery is a derivation-rule hotfix coordinated across all followers — high effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — span-batch derivation gate (`op-node/rollup/derive/`):_ `TestDeriveSpanBatchSDMGate` (rejects `0x7D` pre-SDM, accepts when `LagoonTime` active), `TestSpanBatchReadPostExecTxData`, `TestSpanBatchTxPostExecConvert`, `TestSpanBatchTxPostExecRoundTrip`, `TestSpanBatchTxsPostExecRoundTripFullTxs`, `TestSpanBatchTxsPostExecSkipsChainIDValidation`; `batches_test.go` cases `"postExec tx included pre-SDM"` / `"postExec tx included at SDM"`.
  - _Acceptance:_ `TestSDMPostExecBlockDerivesAndChainProgresses` runs an explicit `span_batch` sub-test (tests/sdm/).
  - _Status vs FMA (partial closure):_ an **activation gate now exists at the span-batch decode layer** — `DeriveSpanBatch` rejects a `0x7D` tx when `IsSDM(blockTimestamp)` is false (`span_batch.go:656-663`), covered by `TestDeriveSpanBatchSDMGate`. A real backstop the FMA's older-branch snapshot didn't credit: a pre-activation `0x7D` in a span batch is now dropped at decode time, not silently passed to the EL. **Still open:** (a) the `batches.go` span _validation loop_ (`:399-413`) still filters only Deposit/pre-Isthmus SetCode and doesn't call `checkSequencerTxData`/gate `0x7D` per-tx (the `isSDM` gate at `:174-202` is singular-path only); (b) the **positional / count / block-number-anchor** checks are Rust-side only — the Go decode gate is activation-only; (c) no negative test for a `0x7D` placed **not-last** in a span batch, and no pre-encode defensive check.

#### FM7: Disabled / flaky flashblocks SDM test leaves the divergence detector dark

- **Description:** `TestFlashblocksSDMMaterializesPostExecBlock`'s original "which flashblock is final?" heuristic could
  latch `observed` on a non-final in-progress flashblock (fewer entries) and compare it against the fully-sealed
  canonical block (more entries) — the documented 9-vs-15 CI failure, amplified by Osaka calldata repricing packing more
  txs/block. The team's response was to `t.Skip` the test, which dark-ened the one signal that separates a harmless
  capture race from a **real builder/EL divergence (FM5)**. Note #21354's framing ("op-node derivation diverged /
  op-rbuilder over-accumulated") is **mis-stated**: op-node derivation never ran before the failing assertion, and the
  higher count was on the _canonical_ side, not the builder stream.
- **Risk Assessment:** **Medium impact, high likelihood.** Flaky/disabled coverage tends to persist until someone fixes
  it, and it blinds us to FM5.
- **Mitigations:**
  1. _Reduce chance:_ the divergence doc §6 prescribes selecting the observed flashblock **by canonical block hash**
     (identity match, not a timing guess); the branch partially implements this.
  2. _Add:_ key the capture map by `diff.block_hash` (not `blockNum`), drop the "next-block = seal" heuristic and the 15 s
     deadline, and **un-skip** the test.
- **Detection:** CI signal once un-skipped; currently none.
- **Recovery Path(s):** Implement hash-keyed selection and un-skip. Low effort.
- **Tests covering this FM (this monorepo):**
  - _Acceptance:_ `TestFlashblocksSDMMaterializesPostExecBlock` (tests/flashblocks/flashblocks_sdm_test.go).
  - _Status vs FMA (largely resolved):_ the test is **no longer `t.Skip`'d** and now implements the prescribed **hash-keyed selection** — captures streamed flashblocks into `observedByHash` keyed by `diff.block_hash`, then matches the sealed canonical block by hash (`:117,201,213-217`), dropping the old heuristic and deadline race. A canonical block matching no streamed flashblock now fails **loudly** ("canonical block … matched no streamed op-rbuilder flashblock") — exactly the real-divergence (FM5) signal. It remains gated by `SkipOnKonaNode` / `SkipOnOpGeth` (`:43-44`), so the detector runs on the **op-reth flashblocks preset only** — the residual gap is preset coverage, not the disabled-detector problem the FMA describes.

### C. Fault Proofs

#### FM8: SDM unproven through the FPVM/Cannon path; no non-zero refund fixture

- **Description:** SDM is wired into the kona proof executor (`.../builder/core.rs:293-310`) and into the FPVM via
  `PostExecEvmFactoryAdapter` (`single.rs:133`, `interop/mod.rs:81`, `fpvm_evm/factory.rs`), and absolute prestate
  artifacts now exist (`rust/kona/prestate-artifacts-cannon/`). Two coverage gaps remain: (a) there's **no
  non-zero-refund fixture** exercised through `StatelessL2Builder` (explicitly deferred), so the proof path is proven
  only on empty/trivial payloads; and (b) the SDM acceptance proof test runs `kona-host super --native`, **not** the
  actual Cannon/FPVM (MIPS) path, and the dedicated Cannon step is `SkipOnKonaNode`. The proof path **applies** the
  claimed refunds and re-derives the output root — it doesn't recompute warming. If the MIPS build diverges from native
  in re-execution, settlement application, or structural validation (e.g. oracle/hint behavior or non-determinism under
  the post-exec hooks), the production proof path breaks even though native tests pass — a real `0x7D` block could be
  unprovable on-chain.
- **Risk Assessment:** **High impact, low–medium likelihood.** The adapter is wired and prestates exist, but
  SDM-specific MIPS-vs-native determinism is unverified end-to-end and there's no non-trivial refund fixture in the proof
  path.
- **Mitigations:**
  1. _Reduce chance:_ shared `alloy-op-evm` Verify path; executor-level reject-matrix tests in `kona-executor`;
     supply-conservation/underflow tests in `block/tests.rs`.
  2. _Add (BLOCKING before activation):_ generate an op-reth devnet fixture with non-empty refund entries and assert
     `kona block hash == op-reth block hash` through `StatelessL2Builder`; add a **Cannon/FPVM-mode SDM proof run** to
     acceptance; ensure the shipped absolute prestate covers SDM-active behavior; sweep across hardforks rather than a
     single fixed fixture.
- **Detection:** dispute-game disagreement / op-challenger unable to counter; Cannon trace divergence vs native;
  output-root mismatch in acceptance e2e.
- **Recovery Path(s):** A kona bug: hotfix kona-client + **new absolute prestate release** (governance-approved swap in
  superchain-registry + op-challenger-runner config) — high effort (prestate rollout + dispute-game window). A true
  cross-client consensus split: hardfork (FM1).
- **Tests covering this FM (this monorepo):**
  - _Unit — kona proof executor (`.../builder/core/tests.rs`):_ `post_exec_valid_empty_payload_executes_without_state_or_gas_change`, the reject-matrix, and the inherit/forced-inactive gate tests (`post_exec_sdm_inherit_rejects_post_exec_tx`, `post_exec_sdm_forced_inactive_rejects_appended_post_exec_tx`) — all through `StatelessL2Builder` + `PostExecEvmFactoryAdapter` against fixture `testdata/block-26207960.tar.gz`. `test_sdm_rides_lagoon` (`rollup.rs`) pins `is_sdm_active` ⇒ Lagoon.
  - _Acceptance:_ `TestInteropSingleChainFaultProofsWithSDM` (single-chain super-root transition over an SDM `0x7D` block, via `kona-host super --native`); `TestSDMPostExecBlockDerivesOnIsolatedVerifier` (tests/sdm/, skipped on `MixedL2CLKona`).
  - _Gaps (BLOCKING):_ (a) **no non-zero-refund fixture** through `StatelessL2Builder` asserting `kona hash == op-reth hash`; (b) **no Cannon/FPVM (MIPS)-mode SDM proof run** — the acceptance proof test uses `--native` and the dedicated Cannon step is `SkipOnKonaNode`, so MIPS-vs-native determinism on a `0x7D` block is unverified end-to-end.
  - _Investigation finding (2026-06-19) — why gap (a) is hard, and a latent sharp edge it exposed:_ all eight bundled fixtures (`testdata/block-2620xxxx.tar.gz`) are **pre-Isthmus** (block ts ≈ `1744218460` vs `isthmus_time = 1744905600`, ~8 days short), so the operator-fee scalar is unset. Driving a _valid non-zero_ refund through `StatelessL2Builder` panics in `op-revm/src/l1block.rs:186` (`Missing operator fee scalar for isthmus L1 Block`), because the SDM settlement (`block/mod.rs:698-703`) calls `operator_fee_charge` **unconditionally**, whereas op-revm's normal path gates operator fee on Isthmus first. This is **not production-reachable** (SDM ⊂ Lagoon ⊃ Isthmus, so the scalar is always present when SDM is active), but it's a latent `.expect()` sharp edge worth a guard or documented invariant assertion. Forcing Isthmus active on a bundled fixture fails differently — `BlockHashContractCall: "Preimage not found"` — because the stateless fixture lacks the trie preimages Isthmus reads. **Conclusion:** the BLOCKING fixture item genuinely requires a **new post-Isthmus fixture with non-zero refunds** via `ExecutorTestFixtureCreator` against a live SDM-enabled op-reth node — it can't be retrofitted onto the current pre-Isthmus testdata.

### D. Activation & Configuration

#### FM9: Inconsistent Lagoon activation across chains/components → fragmentation & unprovable blocks

- **Description:** SDM has no dedicated activation timestamp; it rides `lagoon_time`, which must be set consistently
  across **op-node rollup config and op-reth/kona chainspec** — surfaces that use _mismatched field names_
  (`interop_time` vs `lagoon_time`). (**op-geth is EOL and won't support SDM**: an op-geth node rejects `0x7D` blocks
  outright, so before enabling SDM a chain must ensure no op-geth nodes remain in its sequencer/follower set — that's a
  hard incompatibility, not a config-mismatch to reconcile.) The devnet notes show the hazard directly: op-node read
  `interop_time` and reported `protocolActive:true` while op-rbuilder/op-reth read a separate source and reported
  `false`. Two failure shapes: (a) some Superchain chains get Lagoon and others don't → **fragmentation** into SDM and
  non-SDM chains with divergent fee behavior; (b) within one chain, the **EL operator gate** flips on `0x7D` production
  at a timestamp where the CL/proof considers SDM inactive → kona rejects every block as `UnexpectedPostExecTx` / drops
  the batch → **safe head can't be proven, withdrawals stuck chain-wide**.
- **Risk Assessment:** **High impact, medium likelihood.** Activation must be wired into 3+ config surfaces with
  mismatched names; the EL operator opt-in is an off-chain in-memory toggle with no on-chain anchor, so EL/CL timing
  mismatch is easy to produce.
- **Mitigations:**
  1. _Reduce chance:_ single source via `IsForkActive(forks.Lagoon)` (op-node) and shared `is_sdm_active_at_timestamp`
     (Rust); serde alias `interop_time → lagoon_time` (`types.go:874-876`); kona-native upgrade-epoch tests asserting
     accept/reject per config.
  2. _Add:_ a **cross-component preflight** asserting op-node, op-reth, and kona all carry the same Lagoon timestamp
     before launch **and that no op-geth node is in the set**; give SDM a real fork schedule (not just "rides Interop")
     before any mainnet use; add an activation-boundary fault-proof test; file the superchain-registry timestamp ticket
     (hardfork-estimate Gap #1, currently unticketed).
- **Detection:** `admin_sdmStatus` `protocolActive` differing across endpoints; block N produced with/without `0x7D`
  inconsistently; derivation `BatchDrop` on `0x7D`; proofs failing immediately at "turn-on".
- **Recovery Path(s):** Pre-activation: set the missing timestamp everywhere and redeploy (low effort). Post-activation on
  a live chain: a coordinated **hardfork re-schedule** (high effort).
- **Tests covering this FM (this monorepo):**
  - _Unit — config / serde alias (`op-node/rollup/types_test.go`):_ `TestConfigUnmarshalLegacyInteropTimeAlias` (legacy `interop_time` promotes to `LagoonTime`; canonical `lagoon_time` wins when both present), `TestConfig_IsSDM`, `TestConfig_ActivationTime`, `TestActivations`.
  - _Unit — Rust chainspec:_ `test_sdm_rides_lagoon` (`rollup.rs`).
  - _Unit — sequencer status:_ `TestSequencerSdmStatus` (`ProtocolActive`/`Effective`/`ActivationTime` derived from the Lagoon timestamp).
  - _Acceptance:_ `TestSDMActivatesAtInteropBoundary` (tests/sdm/boundary_test.go) — pre-boundary blocks have no `0x7D`, post-boundary blocks carry it with refund entries.
  - _Gap:_ no **cross-component preflight** test asserting op-node + op-reth + kona carry the same Lagoon timestamp (and no op-geth node in the set) before launch; no activation-boundary **fault-proof** test.

#### FM10: Latent SDM↔Interop coupling (decoupling rename not applied)

- **Description:** `IsSDM` delegates to `IsInterop` (not directly to Lagoon) in all three clients (op-node `toggles.go:28`;
  op-reth `evm/src/lib.rs:138`; kona `rollup.rs:308`). Interop's feature toggle is deliberately kept separable from the
  Lagoon fork "so interop can be decoupled later" — the exact scenario that breaks this. If anyone decouples the interop
  toggle from `lagoon_time`, **SDM silently follows interop to the wrong activation time**, producing a consensus split
  between nodes that did/didn't pick up the change. The rename plan that gates SDM directly on Lagoon
  (`2026-06-15-sdm-toggle-lagoon-rename-plan.md`, 3 one-line edits, "very low risk") is **not merged**. A related
  cosmetic-but-dangerous issue: batch-drop diagnostics still say `PostExecPreLagoon` / "before Lagoon activation"
  (`validity.rs:46,98`, `single.rs:185`, `span.rs:515`) while the real gate is `is_sdm_active`/Interop — so a responder
  debugging stuck withdrawals would chase a non-existent "Lagoon" fork.
- **Risk Assessment:** **High impact, low likelihood.** Requires a future interop-toggle change, but the effect is a
  silent consensus fork.
- **Mitigations:**
  1. _Reduce chance:_ apply the rename plan so SDM gates directly on Lagoon.
  2. _Add:_ rename the misleading `PostExecPreLagoon` diagnostic to `PostExecPreSDM`/`PostExecPreInterop`; confirm the
     kona test-only `sdm_active_override` (`core.rs:107`, `cfg(test/test-utils)`-gated) can never be reached in release
     builds (add a CI grep gate).
- **Detection:** code review of any interop-toggle change; consensus divergence at the interop-vs-Lagoon timestamp delta.
- **Recovery Path(s):** Apply the rename (3 one-line edits). Low effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — kona delegation:_ `test_sdm_rides_lagoon` (`rollup.rs`); span-batch drop diagnostic `test_check_batch_with_post_exec_tx_pre_sdm` (`span.rs`) — still surfaces `BatchDropReason::PostExecPreLagoon`, the misleadingly-named reason the FMA flags.
  - _Unit — Go delegation:_ `TestConfig_IsSDM`, `TestActivations` (`types_test.go`) exercise `IsSDM`→`IsLagoon`.
  - _Gap:_ the coupling itself (`IsSDM` delegating through Interop rather than directly to Lagoon) is **not** guarded by any test — nothing asserts SDM activation can't drift if the interop toggle is decoupled from `lagoon_time`. The test-only `sdm_active_override` has no CI grep gate proving it's unreachable in release builds.

#### FM11: Operator opt-in flag in wrong / split state (non-persistent, multi-endpoint)

- **Description:** Producing `0x7D` requires the in-memory `SdmPostExecOptIn` flag ON. It's **default-OFF on every boot**
  and **not persisted** in op-reth (`sdm_admin.rs:8-9`); the sequencer's op-node, op-rbuilder, and op-reth each have their
  own flag and admin endpoint, all of which must agree. A restart silently reverts to OFF; flipping one endpoint isn't
  enough (the devnet tool flips only one). Result: blocks with no refunds when refunds were expected (or the reverse,
  across builder vs EL-fallback) after any routine restart.
- **Risk Assessment:** **Medium impact, high likelihood.** Default-off + no persistence + multiple endpoints means a
  restart or one-endpoint toggle routinely produces the wrong state. Crucially, this is **opt-in/policy**, orthogonal to
  consensus validity — a wrong opt-in state can't make a _valid_ block _invalid_ for verifiers (Verify mode is
  opt-in-agnostic, `builder.rs:822-834`), so the blast radius is "missing/extra refunds", not a split.
- **Mitigations:**
  1. _Reduce chance:_ persist the opt-in (op-reth persistence is explicitly "out of scope" today) or re-issue
     `admin_setSdmPostExecOptIn(true)` on every component as part of startup automation / the deploy runbook.
  2. _Add:_ wire the existing `admin_sdmStatus` (`desiredEnabled/protocolActive/effective` per node) into a post-restart
     health check; consider a chain-spec-level kill switch beyond the in-memory toggle; exercise the op-node admin RPC
     path in the acceptance suite.
- **Detection:** `admin_sdmStatus` `effective:false` post-activation; absence of `opGasRefund`/`0x7D` in produced blocks;
  the `op_rbuilder_flags_sdm_enabled` gauge reading 0 unexpectedly.
- **Recovery Path(s):** Re-issue the opt-in on every Rust process. Low effort operationally, but recurring without
  persistence.
- **Tests covering this FM (this monorepo):**
  - _Unit — Verify mode is opt-in-agnostic (`rust/op-reth/crates/payload/src/builder/tests.rs`):_ `post_exec_mode_follows_chain_regardless_of_opt_in`, `rebuilds_derived_block_with_embedded_post_exec_tx_regardless_of_opt_in` — a _valid_ embedded `0x7D` is verified for **both** opt-in values, so a wrong opt-in state can't make a valid block invalid. Production-gating: `post_exec_tx_follows_operator_opt_in` (op-rbuilder emits `0x7D` only when opted in).
  - _Unit — admin status (`op-node/`):_ `TestAdminAPISdm` (`node/server_test.go`), `TestSequencerSdmStatus` — `admin_sdmStatus` reports `desiredEnabled`/`protocolActive`/`effective`.
  - _Acceptance:_ `TestSDMEnabledPayloadAndReplayMatch` flips the flag via `admin_setSdmPostExecOptIn`.
  - _Gaps:_ no test that the op-reth opt-in is **default-OFF and not persisted across restart**; no test for the **multi-endpoint** split (op-node vs op-rbuilder vs op-reth flags disagreeing); no atomic-flag unit test for `SdmPostExecOptIn.set()/enabled()`.

### E. User-Facing & Operational

#### FM12: Synthetic 0x7D tx breaks standard user-facing RPC consumers (expose, don't filter)

- **Description:** The `0x7D` tx is a synthetic, sequencer-generated system transaction (no signature, synthesized zero
  `from`) carried in the canonical block body. **The product decision is to _expose_ it** so block explorers can
  visualize the PostExec tx — not to filter it. **Verified ground truth (corrects this FMA's earlier draft):** `0x7D` is
  **already exposed through standard eth RPC today, and nothing filters it anywhere.** The `Optimism` network sets
  `type TransactionResponse = op_alloy_rpc_types::Transaction` (`rust/op-alloy/crates/network/src/lib.rs:34`), and
  `OpTxEnvelope::PostExec` carries custom serde (`serde_post_exec_tx_rpc` / `_de`, `post_exec.rs:479-537`); op-reth's
  `OpEthApi` converts through that type, so `eth_getBlockByNumber(…, true)` / `eth_getTransactionByHash` return the
  `0x7D` tx (it does **not** error). No `post_exec`/`is_post_exec` filtering exists in flashblocks-rpc, the op-reth
  flashblocks crate, or any RPC layer — the earlier claim that "standard RPC has no serialization" and "flashblocks-rpc
  filters it" was **stale and is retracted.** The _real_ residual gap is that the exposed schema is **minimal and
  unverified end-to-end**: it serializes as `{type:"0x7d", hash, from:0x0, gas:0x0, value:0x0, input:<RLP PostExecPayload>}`
  plus standard block fields, with **(a)** refund detail left RLP-encoded inside `input` —
  `gasRefundEntries`/`version`/`blockNumber` are _not_ decoded into JSON (`transaction.rs:393`), unlike the deposit tx
  which exposes `sourceHash`/`mint`; and **(b)** **no signature fields** (unlike `serde_deposit_tx_rpc`, which flattens a
  zero signature, `deposit.rs:402,191`), so a strict EIP-2718 decoder expecting `r/s/yParity` may reject it.
- **Risk Assessment:** **Medium impact (ecosystem-compatibility, not value-at-risk or consensus), low–medium
  likelihood.** Can't cause a chain split or loss of funds — Verify mode and receipts are unaffected, and SDM-adjusted
  receipts already carry correct gas figures. Likelihood is lower than the original draft implied because serialization
  already works (no hard "unknown type" error from op-alloy-aware clients); the residual exposure is a strict third-party
  decoder choking on the **missing signature**, and explorers needing to RLP-decode `input` to render refunds.
- **Mitigations:**
  1. _Already in place:_ `0x7D` is serialized by the shared `op_alloy_rpc_types::Transaction` type across standard eth
     RPC and flashblocks-rpc (consistent, no filtering); receipts are SDM-adjusted and expose `opGasRefund` per affected
     tx (`eth/receipt.rs`), giving explorers the per-tx refund without decoding `input`.
  2. _Add (decided scope — keep schema as-is; docs + tests only):_ **document the `0x7D` RPC schema** as the integration
     contract for explorers; add **end-to-end tests** that `eth_getBlockByNumber(…, true)` / `eth_getTransactionByHash`
     return `0x7D` in that schema and that affected receipts carry the matching `opGasRefund` by index; add a
     **backward-tolerance test** decoding the response with a stock `alloy_rpc_types_eth` decoder (this is the gate that
     tells us whether the signature-less shape actually breaks strict consumers — if it fails, flattening a zero
     signature like the deposit tx becomes a follow-up); add a **flashblocks-rpc parity regression** pinning that `0x7D`
     stays exposed. Land explorer support (Blockscout / Etherscan / Tenderly / OKLink — #21110), anchored on the schema
     doc.
- **Detection:** backward-tolerance test against common client libraries; wallet/explorer error reports; block
  `transactions.length` vs user-visible tx-count mismatch; integration tests querying full blocks post-activation.
- **Recovery Path(s):** Schema is already serialized, so there's no "build from scratch" recovery; if the
  backward-tolerance test shows strict decoders break, adding a flattened zero signature is a localized op-alloy change
  (the deposit-tx pattern). Ecosystem outreach / explorer integration is the long pole. `0x7D` stays at its **canonical
  index**, so there's no filter-induced index-shift hazard for FM13.
- **Tests covering this FM (this monorepo):**
  - _Unit — `0x7D` already serializes to RPC JSON (`rust/op-alloy/crates/rpc-types/src/transaction.rs`):_ `can_serialize_post_exec_rpc_transaction` (`:377`) asserts the RPC `Transaction` emits `"type":"0x7d"`, `from:0x0`, `input:<rlp>`, and that `gasRefundEntries`/`version` are _not_ surfaced. Envelope serde round-trips: `post_exec_envelope_rpc_serde_roundtrip`, `post_exec_envelope_rpc_deserialize_ignores_extra_fields`.
  - _Unit — receipts are SDM-adjusted (`eth/receipt.rs`):_ `convert_receipts_extracts_post_exec_gas_refund_from_embedded_payload` (`:519`) — block with `0x7D` entry `(1, 77)`, receipt at tx index 1 reports `opGasRefund = 77`.
  - _Acceptance:_ `TestSDMDisabledNoRefunds` asserts `opGasRefund` absent when SDM off.
  - _Gaps (the actual FM12 work — docs + tests, schema kept as-is):_ (1) the schema is **undocumented** as an explorer integration contract; (2) **no op-reth end-to-end test** that `eth_getBlockByNumber(…, true)` / `eth_getTransactionByHash` return `0x7D` and cross-link to receipts by index; (3) **no backward-tolerance test** decoding the signature-less `0x7D` with a stock `alloy_rpc_types_eth` decoder; (4) **no flashblocks-rpc parity regression** pinning that `0x7D` stays exposed.

#### FM13: Receipt refund mis-attribution (index anchoring)

- **Description:** Receipt SDM adjustment maps refunds by transaction index
  (`payload.gas_refund_for_idx(input.meta.index)`, `eth/receipt.rs:104-106`); `SDMGasEntry{index,...}` is anchored to a
  position in the canonical block. If the RPC's tx ordering/index differs from what the payload anchored to (flashblocks
  materialization ordering, a reorg, or index shift from filtering the `0x7D` tx), a refund can be attributed to the
  wrong tx — users see wrong `gasUsed`/effective cost.
- **Risk Assessment:** **Medium impact, low likelihood.** Index anchoring is consistent within canonical blocks;
  flashblocks materialization and any index-shifting filtering are the risk surface. Exploitability of an index shift
  wasn't demonstrated (theoretical).
- **Mitigations:**
  1. _Reduce chance:_ `parse_post_exec_payload_from_transactions` validates
     present-before-activation/duplicate/not-last/wrong-block-number (`builder.rs:801-820`); flashblocks-rpc skips `0x7D`
     when building receipt/nonce caches.
  2. _Add:_ a cross-check that the receipt index space equals the index space the payload anchored to after any
     filtering.
- **Detection:** refund attributed to a tx that didn't touch refundable state; receipt `gasUsed` not matching execution;
  `debug_replaySDMBlock` disagreement.
- **Recovery Path(s):** Correct the index mapping / re-derive from the canonical block. Medium effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — index-anchored refund lookup:_ `convert_receipts_extracts_post_exec_gas_refund_from_embedded_payload` (`eth/receipt.rs`) exercises `gas_refund_for_idx(meta.index)`; `parse_accepts_trailing_post_exec_tx` parses an entry at index 1 / refund 9; encoding round-trips with multiple entries — `post_exec_tx_eip2718_roundtrip` (`[(0,100),(5,200)]`), `post_exec_payload_rlp_roundtrip_preserves_block_number`.
  - _Acceptance:_ `TestSDMStorageRefundBreakdown`, `TestSDMEnabledPayloadAndReplayMatch` read `opGasRefund` per-tx and cross-check against `debug_replaySDMBlock`.
  - _Gaps:_ no test for the index-anchoring failure itself (a refund attributed to the wrong tx after index shift from filtering `0x7D` or flashblocks reordering); no `gas_refund_for_idx` off-by-one / `None`-on-missing-index test; no cross-check that the receipt index space equals the payload's anchored index space after filtering.

#### FM14: Admin RPC abuse / exposure

- **Description:** SDM opt-in lives in the `admin` namespace on op-node, op-reth, and op-rbuilder. An attacker (or a
  misconfigured, exposed admin port) calling `admin_setSdmPostExecOptIn(false)` on the sequencer/builder silently stops
  refund production; `(true)` prematurely on a builder could emit `0x7D` before intended. The admin namespace also holds
  sequencer start/stop, so any exposure is already high-severity.
- **Risk Assessment:** **Medium impact, low likelihood.** Requires an exposed/unauthenticated admin RPC (operational
  misconfig). Can't break consensus validity (opt-in is orthogonal to Verify).
- **Mitigations:**
  1. _Reduce chance:_ the plan gates SDM methods behind `--rpc.enable-admin`, but it was **not verified** that the
     op-reth SDM handler actually sits behind that gate (`sdm_admin.rs:36-47` registers into the admin namespace; the
     enable-admin check wasn't located there) — confirm and test.
  2. _Add:_ firewall admin RPC by default; access logging / audit on `admin_setSdmPostExecOptIn`.
- **Detection:** unexpected `admin_sdmStatus` flips; refund volume dropping to zero; admin-RPC access logs.
- **Recovery Path(s):** Firewall/disable admin RPC, rotate access, re-set the desired flag. Low effort.
- **Tests covering this FM (this monorepo):**
  - _Unit — admin status surface (`op-node/`):_ `TestAdminAPISdm` (`node/server_test.go`) exercises `SetSdmPostExecOptIn`/`SdmStatus`; `TestSequencerSdmStatus` checks the computed status fields.
  - _Gap (NON-BLOCKING):_ the op-reth SDM admin handler (`sdm_admin.rs`) has **zero test coverage** — nothing verifies `admin_setSdmPostExecOptIn`/`admin_sdmStatus` are gated behind `--rpc.enable-admin`, and there's no access-logging/audit test.

#### FM15: Policy-abstraction misconfiguration (silent economic error)

- **Description:** The policy-abstraction plan splits `sdm_enabled` into `sdm_post_exec_enabled` (verifier accepts `0x7D`)
  and `sdm_policies` (sequencer-local production policy, e.g. block-warming + contract-refund). Policies are **not**
  encoded in the payload (it carries only aggregate `SDMGasEntry`). A misconfigured policy set (wrong contract address,
  wrong bps, wrong ordering) produces refunds the operator didn't intend; because verifiers only check
  `refund ≤ gas_used`, the bad refunds are accepted as **valid** — a silent economic error, not a consensus failure, and
  undetectable from the chain alone.
- **Risk Assessment:** **Medium impact, low likelihood.** The configurable-policy path (#97) isn't yet code-complete
  (only deterministic block-warming exists today), so this is largely prospective.
- **Mitigations:**
  1. _Reduce chance:_ the `refund ≤ evm_gas_used` invariant caps damage; the aggregate-only payload keeps consensus
     simple.
  2. _Add:_ refund-volume/anomaly dashboards (unticketed, hardfork-estimate Gap #3); reconciliation of
     expected-vs-actual refunds against the configured policy; don't conflate accept-vs-produce in a single
     `sdm_enabled` flag.
- **Detection:** refund-anomaly dashboards; reconciliation against the configured policy.
- **Recovery Path(s):** Correct the policy config and redeploy the sequencer; no chain rollback (the refunds were valid).
  Low config fix; the larger gap is observability.
- **Tests covering this FM (this monorepo):**
  - _Unit — the `refund ≤ evm_gas_used` cap that bounds policy damage:_ `test_verifier_rejects_refund_exceeding_evm_gas` and the kona equivalent `post_exec_payload_rejects_refund_exceeding_gas_used` — the only consensus-level guard against a misconfigured policy.
  - _Gap (largely prospective):_ the configurable-policy split (`sdm_post_exec_enabled` vs `sdm_policies`) is **not yet implemented** — only a single binary `SdmPostExecOptIn` flag exists (`config.rs`), so there are no policy-variant or expected-vs-actual-refund reconciliation tests. Production observability is unticketed.

### Generic items we need to take into account

See [generic hardfork failure modes](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md)
and [generic smart contract failure modes](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).
SDM ships as part of a hardfork (Lagoon) but adds **no new L1/L2 contracts** — its surface is EVM/client consensus,
derivation, proofs, and RPC. The hardfork-generic items (activation timing, client-version coordination, chain-split
detection, rollback) are addressed in FM1, FM6, FM8, FM9, FM10. The contract-generic FMA is largely N/A.

- [ ] Check this box to confirm the generic hardfork and contract failure modes have been reviewed and incorporated.

## Action Items

What needs to happen before launch — to reduce the chance of the above failure modes and to make them detectable and
recoverable. **BLOCKING** items must be resolved before SDM activates on any chain.

**Consensus & refund math**

- [ ] Resolve all comments on this document and incorporate them (Assignee: document author).
- [x] (RESOLVED 2026-06-19) **Aggregate refund cap (FM3): decided NOT to implement.** No percentage-based cap; the cap is `evm_gas_used` — the per-tx bound `refund ≤ evm_gas_used` plus pre-refund gas-limit enforcement supersede it. `MAX_REFUND_BPS` / `sdm-refund-cap-plan.md` closed. Remaining mechanical step: mark `sdm-refund-cap-plan.md` superseded.
- [ ] (BLOCKING) Confirm/add the **`finish()` pre-refund debug-assert** (FM3) — currently absent from `block/mod.rs:1077-1106` (Assignee: TBD).
- [ ] (BLOCKING) Write a **canonical spec for the application & validation semantics** verifiers must share — settlement math, `gas_used = evm_gas_used − refund`, receipt adjustment, the per-tx bound, and the structural rules — so the Go derivation/decode checks and the kona proof program can't drift from op-reth Verify mode (FM1) (Assignee: TBD).
- [ ] (BLOCKING) Add a **differential/replay test in CI** asserting op-reth Verify mode and the kona proof program apply the same `0x7D` payload to the same block hash, and that the Go structural checks accept/reject the same payloads as the Rust verifier, for non-empty refund sets (FM1) (Assignee: TBD).
- [ ] (NON-BLOCKING) Spec the warming **exclusion set** / refund constants as _sequencer policy_ (builder-consistency + economics, not consensus), so op-reth and op-rbuilder builders stay in sync (FM2/FM5) (Assignee: TBD).
- [ ] (NON-BLOCKING) Add a test pinning that saturating settlement arithmetic can't silently truncate under max realistic inputs (FM2) (Assignee: TBD).
- [ ] (NON-BLOCKING) Add full structural validation to the Go `DecodePayload`, or explicitly mark it non-authoritative (FM4) (Assignee: TBD).

**Block building & derivation**

- [ ] (BLOCKING) Port the SDM activation gate + positional/count/block-number-anchor checks into the **span-batch** derivation loop and add a pre-encode defensive check; add negative acceptance tests (FM6) (Assignee: TBD).
- [ ] (BLOCKING) Merge the **overlay-SDM-aware** fix so all materializers re-attach `0x7D` via one shared helper with identical "latest `Some`" selection; add a builder-side non-`None`-final assertion (FM5) (Assignee: TBD).
- [ ] (BLOCKING) Implement hash-keyed flashblock selection and **un-skip** `TestFlashblocksSDMMaterializesPostExecBlock` (FM7) (Assignee: TBD).
- [ ] (NON-BLOCKING) Add a builder-view ≠ seal production metric/alarm (FM5) (Assignee: TBD).
- [ ] (NON-BLOCKING) Add a cross-flashblock cumulative pre-refund-overrun test (FM3) (Assignee: TBD).

**Fault proofs**

- [ ] (BLOCKING) Generate a **post-Isthmus non-zero-refund fixture** via `ExecutorTestFixtureCreator` against an SDM-enabled op-reth node and add an executor test asserting kona hash == op-reth hash. Note (verified 2026-06-19): the existing pre-Isthmus testdata fixtures can't be retrofitted — a non-zero refund on them panics on the unset operator-fee scalar, and forcing Isthmus active fails on missing trie preimages (FM8) (Assignee: TBD).
- [ ] (NON-BLOCKING) Guard the SDM settlement's `operator_fee_charge` call against an unset operator-fee scalar (or assert the SDM ⊃ Isthmus invariant) so it can't `.expect()`-panic in any non-production config (`block/mod.rs:698-703`) (FM8) (Assignee: TBD).
- [ ] (BLOCKING) Add a **Cannon/FPVM-mode SDM proof run** to acceptance and ensure the shipped absolute prestate covers SDM-active behavior (FM8) (Assignee: TBD).
- [ ] (BLOCKING) Add `MixedL2CLKona` to the SDM acceptance suite so kona-node SDM is exercised in CI by default (FM5/derivation) (Assignee: TBD).

**Activation & configuration**

- [ ] (BLOCKING) Add a **cross-component activation preflight** (op-node / op-reth / kona Lagoon timestamps agree, and no op-geth node in the set — op-geth is EOL and won't support SDM) and an activation-boundary fault-proof test (FM9) (Assignee: TBD).
- [ ] (BLOCKING) Apply the **SDM→Lagoon decoupling rename** so SDM gates directly on Lagoon, not Interop (FM10) (Assignee: TBD).
- [ ] (NON-BLOCKING) File the superchain-registry Lagoon-timestamp ticket / fork schedule for target chains (FM9) (Assignee: TBD).
- [ ] (NON-BLOCKING) Rename the misleading `PostExecPreLagoon` diagnostic; add a CI grep gate that `sdm_active_override` is unreachable in release builds (FM10) (Assignee: TBD).
- [ ] (NON-BLOCKING) Persist the operator opt-in, or add startup-automation re-issue + a post-restart `admin_sdmStatus` health check (FM11) (Assignee: TBD).

**User-facing & operational**

- [ ] (BLOCKING) **Document** the existing `0x7D` RPC schema (`{type:"0x7d", hash, from:0x0, gas:0x0, value:0x0, input:<RLP payload>}`) as the explorer integration contract, and add **end-to-end + backward-tolerance + flashblocks-rpc-parity tests** — `0x7D` is already exposed and unfiltered today; the gap is verification and documentation, not serialization (FM12) (Assignee: TBD).
- [ ] (NON-BLOCKING) If the backward-tolerance test shows strict decoders reject the signature-less `0x7D`, flatten a zero signature à la `serde_deposit_tx_rpc` (FM12) (Assignee: TBD).
- [ ] (NON-BLOCKING) Land explorer/wallet support and outreach — Blockscout PRs + Etherscan/Tenderly/OKLink (#21110) (FM12) (Assignee: TBD).
- [ ] (NON-BLOCKING) Confirm SDM admin methods are behind `--rpc.enable-admin`; add access logging (FM14) (Assignee: TBD).
- [ ] (NON-BLOCKING) Stand up refund-volume/anomaly dashboards (FM3, FM15) (Assignee: TBD).

## Audit Requirements

**Recommendation: yes — this needs an external audit before mainnet activation.** Per the
[FMAs in the SDLC](https://github.com/ethereum-optimism/pm/blob/main/src/fmas.md#determine-audit-requirements) framework,
SDM is consensus-critical and high-severity along several axes the decision framework weights heavily:

- It changes **EVM-level value flow** (refunds reduce fee-vault income), with a demonstrated bug class (intrinsic-warm
  fee-vault drain, Immunefi #81847) and a value-conservation/mint risk surface (FM2).
- It introduces a **new transaction type (`0x7D`)** whose structural validation must hold identically across Rust, Go,
  and kona (FM4), and whose refund math must match across multiple independent client implementations and the
  fault-proof program (FM1, FM8) — divergence is a chain split or unprovable output root.
- It touches **derivation** (span-batch gating, FM6) and the **fault-proof program** (FM8), the components most directly
  tied to the Stage-1 security guarantee.

Audit scope: the `alloy-op-evm` refund/settlement/intrinsic-warm logic; the `0x7D` structural validation across all
three language implementations; the derivation gating (singular + span batch); and the FPVM/Cannon proof path for
SDM-active blocks. No L1/L2 contracts to cover (none are added). A re-audit is warranted if the aggregate refund cap
(FM3) is ever implemented as a new consensus rule.

## Appendix

### Appendix A: Verification environment & method

- **Monorepo:** `optimism2` @ branch `test/sdm-settlement-conservation`, HEAD `db5c50cc86`
  (`fix(alloy-op-evm): silence clippy field_reassign_with_default in SDM tests`).
- **Source notes:** `~/sdm-docs` git working copy as of 2026-06-16.
- **Method:** failure modes were mined from the working notes by four parallel investigators
  (refund-math/consensus; fault-proofs/kona; flashblocks/derivation; activation/toggle/RPC), and each claim was checked
  against the live branch via grep + file reads. File:line references are to the branch HEAD above. Anything that
  couldn't be confirmed in code is marked "(unverified)".
- **Reproduction tip (flashblocks SDM test):** build only what the test needs; do **not** run `just test`/`build-deps`
  (it builds the cannon/kona prestate, which needs the MIPS cross-compiler). Use:
  `cd op-acceptance-tests && RUST_JIT_BUILD=1 mise exec -- go test -count=1 -timeout 30m -v -run '^TestFlashblocksSDMMaterializesPostExecBlock$' ./tests/flashblocks/`.

### Appendix B: Doc-vs-code discrepancies found while writing this FMA

Places where the working notes and the live branch disagree — useful for reviewers, and worth resolving:

1. **Aggregate refund cap — decided not to implement (resolved 2026-06-19).** `sdm-refund-cap-plan.md` describes `MAX_REFUND_BPS`, but it was never implemented (only a stray comment at `tests.rs:1066`) and **won't be**: the cap is `evm_gas_used` (per-tx bound) plus pre-refund gas-limit enforcement. Plan superseded/closed. (FM3)
2. **`finish()` pre-refund debug-assert absent.** The `max-gas-limit` notes describe a `finish()` assert; the current `finish()` (`block/mod.rs:1077-1106`) only checks unconsumed entries. (FM3)
3. **Span-batch derivation gate missing.** Singular-batch path gates `0x7D` (`batches.go:200`); the span-batch loop (`batches.go:399-413`) has no SDM/PostExec handling at all. (FM6)
4. **op-alloy overlay drops `0x7D`.** `from_flashblocks_unchecked` (`envelope.rs:217-269`) never appends `post_exec_tx`; the SDM-aware fix is on an unmerged branch. (FM5)
5. **Flashblocks SDM test is `t.Skip`'d** (`flashblocks_sdm_test.go:52`) despite already carrying the disambiguating blockHash assertion (`:215`). _(Now resolved on this branch — see FM7.)_ (FM7)
6. **Decoupling rename not applied.** `IsSDM` still delegates to `IsInterop` in all three clients despite the 2026-06-15 plan. (FM10)
7. **Misleading `PostExecPreLagoon` naming** persists in kona batch-drop diagnostics while the real gate is `is_sdm_active`/Interop. (FM10)
8. **`0x7D` is already exposed; the original "leak/filter" framing was wrong (retracted).** Re-verification on `develop` shows the `Optimism` network's `TransactionResponse = op_alloy_rpc_types::Transaction` already serializes `0x7D` via custom serde (`post_exec.rs:479`), so standard eth RPC returns it (doesn't error), and **nothing filters it** in flashblocks-rpc or any RPC layer. The earlier "standard RPC has no serialization / flashblocks-rpc filters it" was stale. Real gap: the schema is **minimal** (payload RLP-only in `input`; **no signature fields**, unlike the deposit tx) and **unverified end-to-end**. Per the decision to keep the schema as-is, the work is documentation + tests, with a zero-signature follow-up only if the backward-tolerance test fails. (FM12)
9. **Stale tooling:** `sdm-devnet` / the devnet doc reference `admin_setSdmEnabled`; the live method is `admin_setSdmPostExecOptIn`, and the tool flips only one endpoint. (FM11)
10. **Stale caveat (resolved):** the kona-node plan's "missing prestate artifacts" note is stale — artifacts now exist at `rust/kona/prestate-artifacts-cannon/`. (FM8)
