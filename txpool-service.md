# Purpose

The purpose of this document is to illustrate a way to be safe from denial of service attacks in the context of interop
in the mempool. This document is not meant to illustrate a way to be safe from denial of service attacks in all cases
but this should cover the majority.

This document also can apply to [sendRawTransactionConditional](https://github.com/ethereum-optimism/design-docs/pull/2)
or [Flashbots protect](https://protect.flashbots.net/about) revert protection types of designs.

# Summary

A new horizontally scalable service called `op-txpool` is introduced that enables denial of service protection
in the context of interop.

# Problem Statement + Context

Internet services can be susceptible to spam or denial of service attacks when they are permissionless
to participate in. This is not a new problem, SMTP (Simple Mail Transfer Protocol) has been fighting against
spam for a long time to the point where it is [very difficult](https://news.ycombinator.com/item?id=9798800)
to run your own SMTP server at home.

The general rule of thumb is that if there is a marginal cost to participate at a slightly larger than marginal
reward, then there will be spam on a permissionless network. In the context of blockchains, the entire network
processes code on the behalf of pseudonymous entities that can only be identified by an ECDSA signature.
Anytime there is a mispricing of resources, it is [attacked](https://ethos.dev/shanghai-attacks). To ensure
that denial of service attacks are not possible, the user requesting network resources should be forced to
pay for them.

Layer one Ethereum has always had more strict invariants about the validity of transactions in the mempool.
There has been much debate about [3074](https://eips.ethereum.org/EIPS/eip-3074) because
it allows a transaction entering the mempool to invalidate a transaction that has already
been allowed into the mempool. This opens up the door for an attacker to flood
the mempool with transactions and then send a single transaction that causes many to be invalidated. This
would enable the attacker to only pay for a single transaction when the network was forced to process
many transactions.

The idea that execution would be required for transaction validity was so insane that we didn't want to
consider it for interop and instead designed the system around an "only EOA" invariant that enabled
static analysis because all of the data required to verify executing messages was present in the
RLP encoded transaction itself. Post 3074 being considered CFI, it breaks the ability to enforce
an "only EOA" invariant, so we dedided to lean deeply into the problem and turn it into a feature.
The "only EOA" invariant was the most complained apart part of the interop design and it only existed
because we were scared of denial of service.

Enter op/acc - everything is an engineering problem waiting to be solved. When there is a denial of
service attack, figure out how to horizontally scale a solution. This can be achieved by
creating a stateless edge proxy service that runs a forking EVM. We call this the `op-txpool`.

# Alternatives Considered

## Do No Additional Validation in Mempool

It could be possible to get further than we can think by shipping with no extra validation at all in the mempool.
This would mean that the extra interop related rules would be validated at block building time only. This can
create a headache for the block builder and it would take an active adversary to cause problems over time
as we could block malicious IP addresses or accounts sending transactions that induce denial of service
at a higher layer.

## Lobby Against 3074

This isn't very prosocial and its likely that something else breaks the "top level call" invariant detection
in the future anyways.

## Introduce a new EIP to enable our usecase

It is possible to create a new EIP that enables our usecase but it risks going through L1 governance which would
take a long time and be risky as it could end up being rejected after we invest a lot of time into it.

# Proposed Solution

## `op-txpool`

**This is scoped to the release of interop**

The `op-txpool` service is stateless, can horizontally scale and simply needs to be configured with
RPC URLs of remote nodes to fetch state from. It runs an RPC server and implements a single method
in the [txpool](https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-txpool) namespace:

```
txpool_validateTransaction(string) -> bool
```

This RPC method accepts a hex RLP encoded Ethereum transaction and returns whether or not the
transaction should be considered valid.

The mempool today has both stateless and stateful validation that it performs on every transaction
as it is entering the mempool.

Stateless:
- [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) typing
- [EIP-3860: Limit and meter initcode](https://eips.ethereum.org/EIPS/eip-3860)

Stateful:
- Nonce increments based on state or other txs in mempool
- Balance can pay fee
- Gas limit not too small and not too large
- GasFeeCap larger than or equal to basefee
- 4844 sidecar matches blob hashes

Interop adds a new requirement such that execution is required to know if a transaction entering
the mempool can be considered valid. This is where `op-txpool` can be very useful - it can execute
transactions to look for executing messages and then resolve the initiating message from a remote chain.
This architecture removes the need for the EL client to be aware of the remote chain RPCs.
It can run optionally, meaning that full nodes don't need to run it. The first iteration of `op-txpool`
doesn't need to do any of the checking that is done in the mempool already but in the future it can
also act as a first level of checking before the transactions reach full nodes.

## Implementation

`op-txpool` would track the head of the local chain. It would use a foundry style forking EVM where the
state db is backed by a remote node. This means that stateful opcodes would be backed by RPC calls to
the remote node instead of using a local database. A cache should be used, to reduce the number of requests
that need to be made.

The transaction would be executed in a forking EVM which targets "latest" (but uses a static hash/number for consistency)
and any executing messages would be identified and resolved. The interop spec defines the validation.

Inside of `op-geth`, transactions would be forwarded to `op-txpool` if and only if it is configured.
Depending on bechmarking, its possible that this can be safely done once instead of on every new block,
as the initial release scopes to just doing the interop related checks. `op-geth` will still be responsible
for all of the checks it does today.

See [op-geth](https://github.com/ethereum-optimism/op-geth/blob/3653ceb09fc2025d617c6b1033ab8d61a818f67c/core/txpool/validation.go#L76)
and [reth](https://github.com/paradigmxyz/reth/blob/081796b138fd00a6d0af2ad463123fdbe04765ab/crates/transaction-pool/src/validate/eth.rs#L151)
for their implementations of mempool validation.

The forking EVM can be done by implementing a custom geth remote `StateDB` or using `revm` with `EthersDB`.

# Future Directions

The `txpool_validateTransaction` endpoint can be extended to include revert protection, similar
to [Flashbots protect](https://protect.flashbots.net/about).

This proxy service can implement additional RPC endpoints such as:

- `eth_sendRawTransaction`
- `eth_sendRawTransactionConditional`
- `txpool_validateTransaction(string, conditionalOpts)`

## Optimizations

The overhead of RPC over HTTP 1 is very unoptimized. A simple migration to gRPC could 
work as an optimization. There are likely many other optimizations that would be required
as this service scales.

# Risks & Uncertainties

If every transaction in the mempool needs to be revalidated every 2 seconds, it is a lot of churn. There is a tradeoff
with the diff to `op-geth` growing by being smarter with tracking which transactions may need to be validated again.
It would be more simple to add these sorts of optimizations in `op-txpool`, ie track which transactions are not cross chain
messages and just don't reexecute them, assuming that only a very small percentage will execute in a different way at
block building time that makes them produce an executing message.

It would be much better if an optimized db was used to serve the forking EVM. Some DB queries in geth are not very optimized.
