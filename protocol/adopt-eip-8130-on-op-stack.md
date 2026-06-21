# Adopt EIP-8130 on the OP Stack

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Chris Hunter                                       |
| Created at         | 2026-04-13                                         |
| Initial Reviewers  | TBD                                                |
| Need Approval From | TBD                                                |
| Status             | Draft                                              |

## Purpose

This document proposes adopting [EIP-8130](https://eips.ethereum.org/EIPS/eip-8130) on OP Stack
L2s and identifies the protocol, client, and tooling work required to ship it safely.

## Summary

EIP-8130 introduces a new typed transaction and an Account Configuration system that enables
native account abstraction with custom authentication, batching, gas sponsorship, and portable
accounts and owner management.

We propose that the OP Stack adopt 8130 as an L2 protocol feature in a hardfork. The
implementation should target semantic compatibility with the upstream EIP wherever possible and
should not require any OP Stack-specific divergence, as no EVM changes / opcodes are needed.

This document is an adoption proposal, not a replacement for the upstream 8130 spec. The EIP
should remain the normative source for exact transaction encoding and execution behavior.

## Problem Statement + Context

Existing account abstraction solutions such as ERC-4337 address many UX problems but do so outside
of the base transaction model. That approach requires specialized mempools, bundlers, and
validation of wallet behavior before inclusion. This approach adds latency, cost, and integration burden.

EIP-8130 offers a native account abstraction standard. Authentication is separated from wallet logic
through verifier contracts and account configuration storage. Transactions explicitly identify the
sender, payer, and authentication path. This gives sequencers and nodes better visibility into
validation cost and opens a path to native support for passkeys, sponsorship, multisig, and new
signature schemes without forcing every validator to simulate arbitrary wallet code during
transaction acceptance.

Other account abstraction approaches can mitigate some of the same UX problems, but they often do
so with more complex mempool rules and a less uniform model across account types. 8130 instead
provides a standard transaction surface for EOAs, existing smart accounts, and new smart accounts
while keeping validation explicit and comparatively simple.

That simplicity matters for both nodes and builders. In proposals where validity depends on broader
wallet execution or more dynamic state, ingress may require simulation and tracing just to
determine whether a transaction is admissible, and builders take on more wasted compute and packing
risk when invalidation is harder to track. 8130 aims to directly address these issues while still providing 
flexibility. It also gives wallets a clearer standard to target for cross-chain support, and directly 
includes features we care about such as 2D nonces, key rotation, and transaction expiration.

## Proposed Solution

We propose to adopt EIP-8130 on the OP Stack via a scheduled hardfork and to treat the upstream
EIP as the source of truth for execution semantics.

### Scope of Adoption

The OP Stack implementation should include the core 8130 feature set:

- the new AA transaction type
- the Account Configuration system contract
- the Nonce Manager precompile (if offering 2D or nonceless tx)
- the Transaction Context precompile 
- account creation, config changes, and delegation entries in `account_changes`
- payer authorization and sponsored transactions
- call phases and receipt extensions
- protocol-injected logs and RPC extensions described by the EIP

The goal should be to ship 8130 as an additive feature. Existing EOAs, smart contracts, and
ERC-4337 flows should continue to work. 8130 should provide a native path for wallets that want
protocol-level account abstraction, not require the ecosystem to abandon existing wallet patterns.

### OP Stack Integration

Adopting 8130 on the OP Stack requires coordinated work across the protocol and the surrounding
stack:

- `op-reth` must implement the new transaction type, intrinsic gas accounting, validation flow,
  state transition logic, precompile behavior, receipt fields, and protocol-injected logs.
- `op-node`, sequencer transaction ingestion, and batch submission paths must support the new typed
  transaction end to end.
- fixed addresses for the Account Configuration contract, Nonce Manager, Transaction Context,
  initial verifier set, and default account implementation must be standardized for chains
  that opt into the feature. These addresses are TBD until the upstream spec is final.
- SDKs, wallets, explorers, and indexers must be updated to understand the new transaction type,
  receipt shape, and RPC extensions.

The required contract surface should remain limited. Once the final 8130 spec is agreed upon, OP
Stack chains should include the Account Configuration contract and the initial verifier set at
standard addresses. The remaining required functionality is expected to be precompile-based.

### Spec Dependencies to Pin

The OP Stack adoption should be conditional on pinning the final upstream EIP values and artifacts
before activation:

- `AA_TX_TYPE` and payer domain-separation byte
- fixed addresses for the Account Configuration contract, Nonce Manager precompile, Transaction
  Context precompile, initial verifier set, and default account implementation
- Account Configuration contract bytecode and storage layout
- canonical verifier bytecode and deployment process
- default account implementation bytecode
- precompile interfaces and gas behavior
- receipt fields and RPC extension shapes

### Activation, Derivation, and System Transactions

8130 should activate in a named OP Stack hardfork. The fork name and activation schedule are TBD.
Before activation, the new transaction type must be rejected. After activation, it should be
accepted as a standard [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) typed transaction.

Because 8130 is an EIP-2718 transaction type, no special derivation rule changes are expected.
Batches should continue to carry opaque transaction bytes, and execution validity should be enforced
by the execution client after activation. `op-node` and the batcher still need compatibility work so
that the new transaction type is accepted, propagated, included, and decoded correctly throughout the
stack.

Deposit transactions and OP Stack system transactions should not have special 8130 behavior in the
initial adoption. They should continue to use their existing transaction paths. Future upgrades may
choose to leverage 8130 for deposits or system transactions, but that is not part of this proposal.

### OP Stack Policy

8130 allows nodes to maintain a verifier allowlist, and the OP Stack should help define a canonical
verifier set as part of the standard. This verifier set should be agreed upon by the client and
chain teams implementing 8130.

OP Stack chains should agree on which verifiers belong in this canonical set, and chains that claim
conformance with the standard must accept every verifier in the set. This gives wallets and SDKs a
single standard baseline to target across chains.

Chains may choose to enable additional verifiers beyond the canonical set, but that behavior should
be treated as "out of spec". Wallets and applications should not assume those extensions are
available on other OP Stack chains.

The OP Stack will adopt the exact 8130 canonical verifier set agreed upon by the teams implementing
the standard. For an initial rollout, the canonical set is planned to be conservative:

- Secp256k1
- P256
- P256 WebAuthn / passkey
- Delegate, enabling a sub-account model

No other validation paths are considered part of the OP Stack 8130 spec at this time. Additional
canonical verifiers would need to be agreed upon by the teams implementing 8130 and, ideally,
standardized upstream.

Protocol-native verifier paths are acceptable when their behavior and standard gas costs are defined
at the fork. Delegate should charge an additional SLOAD cost for resolving the delegated owner, plus
the underlying wrapped signature verification cost. Verifier contracts themselves must not be
delegated accounts.

### Consensus and Mempool Validation Boundary

The consensus requirements for an included 8130 transaction should be lighter and more mechanical
than the full mempool policy. At minimum, a valid included 8130 transaction means:

- `sender_auth` was valid at inclusion time
- `payer_auth`, when present, was valid at inclusion time
- the payer had enough ETH to cover the transaction
- the nonce was valid under the chain's supported nonce mode
- `account_changes` were valid
- sender authentication was validated again at the end of the flow

Other limits should be treated as mempool policy and agreed upon by client teams. Examples include
limits on owner changes, payer authorization complexity, and other transaction shape constraints
that prevent expensive or pathological validation before inclusion.

The OP Stack may choose whether to support nonceless 8130 transactions. An initial deployment could
also defer nonceless mode if that produces a simpler and safer first fork.

### Account Semantics Adopted from 8130

The OP Stack should explicitly adopt the following EIP semantics:

- a code-empty EOA that sends its first 8130 transaction is auto-delegated to
  `DEFAULT_ACCOUNT_ADDRESS` unless the transaction creates the account or supplies its own
  delegation entry
- calls carry no protocol-level ETH value; ETH transfers must be initiated by wallet bytecode
- each call executes with `msg.sender = from`, `tx.origin = from`, and `msg.value = 0`
- calls execute in ordered phases; each phase is atomic, completed phases persist, and later phases
  are skipped after a phase failure
- owner scope checks and account lock behavior are consensus-critical
- config changes with `chain_id = 0` are intentionally portable across chains and should be treated
  as an audit focus

Auto-delegation is a significant UX and safety surface. The default account implementation must be
pinned and audited before activation, and wallet UX should make clear when a user's account will be
delegated by sending an 8130 transaction.

### Delegation and Existing OP Stack Semantics

8130 reuses the 7702 delegation-indicator model. This means adoption must be checked against existing
OP Stack logic that treats EOAs and contracts differently.

At minimum, implementation work should explicitly review:

- address aliasing behavior for delegated accounts
- bridge convenience methods and any EOA-only checks
- assumptions about `tx.origin`, `msg.sender`, and code presence
- tooling that currently treats "has code" and "is a contract" as equivalent

The expected direction is that OP Stack behavior should remain internally consistent with the
mental model already established for delegated EOAs: a delegated account may have code, but it is
not equivalent to a general smart contract for every protocol decision.

### Rollout

Because 8130 touches consensus logic, node policy, and user-facing RPC behavior, it should be
rolled out in stages:

1. ratify the adoption design and complete a companion FMA
2. choose constants and fixed addresses in coordination with the upstream EIP
3. prototype the feature in `op-reth`
4. validate mempool behavior, tooling compatibility, and verifier policy in devnets
5. complete multiple audit cycles across the EIP, OP Stack implementation, verifier contracts, and
   wallet integration flows
6. ship in a named OP Stack hardfork after testnet coverage and async review

The devnet checklist should include:

- positive and negative transaction validity vectors for the new EIP-2718 transaction type
- verifier acceptance and rejection behavior for the canonical verifier set
- payer authorization, insufficient-balance, nonce, expiry, and `account_changes` failure cases
- receipt fields, protocol-injected logs, and RPC compatibility
- `eth_getAcceptedVerifiers`, extended `eth_getTransactionCount` with `nonceKey`, receipt `payer`,
  and receipt `phaseStatuses`
- mempool policy limits for transaction shape, owner changes, payer authorization, and validation
  cost
- auto-delegation to the default account implementation and explicit delegation override behavior
- owner scope checks, account lock behavior, and portable `chain_id = 0` config changes
- phased call execution, including `msg.value = 0`, persisted completed phases, and skipped later
  phases after failure
- wallet, SDK, explorer, and indexer integration tests owned by Base for the initial rollout

Implementation ownership should be made explicit before approval:

- `op-reth` implementation and consensus tests
- `op-node`, sequencer ingestion, and batcher compatibility
- Account Configuration contract, initial verifier set, precompile behavior, and fixed-address
  assignments
- wallet, SDK, explorer, and indexer support
- security review, audit scheduling, and FMA signoff

### Resource Usage

8130 increases protocol complexity relative to standard 1559-style transactions. Validation can
include verifier execution, payer authorization, config-change simulation, and nonce-channel
handling. Restricting acceptance to the canonical verifier set keeps these costs bounded and
predictable; unknown or stateful verifier paths are explicitly out of scope for the first rollout
because they would materially increase validation complexity.

The proposal also increases state usage. Owner configuration consumes one slot per owner, accounts
carry lock state, and nonce state grows per `(account, nonce_key)`. Transaction payloads may be
larger than standard transactions because of `account_changes`, richer signatures, and phased call
data, which increases L2 data footprint.

These tradeoffs are acceptable if the OP Stack keeps the initial verifier profile conservative and
standardizes enough node policy to avoid pathological validation costs.

### Single Client and Consensus Considerations

New OP Stack feature development, including the next Karst hardfork, is expected to happen on
`op-reth` only. `op-geth` remains supported for security patches and critical bug fixes through May
31, 2026, after which support ends. Because 8130 targets new hardfork functionality, the adoption
plan should assume a single execution client implementation in `op-reth`.

This removes the cross-client divergence risk, but the consensus-critical implementation surface
remains. `op-reth` must implement and test:

- transaction decoding and signature hashing
- intrinsic gas accounting
- authorization and scope checks
- nonce and expiry behavior
- delegation-indicator semantics
- protocol-injected logs and receipt fields
- RPC behavior for new and extended methods

The Account Configuration contract bytecode, initial verifier set, storage layout assumptions,
default account implementation, and reserved addresses also become consensus-critical inputs. These
must be pinned carefully before activation and tested against the final `op-reth` implementation.

There is also an interoperability risk if verifier acceptance policy diverges too much. A
canonical verifier set reduces this risk by giving wallets a guaranteed baseline, but chain-local
extensions can still fragment behavior if applications depend on them. The OP Stack should
therefore treat the canonical set as the in-spec compatibility surface.

## Failure Mode Analysis

The companion FMA lives at
[`security/adopt-eip-8130-on-op-stack-fma.md`](../security/adopt-eip-8130-on-op-stack-fma.md) and
should be reviewed alongside this proposal.

## Impact on Developer Experience

If adopted, 8130 would give OP Stack developers a native transaction model for:

- batching
- sponsorship
- passkey and custom-auth wallets
- efficient high-throughput nonce lanes
- portable owner management across chains

This is a meaningful UX improvement relative to plain EOAs and reduces dependence on bundlers for
many account abstraction use cases.

At the same time, the change introduces new developer surface area. Wallets, SDKs, explorers,
testing frameworks, and local dev environments will all need support for new encoding, new receipt
fields, new system contracts, and verifier discovery or policy awareness. Base will own wallet, SDK,
and indexer compatibility for its rollout, while additional ecosystem integrations should be
encouraged where possible.

The migration path is still attractive because 8130 is additive. Existing ERC-4337 wallets can
continue to operate and can migrate incrementally through the onchain import patterns described in
the EIP instead of requiring users to switch to entirely new addresses.

## Alternatives Considered

### Continue with EOAs Plus ERC-4337 Only

This is the lowest-risk protocol option because it avoids a hardfork-level account abstraction
change. However, it keeps key AA features outside the base transaction model and preserves the need
for separate bundler and mempool infrastructure.

### Design an OP Stack-Specific Account Abstraction Scheme

This could optimize for rollup-specific requirements, but it would fragment the ecosystem and
duplicate standardization work already happening in 8130.

### Wait for 8130 to Mature Further Before Adopting

This reduces the risk of following a moving draft, but it delays native AA support and leaves the
OP Stack dependent on external AA infrastructure for longer.

### Wait for an L1 Account Abstraction Decision

The OP Stack could choose to wait for Ethereum L1 to converge on a protocol-level AA direction
before adopting 8130. This is a different consideration from waiting for 8130 alone to mature. It
would avoid committing to an L2-native path before the broader ecosystem decides which base-layer
AA model it prefers.

This alternative is especially relevant if `EIP-8141` continues to emerge as the front-runner for
L1 AA. Waiting could reduce the risk that the OP Stack adopts a model that later diverges from the
main Ethereum path and would let the stack inherit more of the eventual L1 standardization and
tooling momentum.

The downside is that this delays native AA on the OP Stack for reasons that may be orthogonal to
L2 needs. It also gives up the opportunity for the OP Stack to move earlier on a design that may
still be valuable even if L1 ultimately chooses a different approach.

It is also not clear that an eventual L1-first direction such as `EIP-8141` will match OP Stack
requirements as well as 8130. Relative to 8130, this direction appears to require more complex
ingress rules: nodes may need to simulate and trace transactions to determine whether they satisfy
admission rules such as `EIP-7562`, validation may depend on broader dynamic state, and builders
may take on more wasted compute and packing risk when invalidation is harder to track. Those costs
are especially undesirable when the computation is effectively unbilled.

There are also open questions about whether that direction cleanly supports the feature set the OP
Stack wants. 8130 directly supports 2D nonces, key rotation, expiration, and high-throughput
accounts, while more general models may push important rules into helper contracts or off-protocol
conventions. That can increase divergence across chains and reduce the confidence wallets have in a
standard cross-chain integration path.

### Adopt Only a Subset of 8130

For example, the OP Stack could adopt verifier and account-configuration concepts without adopting
the transaction type. This would shrink the initial scope, but it would also leave much of the UX
benefit on the table and complicate the overall design.

## Open Decisions Before Approval

The following decisions should be resolved, or explicitly accepted as TBD, before this proposal is
ready for approval:

- fork name and activation schedule
- fixed addresses for the Account Configuration contract, precompiles, verifier set, and default
  account implementation
- exact canonical verifier bytecode and deployment process
- whether the first OP Stack deployment supports nonceless transactions or defers `NONCE_KEY_MAX`
  while supporting standard and user-defined nonce keys
- default account implementation bytecode and wallet UX requirements for auto-delegation
- agreed mempool policy limits for validation cost, owner changes, payer authorization, and
  transaction shape
- final RPC extension shapes, including accepted-verifier discovery, 2D nonce queries, and receipt
  extensions
- final `op-reth` implementation scope and required consensus test coverage
- audit owners, audit scope, and signoff criteria for each audit cycle

## Risks & Uncertainties

- 8130 is still draft and may continue to change upstream.
- The final transaction type, reserved addresses, default wallet bytecode, and canonical verifier
  set bytecode and addresses are not yet settled.
- If chains or applications come to depend on verifiers outside the canonical verifier set, wallet
  interoperability may still fragment.
- Delegation semantics must be audited carefully anywhere the OP Stack currently distinguishes EOAs
  from contracts.
- The change spans consensus logic, node policy, RPC, tooling, and wallet infrastructure, so
  rollout coordination is non-trivial.
- Initial deployments may need tighter verifier and nonceless-mode limits than the upstream EIP
  theoretically permits.
