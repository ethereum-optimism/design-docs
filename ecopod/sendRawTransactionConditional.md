# Purpose

To fully unlock ERC-4337 on the OP-Stack.

# Summary

By providing an auxilliary transaction submission mechanism, [eth_sendRawTransactionConditional](https://notes.ethereum.org/@yoav/SkaX2lS9j), we can enable Bundlers to submit Entrypoint transactions with stronger guarantees, avoiding costly inadvertent reverts.

We can implement this endpoint in op-geth with an additional layer of external validation to ensure this endpoint is safe from DoS attacks, does not supersede `eth_sendRawTransaction`, and can be deprecated if native account abstraction efforts in the future absolves the need for this endpoint.

**Note:** As a result of [EIP-3074](https://eips.ethereum.org/EIPS/eip-3074), the interop project is also exploring external validation as a mechanism to preventing DoS attack. A low-stakes scoped version of this for `eth_sendRawTransactionConditional` is a good opportunity for learning.


# Problem Statement + Context

Bundlers aggregate UserOps from an external mempool into a single transaction sent to the enshrined 4337 `Entrypoint` contract. It is the Bundler's responsibility to ensure every UserOp in the transaction will not revert when its validation step is executed. In between tx submission and inclusion, there is a chance network state changes causes the transaction to revert, even after local bundler validation.

With private Bundler mempools, the likelihood of these reverts are small. However shared 4337 mempools are coming into [production](https://medium.com/etherspot/decentralized-future-erc-4337-shared-mempool-launches-on-ethereum-b6c860072f41), launched on Ethereum and some L2 testnets, decentralizing 4337 infrastructure. This is possible with no change on Ethereum through special block builders like Flashbots which support atomic transaction inclusion. An endpoint like `eth_sendRawTransactionConditional` is required to launch a shared mempools on L2 as there's only a single block builder, the sequencer, and the likelihood of reverts with a shared mempool would be too high of a cost for bundlers to operate.


# Alternatives Considered

## Continue with private 4337 mempools

4337 account abstraction is live on optimism. Dapps likely utilize the bundler endpoints that are come with a vendor-specific SDK -- Alchemy, Pimilico, Thirdweb, etc. As 4337 infrastructure becomes more permissionless, we will likely later have to play catch-up to ensure the OP-Stack remains compatible while other L2 offerings have already moved towards supporting `eth_sendRawTransactionConditional`.

## Verticalize the OP-Stack

Rather than externalize the 4337 mempool, the op-stack could natively offer a UserOp mempool alongside the regular mempool, avoiding conflicts as the sequencer can ensure the bundled UserOps do not conflict with the latest chain state when sealing a new block. However, this adds additional complexity to the stack, where the proposer-builder seperation in 4337 nicely keeps account abstraction out of protocol. A chain operator can still choose to verticalize in proposed solution solution by allowlisting only its own bundler through `eth_sendRawTransactionConditional`, achieving the same outcome.


# Proposed Solution

1. Implement `eth_sendRawTransactionConditional` as described in the [spec](https://notes.ethereum.org/@yoav/SkaX2lS9j). A draft implementation for this [exists](https://github.com/ethereum/go-ethereum/compare/master...tynes:go-ethereum:eip4337)today. The conditional is applied on submission and tx inclusion.
2. Introduce additional validation.
    * **Exclusive 4337 Entrypoint Support**: We assert `tx.to() == entrypoint_contract_address` to ensure that this endpoint isn't used for other usecases. Making it easy to rollback if deprecated in the future due to native AA or a better solution to this problem.
    * **Runtime Shutoff**: The endpoint can be shutoff by the sequencer to address issues. As long a single bundler supports a fallback to `eth_sendRawTransaction`, 4337 liveness should remain OK.
    * **Global Rate Limit**: Rate limit how many conditional txs can be buffered at any given time in the mempool.
    * **Authentication**: Requests must be authenticated, flashbot-style, with a self-identified keypair. This enables the development of two authorization policy modules.
        1. _Allowlist_: The sequencer can verticalize their chain with a bundler sidecar or have specific partners allowed to send conditional entrypoint txs.
        2. _Local Rate Limiting_: The authenticated caller has flashbots-style [reputation](https://docs.flashbots.net/flashbots-auction/advanced/reputation) computed based on the success rate of conditional txs, enabling permisionless bundler participation

The endpoint MUST be authenticated but the authorization policies may not be needed to permissionlessly launch this feature if a global rate limit with a runtime shut off is sufficient. A recommendation is to have the allowlist policy implemented with the keys of known bundlers registered, such that it is ready to enable. As this endpoint is used in production, additional policies like the local rate limits can be iterated on and implemented on a as-needed basis.

The restricted usage & rate limits prevents this endpoint from superseding `eth_sendRawTransaction`, only suitable for honest bundlers, and also easy to wind back if needed later on.

# Risks & Uncertainties

**Risk 1: Private 4337 mempools remain the status quo.** Bundlers can still suffer from reverts and have a strong desire for an endpoint like `eth_sendRawTransactionConditional`. In our proposed solution design, we can allowlist these common bundler operators and hold them to an SLA. Our proposed solution also leaves the door open to verticalizing bundler services in the superchain through the allowlist, enhance the account abstraction exprience in the superchain through a superchain-specific 4337 mempool.

**Risk 2: Validation listed in the proposed solution isn't enough for permissionless bundler participation.** The listed validation rules are just a start on how we might scale this approach. With the existing solution we have the ability switch off the endpoint or allowlist known actors while exploring adding more DoS prevention tactics like IP-Banning or strict Local Rate Limiting policies.

**Risk 3: Generalized External Validation.** Validation policies should be DRY'd between interop, sendRawTransactionConditional, and any future use cases. These policies that are implementated should work well between the different usecases as this approach is adopted and scales. For example, should accrued reputation be shared across use cases?
