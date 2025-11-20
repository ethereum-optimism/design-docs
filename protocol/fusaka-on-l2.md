# Fusaka on L2: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Josh Klopfenstein                                  |
| Created at         | 2025-11-06                                         |
| Initial Reviewers  | TBD                                                |
| Need Approval From | TBD                                                |
| Status             | _Draft_                                            |

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

We need to decide which Osaka EIPs to inherit from L1, and the modifications that may be necessary.
This document outlines options for doing so and suggests next steps.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

The following EIPs were added in the L1 Fusaka hardfork but are not relevant to L2.

| EIP | Why irrelevant? |
|-----|-----------------|
| [7594 PeerDAS](https://eips.ethereum.org/EIPS/eip-7594) | Blob optimization |
| [7892 BPO Forks](https://eips.ethereum.org/EIPS/eip-7892) | Blob optimization |
| [7917 Deterministic Proposer Lookahead](https://eips.ethereum.org/EIPS/eip-7917) | L1 CL detail |
| [7918 Bounded Blob Base Fee](https://eips.ethereum.org/EIPS/eip-7918) | Blob optimization |
| [7935 Raise Default Gas Limit](https://eips.ethereum.org/EIPS/eip-7935) | OP Stack operator can set the block gas limit |
| [7934 RLP EL Block Size Limit](https://eips.ethereum.org/EIPS/eip-7934) | Sequencer policy is sufficient |

The following are the Fusaka EIPs that can be adopted on L2.

| EIP |  Adoption Questions  |
|-----|----------------------|
| [7642 eth/69](https://eips.ethereum.org/EIPS/eip-7642)\* | |
| [7823 Upper-Bound MODEXP](https://eips.ethereum.org/EIPS/eip-7823) | |
| [7825 Tx Gas Limit Cap](https://eips.ethereum.org/EIPS/eip-7825) |  What should the cap be on L2? Should it be configurable? |
| [7883 MODEXP Gas Cost Increase](https://eips.ethereum.org/EIPS/eip-7883) | |
| [7910 eth_config Endpoint](https://eips.ethereum.org/EIPS/eip-7910)\* | |
| [7939 CLZ Opcode](https://eips.ethereum.org/EIPS/eip-7939) | |
| [7951 secp256r1 Precompile](https://eips.ethereum.org/EIPS/eip-7951) | Already adopted: should L2s use the new gas cost? |

\* Can technically be implemented outside of a hardfork, but we intend to ship them as part of a hardfork since they require multi-client compatibility.

We need to decide the following:

* Among the Relevant EIPs:
  * Answer the adoption questions.
  * Are there any other adoption concerns associated with the listed EIPs?
* Among the Irrelevant EIPs, are there any relevant EIPs that should be included on L2?

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

Assuming the division of relevant and irrelevant EIPs is correct, we need to answer the adoption questions.

### 7825 Tx Gas Limit Cap: What should the cap be on L2? Should it be configurable?

We propose adopting a constant cap that matches L1: 16,777,216 gas (2^24).
This is the simplest option that maximizes Ethereum equivalence.
If there is a strong need for a different cap or flexibility in setting the cap, that can be considered in future hardforks.

### 7951 secp256r1 Precompile: Should L2s use the new gas cost?

The OP Stack already has a `p256Verify` precompile added in Fjord as specified in [RIP-7212](https://github.com/ethereum/RIPs/blob/bd9b89f0b02d26a579ef8972431bc93540314dd4/RIPS/rip-7212.md).
Both use the same `0x100` address ([L1](https://github.com/ethereum-optimism/op-geth/blob/aa940538653c59df97b432bc24a2a4e4b17a06a8/core/vm/contracts.go#L168), [L2](https://github.com/ethereum-optimism/op-geth/blob/aa940538653c59df97b432bc24a2a4e4b17a06a8/core/vm/contracts.go#L190)), but the L1 version costs [more gas](https://github.com/ethereum-optimism/op-geth/blob/aa940538653c59df97b432bc24a2a4e4b17a06a8/params/protocol_params.go#L181-L182) (3450 vs. 6900).
We propose increasing the L2 gas amount to the L1 gas amount. This maintains the OP Stack's strict equivalence with L1 opcode and precompile costs that limits impact on tooling and maximizes forward-compatibility.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

For 7825:

* Implement a configurable in-protocol cap. The complexity does not seem justified at this time.
* Defer changing the protocol and implement a sequencer policy parameter instead. Possible, but this still requires extra effort for what appears to be minimal benefit.

For 7951, the primary alternative is to ignore L1 and keep the smaller gas amount. This would be the first and only case where L2 diverges from L1 opcode and precompile pricings, which seems unjustified in this case.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

* Adding a transaction gas limit cap breaks use cases and/or makes application development more difficult. While possible, it is unclear that any application would be impossible to build with a tx gas limit cap. While some modifications to existing apps may be necessary, encouraging developers to use multiple transactions helps with scalability (especially after BALs/FALs ship).
* Changing the `secp256r1` precompile price breaks use cases that rely on the cheaper cost. The impact is judged to be minimal: at current prices on OP Mainnet, the dollar-denominated difference in a single call is fractions of a cent.

If any of these risks are perceived to be significant, we can collect onchain data to gain confidence in the final decisions.
