# Smart Batching: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | @jbearer (Espresso Systems)                        |
| Created at         | 2025-03-25                                         |
| Initial Reviewers  | Eli Haims, Sam McIngvale                           |
| Need Approval From |                                                    |
| Status             | In Review                                          |

## Purpose

Enabling plugin logic for application-specific sequencing or batching behavior without the need to fork the derivation pipeline.

## Summary

Enables chains to deploy a contract at the batch inbox address with custom logic for filtering
batches. The derivation pipeline is modified to ignore batches where the contract reverts. This
expands the design space for OP stack rollups running the canonical derivation pipeline.

Example uses:

- TEE-sequencer: L1 enforces that the rollup sequencer runs in a TEE, which enforces additional sequencing policies:
    - Unichain wants to use a TEE for Flashblocks and verifiable priority ordering. Currently the derivation pipeline cannot check the TEE attestation, so the current implementation is no better than a trusted sequencer. This proposed change would provide an avenue to fix that. It will also be needed for the UVN.
    - The Celo sequencer will run a TEE that first posts its transaction batches to a fast BFT consensus (run by Espresso Network) and only posts batches to the L1 that were already confirmed by this BFT consensus. This allows users and protocols (exchanges, bridges, etc) to obtain rollup soft confirmations (pre-L1 finality) from the fast BFT consensus that are more secure than confirmations coming from a regular centralized sequencer. Other OP stack chains are interested in adding this protective layer as well.
- L1 directly verifies that the sequenced batches have been confirmed by an external layer (e.g. Espresso) instead of relying on a TEE.
- Enabling various decentralized sequencing policies, by deploying an inbox contract with logic for determining which sequencer is eligible to post any given batch (similar to [RFP](https://github.com/ethereum-optimism/ecosystem-contributions/issues/63) that Espresso completed for OP Labs)

## Problem Statement + Context

It is desirable for many chains to customize the behavior of the OP stack. This is always possible
by forking and modifying the derivation pipeline, but it is desirable to be able to run the canonical stack, with custom behavior as plugins. OP stack natively supports customization of certain aspects of the rollup (such as alt-DA layers) but lacks sufficient customizability to deploy new batching and sequencing policies.

A simple way for rollups to deploy custom batch posting rules without needing to run a fork of
validators and RPC nodes is to deploy a *batch inbox* contract on L1, which acts as a pre-filter to
the derivation pipeline, rejecting any batches that don't satisfy the custom rules while
allowing the derivation pipeline to ingest the remaining batches and execute them as normal.

Unfortunately, this is not quite possible with the current derivation pipeline. While the inbox
address can be configured to the address of a contract, the contract cannot stop batches from being ingested by the derivation pipeline, because **the derivation pipeline does not check revert status of transactions**. Thus, even if the contract reverts, the reverting transactions can still end up included in an L1 block, and the derivation pipeline will still parse and execute their calldata or
blob data.

## Proposed Solution

Modify the L1 derivation stage of the pipeline (`calldata_source` and `blob_data_source`) to fetch receipts for each L1 block they process, and skip any transaction where the `status` field of the receipt is 0 (reverted transactions).

### Compatibility

The change is backwards compatible. If the inbox address is not a contract, transactions to it will never revert. Thus the only action necessary to opt in is to deploy a contract and upgrade to change the batch inbox address.

### Resource Usage

The derivation pipeline already fetches transaction receipts in several places (e.g. attributes
derivation) and it caches them when fetched. Thus, adding an additional place where they are fetched does not meaningfully increase the cost of fetching L1 data.

There may be additional costs related to the deployment and operation of the contract, but that is
rollup-specific and opt-in.

### Single Point of Failure and Multi Client Considerations

There are no changes in op-geth, only op-node.

## Alternatives Considered

- Using alt-DA to implement custom batching rules. It is possible to put custom batch filtering
logic in a DA implementation; however the `altda_data_source` treats any L1 input data which is not a DA commitment as a normal batch, allowing malicious sequencers/batchers to bypass filtering logic in the DA layer.
    
  Removing this fallback is another way to implement the desired plugin functionality; but this is undesirable even if it only applies to rollups opting into the new feature, as they may still want the ability to bypass the alt DA layer for liveness purposes.
    

## Risks & Uncertainties

Introducing an additional filtering stage to the pipeline could incur liveness risks depending on
the custom logic being deployed. This is rollup-specific and opt-in.

Note that *regardless* of the custom logic deployed, this feature does not incur any additional risk
of safety violations, double spends, etc. as the only change in op-node itself is an additional way
a batch can be skipped over. There is no additional or modified code path for accepting a batch or changing how batches are interpreted and executed.
