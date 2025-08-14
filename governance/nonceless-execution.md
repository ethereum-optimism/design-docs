# Nonceless Execution

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Alberto Cuesta Cañada                              |
| Created at         | 2025-08-12                                         |
| Initial Reviewers  | [ ] John Mardlin                                   |
|                    | [ ] Kelvin Fichter                                 |
|                    | [ ] Matt Solomon                                   |
| Need Approval From | _Reviewer Name_                                    |
| Status             | Draft                                              |

## Problem Statement + Context

Safe accounts maintain a single, monotonically increasing `nonce` that is included in the EIP-712 typed data each owner signs and is checked during `execTransaction`. The contract only accepts execution for the current nonce; upon success it increments the nonce by 1. This enforces a single global queue: at any time exactly one nonce is executable. Owners may propose alternate transactions for that same nonce as replacements, but nothing with a future nonce can be executed until the current one is finalized. If execution reverts, the nonce is unchanged and the queue remains blocked. This ordering provides replay protection and simple reasoning, but it also couples otherwise independent actions and makes execution latency of one item the bottleneck for all others.

Operationally, this single-queue design turns independent governance actions into a serialized pipeline: urgent remediations can be blocked behind planned items, execution failures stall the queue, and coordination and development costs grow as we scale.

## Customer Description

The customers for this design doc are any participants in the multisig accounts that would utilize this module, as well as any 3rd parties who rely on the security properties of these accounts.

### Customer Reviewers
- Coinbase (potentially impacted 3rd party)
- Uniswap (potentially impacted 3rd party)
- Security Council
- Optimism Foundation

## Requirements and Constraints
- Enable unordered execution securely.
- Transaction executions are public.
- Avoid replay of executed transactions.
- Keep code as minimal as possible to avoid complex logic in Safe modules.
- Reduce mental overhead for Safe owners and multisig leads.

## Proposed Solution

### Singleton
The `NoncelessExecutionModule` (alternatively `UnorderedExecutionModule`, the name is important to get right for clarity) is a singleton that exists at a known address on all chains and has no constructor parameters. Any Safe on any network can choose to enable the module within the Safe. No configuration is required.

### Nonceless Module Execution
Normal owner-signed executions use `execTransaction(...)`, which include the Safe’s nonce in the EIP-712 digest owners sign. This function requires the current nonce and increments it on success, with the purpose of providing replay protection and enforcing a single global ordering.

Module execution uses `execTransactionFromModule(...)` and doesn't use nonces. We use this function to an alternative execution pathway in the module with a custom replay protection mechanism. The module leverages Safe's existing signature verification through `checkSignatures()`. Signers sign the same transaction hash they would for a normal Safe transaction, which includes the Safe's domain separator. The key difference is that instead of using the Safe's sequential nonce, the module uses the calculated hash-once value as the nonce parameter in `getTransactionHash()`. This means signers can use any existing Safe signing tools and workflows - they're signing a standard Safe transaction, just with a different nonce derivation.

For replay protection, the module uses a hash-once mechanism instead of sequential nonces (i.e. number-once):

1. **Hash Calculation**: `hashOnce = keccak256(abi.encode(hashOnceInputs))` where `hashOnceInputs` contains the `prevHashOnce` and `mixHash` fields
2. **Transaction Hash**: This `hashOnce` value is cast to `uint256` and used as the nonce parameter in Safe's `getTransactionHash()` function to generate the final transaction hash that signers sign
3. **Storage**: The resulting transaction hash is stored in `executedTransactions` mapping to prevent replay

Any hash can be executed at any time as long as it has the required signatures and hasn't been used before.

### Optional transaction ordering

The module supports optional transaction dependencies through a prerequisite mechanism. When creating a transaction, signers can specify that it should only execute after a specific previous transaction has completed.

This is implemented using the `HashOnceInputs` struct with the following fields:
- `prevHashOnce`: Set to zero for independent transactions, or to the transaction hash of a required predecessor
- `mixHash`: Additional entropy to ensure hash uniqueness

When `prevHashOnce` is non-zero, the module validates that the referenced transaction has already been executed before proceeding. This enables controlled sequencing when needed (e.g., multi-step upgrades) while preserving unordered execution for independent operations.

## Non-goals

This change does not involve changing thresholds or governance ownership models, adding new signing tools or UIs, or redesigning incident response.

## Resource Usage

On-chain costs are similar to current multisig execution with small module overhead. Operationally, this reduces coordination overhead and enables faster handling of urgent items.

## Alternatives Considered

The status quo of nonce ordering requires no implementation effort but negatively impacts operations and UX due to the coordination and planning required.

## Risks & Uncertainties

- The computed transaction hashes need to not collide with an already used nonce. Using keccak256 hashes should be enough to avoid that.
- Although this has very little impact on signing flows, it does significantly change transaction execution (and approval with nested safes). Hard to say how heavily it would impact on the superchain-ops repo.
- Is this something that can be adopted piecemeal, ie. could we manage with safes that do and do not use this module? I think we could likely autodetect when a Safe supports nonceless execution, but more branches is still harder to maintain.
- Unexpected effects due to unordered changes.

## References

- [Unordered Execution of Safe Transactions](https://www.notion.so/oplabs/Unordered-execution-of-Safe-transactions-20ef153ee1628054ade1e7e8beeadfef#20ef153ee162800689d1cd1268c85f91)
- Nonceless execution module (Safe-compatible): [UnorderedExecutionModule.sol](https://github.com/ethereum-optimism/optimism/blob/safer-safes/nonceless/packages/contracts-bedrock/src/safe/UnorderedExecutionModule.sol)
