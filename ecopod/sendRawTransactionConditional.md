# Purpose

To fully unlock ERC-4337 on the OP-Stack.

# Summary

By providing an auxilliary transaction submission mechanism, [eth_sendRawTransactionConditional](https://notes.ethereum.org/@yoav/SkaX2lS9j), we can enable Bundlers to submit [Entrypoint](https://eips.ethereum.org/EIPS/eip-4337#entrypoint-definition) transactions with stronger guarantees, avoiding costly inadvertent reverts. The endpoint is an extension to `eth_sendRawTransaction` with the added ability to attach a conditional. The conditional is set of options that specify requirements for inclusion, otherwise rejected.

We can implement this endpoint in op-geth with an additional layer of external validation to ensure this endpoint is safe from DoS attacks, does not supersede `eth_sendRawTransaction`, and can be deprecated if native account abstraction efforts in the future absolves the need for this endpoint.

**Note:** As a result of [EIP-3074](https://eips.ethereum.org/EIPS/eip-3074), the interop project is also exploring external validation as a mechanism to preventing DoS attack. A low-stakes scoped version of this for `eth_sendRawTransactionConditional` is a good starting point and opportunity for learning.


# Problem Statement + Context

Bundlers aggregate [UserOp](https://eips.ethereum.org/EIPS/eip-4337#useroperation)s from an external mempool into a single transaction sent to the enshrined 4337 [Entrypoint](https://eips.ethereum.org/EIPS/eip-4337#entrypoint-definition) contract. It is the Bundler responsibility to ensure every UserOp can succesfully execute its validation step, otherwise resulting in a reverted transaction. With private Bundler mempools, the likelihood of these reverts are small as Bundlers are operating over different sets UserOps. Unless a smart account frontruns a submitted UserOp to the Entrypoint, a Bundler doesn't have to worry about changes in network state such as an incremented smart account nonce invalidating the transaction during the building process.

However shared 4337 mempools are coming into [production](https://medium.com/etherspot/decentralized-future-erc-4337-shared-mempool-launches-on-ethereum-b6c860072f41), launched on Ethereum and some L2 testnets, decentralizing 4337 infrastructure. Multiple Bundlers operating on this mempool can create transactions including the same UserOp, increasing the likelihood of reverts due to changed account states. These reverts are too high of a cost for bundlers to operate.

This problem is worked around on L1 through special block builders like Flashbots that provide atomic inclusion when building blocks through a custom rpc like [eth_sendBundle](https://docs.flashbots.net/flashbots-auction/advanced/rpc-endpoint#eth_sendbundle), avoiding on-chain reverts for the caller. However on L2 there's only a single block builder, the sequencer, creating the requirement for the `eth_sendRawTransactionConditional` endpoint providing similar inclusion guarantees to launch shared 4337 mempools.


# Alternatives Considered

## Do Nothing

4337 account abstraction is currently live on optimism. Dapps utilize the bundler endpoints that are come with a vendor-specific SDK -- Alchemy, Pimilico, Thirdweb, etc. As 4337 infrastructure becomes more permissionless, we will later have to play catch-up to ensure the op-Stack remains compatible while other L2 offerings have already moved towards supporting `eth_sendRawTransactionConditional`.

## Verticalize the OP-Stack

Rather than externalize the 4337 mempool, the op-stack could natively offer a UserOp mempool alongside the regular tx mempool. When creating a new block, the sequencer can pull from the two, ensuring the bundled UserOps do not conflict with the latest network state. However, this adds additional complexity to the stack, where the proposer-builder seperation in 4337 nicely keeps account abstraction out of protocol.

Verticalization is also possible in the proposed solution by configuring the allowlist of the authenticated `eth_sendRawTransactionConditional` endpoint to either a self-managed bundler or that of a partner, achieving the same outcome as native mempool without the complexity of a deeper change in the op-stack.


# Proposed Solution

1. Implement `eth_sendRawTransactionConditional` as described in the [spec](https://notes.ethereum.org/@yoav/SkaX2lS9j), for which a draft implementation [exists](https://github.com/ethereum/go-ethereum/compare/master...tynes:go-ethereum:eip4337). The conditional attached to the transaction is checked against the latest state prior to mempool submisison and re-checked when a block including the transaction is being built.

2. Implement external validation rules. These rules are not enshrined in the node software's implementation of `eth_sendRawTransactionConditional`, instead scoped a layer higher, i.e a proxy

    * **Only 4337 Entrypoint Contract Support**: `tx.to() == entrypoint_contract_address`

        The rationale is to make it easier to rollback if dprecated in the future due to native account abstraction or better solutions. Otherwise new uses case might create unwanted dependencies on this endpoint.

        There does exist different [versions](https://github.com/eth-infinitism/account-abstraction/releases) of the Entrypoint contract as the 4337 spec is iterated on. We'll need to stay up to date with these version as a part of op-stack [preinstalls](https://docs.optimism.io/builders/chain-operators/features/preinstalls) and pass through calls to all the supported versions of `EntryPoint`.

    * **Authentication**: Allowlist Policy

        Requests to this endpoint MUST be authenticated with a secp256K1 keypair, similar to [flashbots authentication](https://docs.flashbots.net/flashbots-auction/advanced/rpc-endpoint#authentication). The [EIP-191](https://eips.ethereum.org/EIPS/eip-191) hash of the json-rpc payload must be signed and included in the the `X-Optimism-Signature` header of the request in a `<public key adddress>:<signature>` format.

        With the public key of the caller, we can implement an allowlist policy module, allowing the chain operator to verticalize by running their own bundler or delegating to partners.

    * **Runtime Shutoff**

        Through a new admin endpoint exposed through op-node, the sequencer can reject all calls to this endpoint, similar to the `admin_(start|stop)Sequencer` endpoints. If a proxy implementing these validation rules are setup to run on seperate hosts from the sequencer, this runtime switch must be readable such that these requests can be terminated without reaching the sequencer.

        As long as a single bundler supports a fallback to `eth_sendRawTransaction`, 4337 liveness should remain OK

    * **Global Rate Limit**

        Rate limit how many conditional txs can be sent over a configured time period. Like with the runtime shutoff, if these validation rules are horizontally scaled externally to the sequencer through a proxy, confiugration must be readable by these hosts & the design space for this rate limit should be explored -- i.e globally synchronized rate limit vs per-host limits.


With this initial set of validation rules, we should be in a good position to safely launch this endpoint with either permissioned or permissionless bundler participation. If permissionless, the public keys of known bundlers should be collected and registered, such that the allowlist can be enabled to avoid potential 4337 downtime while implementing more validation mechanisms. In the worst case, this endpoint can be completely shutoff

If high rates of rejected transactions are observed, additional validation rules can be iterated on.

* **Local Rate Limiting**

    Adopt flashbots-style [reputation](https://docs.flashbots.net/flashbots-auction/advanced/reputation) score that's computed based on the success rate of conditional txs. Designed to filter out inefficient callers and promote honest ones.


# Risks & Uncertainties

**Risk 1: Private 4337 mempools remain the status quo.** Bundlers can still suffer from reverts and have a strong desire for an endpoint like `eth_sendRawTransactionConditional`. In our proposed solution design, we can allowlist these common bundler operators and hold them to an SLA. Our proposed solution also leaves the door open to verticalizing bundler services in the superchain through the allowlist, enhance the account abstraction exprience in the superchain through a superchain-specific 4337 mempool.

**Risk 2: Validation listed in the proposed solution isn't enough for permissionless bundler participation.** The listed validation rules are just a start on how we might scale this approach. With the existing solution we have the ability switch off the endpoint or allowlist known actors while exploring adding more DoS prevention tactics like IP-Banning or strict Local Rate Limiting policies.

**Risk 3: Generalized External Validation.** Validation policies should be DRY'd between interop, sendRawTransactionConditional, and any future use cases. These policies that are implementated should work well between the different usecases as this approach is adopted and scales. For example, should accrued reputation be shared across use cases?
