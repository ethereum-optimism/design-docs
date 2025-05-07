# External Block Production

#### Metadata

Authors: @dmarzzz, @avalonche, @ferranbt 

Created: September 19, 2024.

# Purpose

The purpose of this design-doc is to propose and get buy-in in on a first step towards building the ideal software and API for connecting proposers in the OP Stack (`op-node`) to an external block builder.

# Summary

This document proposes a sidecar to `op-node` for requesting block production from an external party. This sidecar has two roles: 1) obfuscate the presence of builder software from the `op-node` and `op-geth` software and 2) manage communication with a block builder and handle block delivery to `op-node`. The first role is achieved via the sidecar forwarding all API calls to it's local `op-geth` and delivering 1 block exactly for each block request from `op-node`. The second role is achieved by the sidecar implementing the communication protocol with the builder, including authentication, and payload selection rules.

By decoupling the block construction process from the Sequencer's Execution Engine, operators can tailor transaction sequencing rules without diverging from the standard Optimism Protocol Client. This flexibility allows individual chains to experiment on sequencing features, providing a means for differentiation. This minimum viable design also includes a local block production fallback as a training wheel to ensure liveness and network performance in the event of local Block Builder failure.

# Problem Statement + Context

## Problem Statement

The tight coupling of proposer and sequencer roles in the `op-node` limits the ability to implement customize transaction sequencing rules or optimize block production via external block builders. This manifests as a lack of flexibility in the OP Stack.

## Context

As of September 2024, the `op-node` sofware in `sequencer` mode performs both the role of "proposer" and "sequencer". As the "proposer", the `op-node` propagates a proposal, with full authority, for the next block in the canonical L2 chain to the network. Unlike a layer 1 "proposer", it does not have a "vote" in the finality of that block, other than by committing it to the L1 chain. As a "sequencer", it is also responsible for the ordering of transactions in the L2 block's it proposes. Today, it uses a `op-geth`, a diff-minimized fork of the Layer 1 Execution client `geth`, and it's stock transaction ordering algorithm.

On Ethereum Layer 1, a concept known as "Proposer Builder Separation" has become popularized as a client architecture decision to purposefully enable the "proposer" to request a block from an external party. These parties run modified versions of `geth` and newer clients like `reth` to build blocks with numerous ordering algorithms and features. On Layer 1, the communication between the proposer and the builder is achieved via the [`mev-boost` software](https://github.com/flashbots/mev-boost). 

# Proposed Solution

The proposed solution introduces a yet-to-be named "block builder sidecar" software which acts as an intermediary between the `op-node` and `op-geth` components within the OP Stack Proposer's system, as well as facilitating communication with the external Block Builder.

Key components and their interactions:

1. Proposer System:
   - `op-node`: Initiates the block production process via engine API calls.
   - block builder sidecar: Forwards all API calls to local `op-geth` and multiplexes `engine_FCU` (with Payload Attributes)
   - `op-geth`: The local execution engine.
   - `Eth JSON RPC`: Accepts standard RPC requests from user's and multiplexes `eth_sendRawTransaction` to the builder client. This can be achieved in the chain operators load balancer or with a simple nginx-style proxy. Future work could perform multiplexing of transactions through the sidecar to ensure that the block builder is not censor'ing.

2. Builder System:
   - `builder-op-node`: A stock version of the op-node client.
   - `builder-op-geth`: The external block builder's execution engine.

4. Communication Flow:
   - The `op-node` sends an `engine_FCU` (Fork Choice Update) call with attributes to the sidecar.
   - The sidecar forwards this call both to the local op-geth and to the builder-op-geth if the node is synced to the tip of the chain.
   - When the op-node makes an `engine_getPayload` call, the sidecar intercepts it.
   - The sidecar communicates with the external `builder-op-geth` to request an externally built block.
   - Upon receiving the external block, the sidecar validates it with the local `op-geth` node. If valid, it forwards it to the op-node, otherwise it falls back to the locally produced block.

This design achieves the two main roles of the sidecar:

1. Obfuscation: The presence of the external builder is hidden from both op-node and op-geth. From their perspective, they are communicating with each other as usual.

2. External Communication Management: The sidecar handles all communication with the external builder, including authentication and payload selection.

### Liveness Failsafe
To maintain network liveness while utilizing the "block builder sidecar", the local block from the Proposer's `op-geth` will be used in the event a block builder's payload is invalid or does not come in time. This fallback mechanism should be seen as a training wheel.

### Boost sync

Our experience has shown that there can be a notable delay in the synchronization process of the builder-op-geth with the latest tip of the chain. When relying solely on p2p communication and the op-node, we've observed this delay to typically range from about 200 to 300 milliseconds, though this can vary. To address this issue and improve efficiency, we are implementing a `boost sync` mechanism to rapidly transmit the tip of the block to the builder-op-geth.

The boost sync process operates as follows:
- The sidecar monitors the `builder-op-geth` sync status.
- If the sidecar detects that the `builder-op-geth` is synced with the tip of chain, it relays the `engine_newPayload` and `engine_FCU` (without attributes) API calls.

Preemptively sending these API calls from the sidecar, instead of waiting for them to come from the op-node, helps the builder sync up the latest block faster. We have observed times reduced to approximately 3-4 ms.

## Resource Usage

This approach doubles the amount of bandwidth needed for sending a block to the `op-node` by accepting an external block from the block builder. However, the sidecar only forwards one payload to `op-node` based on its selection criteria.

## Software Maintence

Flashbots will develop and maintain the initial versions of this software in a modular and contributor friendly manner to the standards of our existing Ethereum L1 `mev-boost` sidecar. We will take a crawl, walk, run approach with this software by trial'ing it with one OP-stack chain outside of local testing. From there, we can decide if the feature set is standardized enough to begin efforts to merge into the OP-stack, or in the event we want to delay this decision further, Flashbots will contribute this sidecar to the docker compose setup of OP stack and assist in ensuring smooth operation during any hardfork related work as we have historically done on Ethereum L1.

## Tradeoffs

### Benefits

1. Flexibility: Allows for customization of transaction sequencing rules without modifying either core Optimism Protocol Clients.
2. Experimentation: Enables individual chains to test different sequencing features.
3. Fallback Mechanism: Allows for a local block production fallback to ensure liveness and network performance if the external Block Builder fails.
4. Minimal Modifications: Requires no changes to existing `op-node` or `op-geth` components, simplifying integration and maintenance until an integrated solution is designed.

This solution provides a balance between enabling external block production and maintaining the integrity and simplicity of the existing system architecture without enshrining any premature decisions.

### Costs

1. It breaks any existing and future assumptions around there being 1 execution layer for each consensus layer client in the OP Stack.
2. Adding software between a source and desintination will always incur some latency hit.
3. Without propoer illumination, it could make portions of the protocol opaque to the user, but this may be true of any custom ordering rule.
4. A working solution could delay an in-protocol solution indefinitely due to lack of urgency to merge in.

## Resource Usage

This approach doubles the amount of bandwidth needed for sending a block to the `op-node` by accepting an external block from the block builder. Bu the sidecar only forwards one payload to `op-node` based on it's selection criteria.

# Alternatives Considered
A variety of alternate desgns we're considered and some implemented.

1. Proposer `op-node` <> `builder-op-geth` (payload attributes stream):
   - Proposer's` op-node` requests block from builder's op-geth.
   - Proposer's` op-node` provides payload attributes stream to Builder'sop-geth.
   - Pros: uses alternative and customizable communication channel for payload attributes stream.
   - Cons: requires modification to `op-node`.

2. Proposer `op-node` <> Builder `op-geth` (no payload attributes stream):
   - Similar to design 1 above, but without payload attributes stream.
   - Block building triggered by enabling block payload attributes in engine_forkchoiceUpdated call.
  - Pros: fewer modifications than design 1.
   - Cons: stil requires modification to `op-node`.

3. Proposer `op-node` <> Builder `op-node`: (builder API)
   - Proposer's` op-node` requests block from builder's` op-node` using a special API.
   - Pros: more modifications to `op-node` than design 2 but less than design 1.
   - Cons: allows for more flexibility in information requested from block builders since request isn't restricted to payloadAttributes.

4. Proposer `op-geth` <> Builder `op-geth`:
   - Proposer's op-geth requests block directly from builder's op-geth.
   - Pros: likely the fastest approach since proposers `op-geth` is the ultimate executor of the payload.
   - Cons: requires modification to `op-geth` which is in some ways more sacred than `op-geth` due to it's policy of a minized code diff to upstream geth.

# Risks & Uncertainties

These questions, along with protocol discovery, will be the main goal of documenting during the build out and productionizing of this first step.

1. **Architectural assumption:** The proposed solution would challenge any existing assumption of one Execution Layer (EL) client for each Consensus Layer (CL) client in the OP Stack, potentially leading to unforeseen consequences.

2. **Performance impact:** Introducing a sidecar component between `op-node` and `op-geth` may introduce additional latency in block production and propagation, impacting overall system performance.

3. **Security vulnerability:** The sidecar introduces a new attack surface that needs to be secured, particularly in terms of authentication and authorization between the sidecar and external block builders.

4. **Maintenance:** Future updates to `op-node` or `op-geth` may break the sidecar's functionality if it relies on specific interfaces or behaviors, creating ongoing compatibility challenges that need to be incorporated in the clients e2e testing environment.

5. **Increased complexity:** While the sidecar approach minimizes changes to existing components, it adds complexity to the overall system architecture, potentially making it harder for new developers to understand and contribute to the project.
