# Purpose

This document describes the design of Plasma Mode for different DA Layers with minimal op-node & specs repo changes.
Once ratified, we will implement the described design.

# Problem Statement + Context

Plasma Mode is a modification the OP Stack protocol where the data is posted to a dedicated DA layer instead of Ethereum. This is done to reduce the cost of operating the specific chain. To maintain security, we provide a mechanism to challenge commitments and make data available on Ethereum when there is a dispute over the commitment.


We have the following requirements for Plasma Mode:
- Data is posted to a DA layer instead of Ethereum
- There is a DA Challenge which validates that the data is available
- Optionally, the DA challenge process makes the data available on Ethereum
- The op-node is able to mark L2 as finalized when commitments used to derive the L2 chain are no longer able to marked as invalid

Additionally, we take the following development-process requirement:
- The op-node and specification repo does not need to be modified for each new DA Layer

The largest challenge to this developement process requirement is that DA Providers have different ways of committing to data and semantics of challenges. We need to create a design that generically supports a broad set of DA Providers without heavy modification or forking of the OP Stack.

## High Level Overview of Plasma Mode

Instead of posting data directly to L1, the batcher gives the data to a DA server which posts the data to a DA layer.
The DA server returns a commiment to the data to the batcher & the batcher posts that commitment to L1.
The op-node performs derivation with the modification that when it finds a commiment on L1, it asks its DA server to get the data associated with the commitment.
Then the rest of the derivation pipeline functions as normal.
Finalization is modified because a L2 block can only be finalized when the commitment that was used to derive it can no longer be challenged.
Commitment finalization takes significantly longer in Plasma Mode because of the challenge period.
Data from commitments is also used prior to knowing if the commitment is valid or not.

# Proposed Solution

We propose a generic commitment type which can be interpreted by a DA server to fetch & validate the data. Challenge commitments is done in per DA layer smart contract where the smart contract emits events about the status of the challenge. The op-node then interprets those events to handle commitment validity and finality.

## DA Server Spec

The DA Server is [specified here](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/plasma.md#da-server).
At a high level, the DA server implements a HTTP `get` that the op-node uses to fetch data and a HTTP `put` that the batcher uses to post data.
The DA Server is responsible for verifying that the data returned from the DA layer matches the commitment.


### Alternatives

We considered moving the Data Fetching logic into the op-node. It would be possible to place it the fetcher component behind a Go interface; however we rejected this solution in favor of putting the fetching logic over a network boundary.
Moving the logic to a dedicated HTTP server means that the same op-node binary can work with multiple layers.
The other option to avoid compiling in the DA Layer specific logic is to use [go plugins](https://pkg.go.dev/plugin). We reject this solution because it requires that the DA fetching logic be implementing in Go and it add places restrictions on how the resulting binary may be used.

## Generic Commitment

The generic commtiment is [specified here](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/plasma.md#input-commitment-submission).
The relevant part is copied to this document:

The da-service commitment is as follows: `da_layer_byte ++ payload`. The DA layer byte must be initially restricted to the range `[0, 127)`. This specification will not apportion DA layer bytes, but different DA layers should coordinate to ensure that the DA layer bytes do not conflict. DA Layers can do so in [this discussion](https://github.com/ethereum-optimism/specs/discussions/135). The payload is a bytestring which is up to the DA layer to specify. The DA server should be able to parse the payload, find the data on the DA layer, and verify that the data returned from the DA layer matches what was committed to in the payload.


### Alternatives

We considered and rejected making the op-node aware of each DA Layer's commitment format.
This would allow the op-node to be stricter about validating which DA layers are being used and relies less on the DA Server.
The downside is that this requires each DA Layer to integrate with the specs and op-node source.


## Data Availability Challenge

The Data Availability Challenge ensures that nodes are able to fetch the data corresponding to the commitment posted on L1.
If the data is not available, nodes should not attempt to use the commitment.
If the data is only made available to a small portion of nodes, it should either be made available to all nodes by posting the data to L1 or the commitment should be rejected.
Ensuring that nodes have a consistent view of the data is important to ensure that nodes derive the same chain and the fault proofs function correctly.

Each DA Layer will have to implement logic on L1 to validate their commitments. This logic will emit a specific set of events that the op-node will read.
These events then get adapted into a golang interface that the derivation pipeline uses to manage L2 reorgs & finality.

### Data Availability Challenge Smart Contract Specification

#### V1: Optimisic Bridge
The DAC SC will emit the following event
```solidity
enum ChallengeStatus {
    Uninitialized, // Never emitted
    Active,        // Emitted when a challenge is initiated
    Resolved,      // Emitted when a challenge is resolved. This means the data has been made available on L1.
    Expired        // Never emitted
}

event ChallengeStatusChanged(bytes indexed challengedCommitment, uint256 indexed challengedBlockNumber, ChallengeStatus status);
```

The flow is that after a commitment is posted to L1, it can be challenged within `challenge_window` blocks.
If it is challenged an event with the `Active` status is emitted.
If the challenge has not been resolved within `resolve_window` blocks, the challenge is presumed to suceed and the commitment is invalid.
If the challenge is resolved within `resolve_window`, an event with the `Resolved` status is emitted and the commitment is valid.
If a commitment is never challenged within `challenge_window` blocks, it is valid.

This works essentially as an optimistic bridge. The commitment is assumed to be valid until someone claims that it is not.
The validity is checked by a third party smart contract which understands the commitment format.


#### V2: ZK Bridge

DA Layers which have ZK bridges from their DA Chain to Ethereum can write a SC which emits the following event.

```solidity
    event MarkedInvalid(bytes indexed challengedCommitment, uint256 indexed challengedBlockNumber)
    event MarkedValid(bytes indexed challengedCommitment, uint256 indexed challengedBlockNumber)
```

The flow is that after a commitment is posted to L1, it can be challenged within `challenge_window` blocks.
If the challenge succeeds, the `MarkedInvalid` event is emitted. The `MarkedInvalid` event should not be emitted unless the data is not available for all nodes.
If the challenge fails, the `MarkedValid` event is emitted. If the `MarkedValid` event is emitted, the data MUST be available for all nodes.

### Derivation Integration

The op-node uses the following interface inside the derivation pipeline.
```go
type CommStatus int
const (
    OptimisticOK     = iota
    ConfirmedValid
    ConfirmedInvalid
)
func CommitmentStatus(comm GenericCommitment, L1Source eth.BlockID) (CommStatus, error)
```

Commitment Status Defintions:
- `OptimisticOK` means that the commitment is assumed to be valid, but could be challenged & invalidated later
- `ConfirmedValid` means that the commitment cannot be invalidated on the given L1 chain
- `ConfirmedInvalid` means that the commitment has been invalited & will not become valid on the given L1 chain

We adapt the SC events into this golang interface to support multiple different bridge types without having to modify derivation.
In addition, this confirmation model is more closely aligned to how the derivation pipeline works with commitment validity.

When integrating with derivation, safe blocks are derived from commitments that are OptimisiticOK.
If a commitment is marked as `ConfirmedInvalid`, the L2 safe chain must be reorged via a pipeline L2 Reset if that commitment was used.
If the DA Server was not able to find the data for that specific commitment, the commitment can be skipped.
When a commitment is marked as `ConfirmedValid`, we can finalize portions of the L2 chain. The exact finalization algorithm is described [here](#Finalization)

The DA server MUST look at the challenge contract on L1 to find the data from the commitment when it is resolved with an Optimistic bridge.
The DA server MAY look at the challenge contract on L1 to find the data from the commitment when it is resolved with an ZK bridge; however, it is a breaking change to switch from not looking at the challenge contract to looking at the challenge contract.

By only looking at an opaque commitment and letting the DA server be responsible for fetching the data & letting the DA Challenge contract be responsible for validating the data, no parts of the monorepo need to be modified to work with a new DA Layer.


### Alternatives

We initially considered the following options for designing a DA challenge. These where rejected because they are too DA layer specific inside the op-node.
* The op-node does verification of data (no bond payouts / commitment validity checks on L1)
* Modify the existing DAC bridge to work with different commitments.
* Integrating a ZK Bridge (for example Blobstream).

We did consider the following as legitimate generic options.

We could combine the V1 and V1 SC events into a single interface. This would mean that there is only one adapter from L1; however it would mix together bridges of different types and increase implementation complexity.

We also could directly read SC events in the derivation pipeline (and we do this at the moment); however, it requires mixing the V1 and V2 interfaces into. In addition converting to the status events works better inside the pipeline & enables future optimizations.


### Future Work

In the future, we can remove the SC adapter & the DA server can become responsible for providing the status information.
The DA server would be able to return `ConfirmedValid` or `ConfirmedInvalid` when it knows that a transaction on L1 would result in the same event.
The benefit of L1 execution for `ConfirmedValid` is that we will be able to significantly reduce the finalization delay for DA Layers that have timely ZK bridges.

## Finalization

At a high level, finalization is process of marking L2 blocks as not being able to be be reorged.
If a commitment is marked as invalid, we have to reorg the L2 chain, therfore finality in Plasma Mode is about tracking when commitments finalize.

To implement finality, we track commitments (even skipped or invalid commitments in order). When the derivation pipeline has advanced to the point where
the commitment status changes from `OptimisticOK` to `ConfirmedValid` or `ConfirmedInvalid`, we record the L1 block in which that occurred.
We then wait for that L1 block to be finalized. Once that L1 block has been finalized, the commitment status can never change.
The L1 block in which the commitment status became unchangeable is then passed as a finalized block to the L2 Finality controller.
L2 blocks which are fully derived from a block below that L1 block can be finalized.

An alternate approach is to run a second copy of derivation which only advances when commitments are no longer challengeable in finalized blocks.
This is proposal helps inform how we should implement finality tracking; however, we can track the L2 Safe data without running a second copy of the derivation pipeline.


# Risks & Uncertainties

## DA Server / Challenge Code Quality Integration

External parties will be responsible for both the DA server and DA Challenge Contract.
This means that code will not follow the standard developement process of the op-stack.

## Superchain Registry Integration

Chains in the superchain registry should do what they say they will be doing.
This includes the DA layer that they choose to use.
There is no in-protocol enforcement of the DA Layer until Fault Proofs are live.
Out of protocol enforcement can be done by running a node.

## Fault Proof Integration

Fault Proofs will eventually require different DA Layer validation to be compiled into the op-node.
We will do this with a standardized golang interface. Because of the FP Architecture, the different execution properties are very simple to include.

Data on the DA layer will have to be made available to the pre-image oracle and there will be a new hint type to get data. See the [Plasma Mode specs here](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/plasma.md#fault-proof).

# Open Questions

* Do we ensure that the data becomes available on L1 after a DA challenge?

