# Adopt EIP-8130 on the OP Stack: Failure Modes and Recovery Path Analysis

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Chris Hunter                                       |
| Created at         | 2026-04-27                                         |
| Initial Reviewers  | TBD                                                |
| Need Approval From | TBD                                                |
| Status             | Draft                                              |

## Introduction

This document analyzes failure modes for adopting
[EIP-8130](https://eips.ethereum.org/EIPS/eip-8130) on the OP Stack. The feature introduces a new
EIP-2718 account abstraction transaction type, Account Configuration storage, canonical verifiers,
new precompiles, sponsored transactions, account creation, phased call execution, and new RPC and
receipt surfaces.

This FMA covers the OP Stack adoption risks. It does not replace the upstream EIP security review or
audits of the Account Configuration contract, default account implementation, or `op-reth`
implementation.

References:

- Design doc: [Adopt EIP-8130 on the OP Stack](../protocol/adopt-eip-8130-on-op-stack.md)
- Upstream spec: [EIP-8130](https://eips.ethereum.org/EIPS/eip-8130)
- Generic hardfork FMA: [fma-generic-hardfork.md](./fma-generic-hardfork.md)
- Generic contracts FMA: [fma-generic-contracts.md](./fma-generic-contracts.md)

## Summary

| Severity Level | Outcome | Likelihood | Key Mitigations |
| --- | --- | --- | --- |
| High | Invalid block acceptance or consensus implementation bug | Possible | Pin spec artifacts, consensus test vectors, devnets, multiple audits |
| High | Account takeover or stranded accounts | Possible | Audit Account Configuration, owner scopes, default account, and wallet UX |
| Medium | Mempool DoS or sequencer validation overload | Possible | Bounded canonical verifier set, mempool limits, defer nonceless mode |
| Medium | Receipt phase status misinterpreted | Likely | RPC/receipt compatibility testing, Base-owned integration checklist |
| Medium | Cross-chain owner changes misunderstood | Possible | Explicit docs and tests for `chain_id = 0` portability |

## Failure Modes and Recovery Paths

### FM1: Consensus Implementation Bug

- **Description:** `op-reth` incorrectly implements transaction decoding, signature hashing,
  intrinsic gas, verifier behavior, account changes, nonce handling, receipt fields,
  protocol-injected logs, or precompile behavior. This can produce invalid block acceptance or an
  execution rule that must be fixed by client release or hardfork.
- **Risk Assessment:** High severity, possible likelihood.
- **Mitigations:**
  - Pin all upstream EIP constants, bytecode, storage layouts, fixed addresses, precompile behavior,
    and RPC/receipt shapes before activation.
  - Create positive and negative consensus vectors for every transaction path.
  - Run devnets with the final `op-reth` implementation.
  - Complete multiple audit cycles across spec, `op-reth`, contracts, and wallet flows.
- **Detection:**
  - Hive-style tests and fuzzing for transaction parsing and execution.
  - Devnet and testnet monitoring for unexpected state roots, receipt roots, logs, or transaction
    rejection behavior.
- **Recovery Path(s):**
  - Before mainnet activation, fix implementations and regenerate vectors.
  - After activation, recovery may require emergency client releases or a hardfork depending on the
    bug.

### FM2: Account Takeover or Stranded Accounts

- **Description:** Bugs in owner authorization, owner revocation, owner scope, account lock,
  auto-delegation, default account bytecode, or wallet UX allow the wrong party to control an
  account or prevent the legitimate user from recovering control.
- **Risk Assessment:** High severity, possible likelihood.
- **Mitigations:**
  - Audit Account Configuration bytecode, storage layout, owner scope checks, account lock lifecycle,
    and default account implementation.
  - Make auto-delegation explicit in wallet UX.
  - Test implicit EOA owner behavior, revocation, lock/unlock flows, and delegation clearing.
  - Treat `chain_id = 0` portable owner changes as an explicit audit focus.
- **Detection:**
  - Devnet and testnet scenarios for owner rotation, revocation, lock/unlock, import, and
    auto-delegation.
  - Indexer checks for owner and delegation events.
  - User support and wallet telemetry during testnet rollout.
- **Recovery Path(s):**
  - User-level recovery may be possible through configured recovery owners, unlock flows, or
    delegation clearing.
  - Widespread account-stranding bugs may require wallet releases, contract mitigations, or a
    hardfork depending on the root cause.

### FM3: Mempool DoS or Sequencer Validation Overload

- **Description:** Attackers submit transactions that are expensive to validate, hard to invalidate,
  or cause excessive pre-inclusion work through complex account changes, payer auth, nonceless mode,
  or verifier behavior.
- **Risk Assessment:** Medium severity, possible likelihood.
- **Mitigations:**
  - Keep the canonical verifier set conservative and bounded.
  - Define mempool policy limits for owner changes, payer authorization complexity, transaction
    shape, and validation cost.
  - Defer `NONCE_KEY_MAX` nonceless mode unless client teams agree it is ready.
  - If nonceless mode is enabled, require short expiry windows and duplicate-hash exclusion by
    builders.
- **Detection:**
  - Sequencer validation latency and rejection-rate metrics.
  - Devnet load tests with invalid transactions and adversarial transaction shapes.
  - Mempool profiling for verifier and account-change costs.
- **Recovery Path(s):**
  - Tighten mempool policy and release updated node software.
  - Tighten or defer nonceless mode where policy permits.

### FM4: Receipt Phase Status Misinterpreted

- **Description:** Wallets, SDKs, explorers, indexers, and testing frameworks mishandle the new
  receipt status model. 8130 receipts can include `phaseStatuses`, so tools cannot treat the receipt
  as only a single success/failure value when displaying or indexing phased transactions.
- **Risk Assessment:** Medium severity, likely likelihood.
- **Mitigations:**
  - Base owns wallet, SDK, and indexer compatibility for its rollout.
  - Include receipt `status` and `phaseStatuses` in compatibility tests.
  - Document how completed phases can persist even when a later phase fails.
- **Detection:**
  - Devnet integration tests across wallets, SDKs, explorers, indexers, and local dev tooling.
  - Testnet monitoring for malformed transaction displays and incorrect phase-status indexing.
- **Recovery Path(s):**
  - Release tooling and SDK fixes before mainnet activation.
  - If compatibility gaps are severe, delay activation until critical integrations are ready.

### FM5: Cross-Chain Owner Changes Misunderstood

- **Description:** Config changes with `chain_id = 0` are intentionally portable across chains. If
  wallets, users, or operators treat these as chain-local changes, owner updates may apply more
  broadly than expected.
- **Risk Assessment:** Medium severity, possible likelihood.
- **Mitigations:**
  - Document portable owner changes clearly in wallet and developer guidance.
  - Test both `chain_id = 0` and chain-specific config changes.
  - Require wallet UX to distinguish portable owner changes from chain-local owner changes.
- **Detection:**
  - Devnet tests that apply portable owner changes across multiple chains.
  - Indexer checks that surface whether config changes are portable or chain-specific.
- **Recovery Path(s):**
  - Users can apply follow-up owner changes to restore intended owner configuration.
  - Widespread UX issues should block mainnet rollout until wallet behavior is fixed.

## Action Items

Below is what needs to be done before launch to reduce the chance of the above failure modes and to
ensure they can be detected and recovered from:

- [ ] Resolve all comments on this FMA and incorporate them into the document.
- [ ] Pin final EIP constants, fixed addresses, bytecode artifacts, storage layouts, precompile
  behavior, RPC shapes, and receipt fields.
- [ ] Decide whether the first OP Stack rollout supports or defers `NONCE_KEY_MAX` nonceless mode.
- [ ] Define mempool policy limits for validation cost, owner changes, payer authorization, and
  transaction shape.
- [ ] Produce consensus, RPC, receipt, account-change, and nonce test vectors.
- [ ] Run devnets with wallet, SDK, explorer, indexer, sequencer, batcher, and `op-reth`
  compatibility tests.
- [ ] Complete multiple audit cycles covering the upstream EIP, OP Stack implementation, Account
  Configuration contract, default account implementation, and wallet flows.
- [ ] Assign owners for monitoring, incident response, and launch readiness signoff.

## Audit Requirements

This feature should require multiple audits. It changes consensus behavior, introduces new account
authorization paths, adds precompile behavior, depends on consensus-critical contract bytecode and
storage layout, and creates new wallet flows that can affect user funds.

Audit scope should include:

- upstream EIP semantics and specification completeness
- `op-reth` implementation and consensus tests
- Account Configuration contract and storage layout
- default account implementation and auto-delegation behavior
- wallet and SDK integration flows
- RPC, receipt phase status, and protocol-injected log compatibility

## Appendix

### Appendix A: Initial Rollout Assumptions

The current adoption proposal assumes:

- 8130 activates in a named OP Stack hardfork.
- Deposit transactions and OP Stack system transactions do not receive special 8130 behavior in the
  first rollout.
- The canonical verifier set is limited to Secp256k1, P256, P256 WebAuthn/passkey, and Delegate.
- Unknown or stateful verifier acceptance is outside the OP Stack 8130 conformance surface for the
  first rollout.
- Delegated-account behavior should match the delegated-EOA model already established for EIP-7702.
- Standard and user-defined nonce keys are supported; `NONCE_KEY_MAX` may be deferred.
- Wallet, SDK, and indexer compatibility for the Base rollout is owned by Base.
