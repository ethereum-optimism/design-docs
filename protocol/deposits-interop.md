# Purpose

[executing-message]: https://github.com/ethereum-optimism/specs/blob/4f9b6abad85a4d8cfa6a7eb653480841cc535bb0/specs/interop/messaging.md#executing-messages

This design doc is meant to illustrate designs for enabling censorship resistant cross chain messaging in the context of interop.
What this means in practice is that a deposit transaction can be used to create an [executing message][executing-message].

# Summary

Some deep protocol modifications need to be made, particularly in the derivation pipeline. A new way of mapping deposit
events into deposit transactions is introduced via a new invariant as well as way to check for executing message validity
as part of the derivation pipeline.

# Problem Statement + Context

Since the interop spec defines that a block that contains an invalid executing message is invalid and will be reorganized
out of the chain, this adds risk to any block builder that includes cross chain transactions. An invalid executing message
means that its `Identifier` doesn't corrently reference the `types.Log` that is included in the calldata. See the
[spec](https://github.com/ethereum-optimism/specs/blob/4f9b6abad85a4d8cfa6a7eb653480841cc535bb0/specs/interop/messaging.md#message) for
additional details if necessary.

The best practice for the block builder is to drop transactions that create invalid executing messages. This is not possible
in the case of deposit transactions since they are force include. Right now, the spec defines a mapping from an event to
an OP Stack specific [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) transaction type called a `DepositTx`. This mapping
can be described as "statelessly [injective](https://en.wikipedia.org/wiki/Injective_function)", meaning that there is a deterministic
way to map the event to the deposit transaction without needing to reference any external state.

The injective function is defined by:

$$f:A→B$$

The event emitted from the `OptimismPortal` is defined as:

```solidity
event TransactionDeposited(address indexed from, address indexed to, uint256 indexed version, bytes opaqueData);
```

The `DepositTx` is defined as:

```go
type DepositTx struct {
	// SourceHash uniquely identifies the source of the deposit
	SourceHash common.Hash
	// From is exposed through the types.Signer, not through TxData
	From common.Address
	// nil means contract creation
	To *common.Address `rlp:"nil"`
	// Mint is minted on L2, locked on L1, nil if no minting.
	Mint *big.Int `rlp:"nil"`
	// Value is transferred from L2 balance, executed after Mint (if any)
	Value *big.Int
	// gas limit
	Gas uint64
	// Field indicating if this transaction is exempt from the L2 gas limit.
	IsSystemTransaction bool
	// Normal Tx data
	Data []byte
}
```

In earlier iterations of the interop design, there was an "only EOA invariant" that was used to enable static
analysis of transactions. By only allowing for EOAs to send cross chain transactions, this meant that the
calldata for the `CrossL2Inbox` would be present in the transaction's data itself. This could be statically
analyzed by both the protocol (fork choice rule) and block builders. Forcing EOAs to send the executing
messages was the biggest complaint by people as it greatly reduced the design space, prevented compatibility
with smart contract wallets and prevented compatibility with [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) bundlers.

With the removal of the "only EOA invariant", execution is now a requirement to determine the validity of a transaction.
This applies for deposit transactions as well. The derivation pipeline needs some sort of legibility into the validity
of these deposit transactions.

To prevent race conditions on deposited executing messages that reference recent initiating messages, a delay should
be enforced on deposits that contain executing messages. This prevents attacks where some amount of p2p latency
can cause nodes to inconsistent chains.

# Proposed Solution

The stateless injective function for mapping deposit events into deposit transactions is updated to be a
2-valued function, meaning that any input can be mapped to one of two inputs. In mathematical terms:

$$f:A→\{b1,b2\}$$

A deposit transaction event can be mapped to either a valid deposit transaction or a noop deposit transaction.

A new invariant called the "sequencing window invariant" is introduced that enforces a delay for deposit
transactions that trigger executing messages.

## Derivation Pipeline Execution

The derivation pipeline is updated to include EVM execution of the deposit transactions. A forking EVM provider backed
by L2 tip state is used. This enables simulation of the deposit transactions before they are included in L2. If the deposit
transaction fails the sequencing window invariant or creates an invalid executing message, it is mapped into a noop deposit
transaction.

A noop deposit transaction:
- keeps its sourcehash for tx hash uniqueness
- keeps its from, so the nonce can increment
- keeps its to, in case its also depositing `ether`
- keeps its mint, in case its depositing `ether`
- gas is set to 0
- data is set to empty

This should hold the invariant that the `ether` is always minted when sent into the `OptimismPortal` but prevents
any additional execution on top of that.

In pseudocode:

```python
txs = []
evm = prepare_evm(eth)
for i, tx in enumerate(payload.transactions):
  if tx.type() != DEPOSIT_TYPE:
    tx.append(tx)
    continue
  logs = evm.call(tx.to_message())
  for log in logs:
    if is_executing_message(log):
      if sequencing_window_invariant(log) == False or is_valid_message(log) == False:
        tx.gas = 0
        tx.data = ""
        txs.append(tx)
        break
```

If static analysis was possible, then no execution would be required to derive the information required to check the
validity of the executing messages. The same two checks `sequencing_window_invariant` and `is_valid_message` could
be applied but solely with static analysis instead of needing to execute to get the executing message.
The definition of `sequencing_window_invariant` is defined below and `is_valid_message` is defined in the specs.

Note that only the deposits are simulated as part of the derivation pipeline, not any transactions that are sent
directly to the sequencer. This means that there is a guaranteed worst case in the amount of gas for this execution.

## Sequencing Window Invariant

With or without the only EOA invariant, the sequencing window invariant needs to be defined to prevent
race conditions with deposit transactions and the sequencer having a slow p2p network. Without this invariant,
it would be possible to trigger reorgs by sending a deposit transaction that references an initiating message
that is at the tip of a remote chain. If the sequencer doesn't immediately always have the remote chain, it
could result in reorgs.

The definition of the sequencing window invariant is as follows:

```
require(identifier.timestamp + sequencer_window <= block.timestamp)
```

Where the `block.timestamp` is the timestamp of the pending block, ie the next block that is being built.

This states that a sequencing window must bass on top of an initiating message before it can be considered
valid via a deposit. This removes the concept of p2p based preconfirmations from consensus. If the sequencing
window has elapsed on top of an initiating message, it can be safely considered as final.

# Alternatives Considered

The design spaces for solutions involve one of the following patterns:
- guarantee that there is no way to create an invalid executing message with a deposit
- update the mapping from events to deposit transactions so that it can result in noop deposit transactions

## Banning Deposits Creating Executing Messages

With the "only EOA invariant", this could be handled at the level of the `OptimismPortal`. The idea
is that the protocol itself could ban deposits that call the `CrossL2inbox` via static analysis.
This design is not possible with the removal of the "only EOA invariant". An arbitrary subcall
can call out to the `CrossL2Inbox`.

With the "only EOA invariant", this could also be handled in the derivation pipeline. The `DepositTx`
could be mapped into a null value if its `tx.to` as equal to the `address(CrossL2Inbox)`.

## State Transition Diff

It would be possible to not require that there is execution in the derivation pipeline but this would
result in the requirement for the EL to malleate `ExecutionPayload`s. The execution payload contains
the raw RLP hex serialized transations for a block. Right now there is an invariant such that the
`ExecutionPayload` is just accepted as is and is executed. It includes the deposit transactions
and the `op-node` is responsible for building the `ExecutionPayload` correctly. It is possible to
have the EL modify (malleate) the `ExecutionPayload`'s deposit transactions before considering
them final. This would essentially be adding more rules to the state transition function.

This would look like a diff to the state transition function that applies the sequencing
window invariant and does the executing message validity checking during the state transition.
This creates a leaky abstraction as now the state transition function needs to know about
the dependency set, doing remote RPC calls during execution. This does not seem like
an ideal solution for that reason.

# Risks & Uncertainties

- This is a fairly big diff to the derivation pipeline, it needs buy in from many teams
- Without building consensus on solutions for these problems, it could delay the launch of interop
- Need to derisk ability to safely have a L2 RPC client as part of the derivation pipeline. This could add a lot of extra work for the fault proof program.
