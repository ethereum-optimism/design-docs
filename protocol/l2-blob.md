# Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->
Currently, EIP-4844 BLOB transactions are disabled on the OP Stack, leading to two issues:

 - There is no enshrined L2 BLOB DA solution for L3s, forcing them to integrate third-party DA providers and manage the security risks associated with DA bridges if they don’t want to rely on the same DA as their L2. This is especially problematic if the L2 uses L1 DA, which has higher costs than Alt-DA, adding significant development complexity for L3 teams.
 - It prevents many applications based on L1 BLOBs from migrating to L2 to take advantage of lower execution costs.

The purpose of this design document is to propose supporting L2 BLOBs based on Alt-DA, allowing L3s to have an enshrined L2 BLOB DA to rely on, while also enabling applications based on L1 BLOBs to migrate to L2 seamlessly.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

This document proposes support for BLOB transactions, enabling a **hybrid DA model** in the OP Stack:
 - L2 calldata uses L1 DA.
 - L2 BLOBs use Alt-DA.
 
This enables users and applications on an L2 to choose between L1 DA and Alt-DA for different types of transaction data within the same network, without the need to maintain multiple L2s:
- High-value data (e.g., financial data) goes to L2 calldata & L1 DA;
- Low-value data (e.g., GameFi) goes to L2 BLOB & Alt-DA.

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

The Ethereum Cancun upgrade has significantly reduced Layer 2 (L2) data uploading costs by introducing BLOB
transactions to Layer 1 (L1). This innovation has also enabled a variety of additional applications based on
the BLOBs due to their low cost, such as [blob.fm](https://blob.fm/), [EthStorage](https://ethstorage.io), and
[Ethscriptions](https://ethscriptions.com/). However, while the data upload costs have decreased, the execution
costs on L1 remain high compared to L2, leading to high costs for L2 state proposals and non-financial applications
that rely on BLOBs.

The goal of this specification is to **support L2 BLOB transactions in the OP Stack**. This would allow L3 solutions,
which settle on L2, to have an enshrined 4844-compatible DA layer that they can use directly, without needing to
integrate third-party DA providers or deal with the security risks associated with DA bridges. Additionally, the
applications mentioned above could migrate to L2 with minimal costs.

Furthermore, this spec proposes using
[Alt-DA](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/alt-da.md) to upload L2 BLOBs while
still using L1 DA for L2 calldata. This approach, referred to as a “hybrid DA L2”, combines the best features of
different DA solutions. This allows
users and applications of an L2 to choose between L1 DA and alt-DA for different types of transaction data within the
same network, without the need to maintain multiple L2s. Specifically, users can upload and store non-financial data
at a very low cost using L2 BLOBs and Alt-DA, while still conducting critical financial data using L2 calldata and
L1 DA. In some cases, these two types of data may even occur within the same transaction. Here are a few potential
scenarios:

- While the Optimism mainnet continues to use L1 DA for uploading calldata, multiple app-specific or game-focused
L3s settled on it can directly use the enshrined 4844-compatible L2 DA layer, benefiting from easy integration, robust
security, and lower costs.
- Users might use a platform like Decentralized Twitter primarily for social networking (non-financial), while
also sending payments (financial) to other users within the same application.

The following diagram illustrates the transaction data flow for a hybrid DA L2:

```mermaid
flowchart
    A[Users] -->|Non-financial Tx Using BLOB| B(L2)
    A[Users] -->|Financial Tx Using Calldata| B(L2)
    B -->|L2 BLOB| C(Alt-DA)
    B -->|L2 Calldata| D(L1 DA)
```

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

## Enabling BLOB Transactions in L2 EL

The interface and implementation of L2 EL should remain consistent with L1 EL to ensure seamless migration of
applications. Note that while BLOBs are gossiped within the L1 P2P network, for enshrined BLOB DA support in L2, the BLOBs should be sent directly to the L2 sequencer or relayed through a node that forwards them to the L2 sequencer.

## Uploading BLOB to Alt-DA

The sequencer is responsible for uploading BLOBs to the Alt-DA layer. When the
CL (op-node) of the sequencer receives the payload from its EL via the engine API, it should inspect the envelope for any `BlobsBundle`
and upload them to the Alt-DA. Only after ensuring successful BLOB uploads can the sequencer upload the block data
to the L1 DA. Similarly, the sequencer may need to respond to any data availability challenges afterward.

## Data Availability Challenge

Any third party, including full nodes deriving L2 data, might find they cannot access the BLOB corresponding to
the hash included in the BLOB transaction. In this case, they can initiate a data availability challenge. The
workflow will largely follow the Alt-DA process outlined
[here](https://github.com/ethstorage/specs/blob/l2-blob/specs/experimental/alt-da.md#data-availability-challenge-contract).

We will reuse the [DataAvailabilityChallenge][1] contract with minimal modification. First, since the BLOB hash in the BLOB
transaction is a
[VersionedHash](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md#helpers)
instead of a Keccak256 hash, we need to use the version hash as the commitment for BLOB uploading/downloading and
during challenge resolution. Therefore, we will add a CommitmentType to the DataAvailabilityChallenge contract:

```solidity
enum CommitmentType {
    Keccak256,
    Generic,
    VersionedHash
}
```

Additionally, a new challenge and resolve function should be added to the contract:

```solidity
function challengeL2BLOB(
    uint256 challengedOriginBlockNumber, 
    bytes calldata challengedCommitment
) external payable

function resolveL2BLOB(
    uint256 challengedOriginBlockNumber, 
    bytes calldata challengedCommitment,
)
```

This new resolve function should use an L1 BLOB transaction to upload the BLOB, then employ the EIP-4844 `blobhash()`
opcode to obtain the `versionedhash` of the BLOB.

Note that the parameter `challengedOriginBlockNumber` in both the new challenge and resolve functions refers to the
original L1 block number corresponding to the L2 block containing the BLOB being challenged. This change also requires
that the `challenge_window` must be larger than the `sequence_window` to prevent cases where the challenge window has
already passed by the time the sequencer submits the batch on L1

[1]: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L1/DataAvailabilityChallenge.sol

## Storage Requirement and BLOB Gas Cost

According to the EIP-4844 specification, BLOBs must be kept for at least
[MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS][2]
epochs, which is around 18 days. The storage upper limit can be calculated using the formula:
`BLOB_SIZE * MAX_BLOB_PER_BLOCK / BLOCK_TIME * MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS * SECONDS_PER_EPOCH`.
Assuming `MAX_BLOB_PER_BLOCK = 6` and `BLOCK_TIME = 2`, the upper limit is approximately 618 GB.

To serve data challenge purposes, we can reduce disk requirements even further. According to the Alt-DA
[specification][3], data only needs to be available for `l2blobChallengeWindow + l2blobResolveWindow`.
If `l2blobChallengeWindow` is 43200 seconds and `l2blobResolveWindow` is 3600 seconds, the disk requirement
will be around 17 GB.

These storage requirements are manageable for a commodity computer, and the cost of storing the data is minimal
compared to L1 data upload costs. This cost can be covered by adjusting the L1 `blobBaseFeeScalar` value.

[2]: https://github.com/ethereum/consensus-specs/blob/4de1d156c78b555421b72d6067c73b614ab55584/configs/mainnet.yaml#L148
[3]: https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/alt-da.md#data-availability-challenge-contract

## Derivation

Most of the derivation and reorg logic remains consistent with Alt-DA. However, instead of invalidating the entire
batch of the challenged origin, only the transaction containing the BLOB that is challenged and fails to be resolved
will be deleted. The basic workflow is as follows:

1. When the op-node derives a batch from the `BatchQueue`, it collects all the BLOB commitments included in the batch.
2. It tracks the challenge and resolution status for the collected BLOB hashes. If some BLOBs are challenged and fail
to be resolved, the op-node should trigger a `ResetError` to initiate a reorg.
3. If a reorg is triggered, the op-node will revert to an older block and rederive the chain. Upon encountering a
block containing the expired BLOB hash, it will remove the corresponding transaction and pass the remaining block
data to the EL.

## Fault Proof

The derivation pipeline integrates with fault proofs by adding additional hint types to the preimage oracle to query the on-chain challenge status.

### `l1-l2blob-challenge-status <commitment> <blocknumber>`

Retrieves the challenge status for the given `<commitment>` at the specified `<blocknumber>` on the L1
DataAvailabilityChallenge contract.

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

As mentioned above, L3s could integrate third-party DA services on their own, but they would need to manage the complexities of integration. Different DA services come with varying interfaces and security assumptions, especially concerning DA bridges. Having an enshrined 4844-compatible L2 DA layer would allow L3s (most of which are app-specific) to focus on their application logic, rather than spending time dealing with infrastructure.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

Should be similar to those described in the Alt-DA [spec](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/alt-da.md).

# Reference Implementation

1. [optimism repo changes](https://github.com/ethstorage/optimism/pull/23)
2. [op-geth repo changes](https://github.com/ethstorage/op-geth/pull/2)
