# Osaka Features on Karst

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [FM1: EIP-7825 Tx Gas Limit Cap Affects L2CM Deployment or Operation](#fm1-eip-7825-tx-gas-limit-cap-affects-l2cm-deployment-or-operation)
  - [FM2: EIP-7825 Tx Gas Limit Cap Breaks Existing On-Chain Usage](#fm2-eip-7825-tx-gas-limit-cap-breaks-existing-on-chain-usage)
  - [FM3: Osaka EVM Changes Not Correctly Implemented in Fault Proof Programs](#fm3-osaka-evm-changes-not-correctly-implemented-in-fault-proof-programs)
  - [FM4: EIP-7951 secp256r1 Gas Cost Increase Breaks Existing Contracts](#fm4-eip-7951-secp256r1-gas-cost-increase-breaks-existing-contracts)
  - [FM5: EIP-7883/EIP-7823 MODEXP Changes Break Existing Contracts](#fm5-eip-7883eip-7823-modexp-changes-break-existing-contracts)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- _Italics are used to indicate things that need to be replaced._ -->

|                    |                                     |
| ------------------ | ----------------------------------- |
| Author             | Josh Klopfenstein                   |
| Created at         | 2026-04-19                          |
| Initial Reviewers  | todo                                |
| Need Approval From | todo                                |
| Status             | Implementing Actions 🛫             |

| Related References | Links                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------- |
|  |  |

<!--
> [!NOTE]
> 📢 Remember:
>
> - The single approver in the “Need Approval From” must be from the Security team.
> - Maintain the “Status” property accordingly. An FMA document can have the following statuses:
>   - **Draft 📝:** Doc is created but not yet ready for review.
>   - **In Review 🔎:** Security is reviewing, and Engineering is iterating on the design. A checklist of action items will be created during this phase.
>   - **Implementing Actions 🛫:** Security has signed off on the content of the document, including the resulting action items. Engineering is responsible for implementing the action items, and updating the checklist.
>   - **Final 👍:** Security will transition the status of the document to Final once all action items are completed.

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the project. The FMA review process can be accelerated by proactively identifying action items during the writing process. -->

## Introduction

This document covers implementation of Osaka features in the OP stack. This requires a hard fork due to backwards-incompatible EVM changes such as new opcodes. This does not cover generic hardfork related security concerns.

All Fusaka EIPs are copied below in two separate tables: one for Fusaka EIPs not adopted on L2, and one for EIPs that are adopted. Similar tables appear in the [design doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/fusaka-on-l2.md).

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

| EIP |  Implementation Notes |
|-----|----------------------|
| [7642 eth/69](https://eips.ethereum.org/EIPS/eip-7642)\* | |
| [7823 Upper-Bound MODEXP](https://eips.ethereum.org/EIPS/eip-7823) | |
| [7825 Tx Gas Limit Cap](https://eips.ethereum.org/EIPS/eip-7825) |  |
| [7883 MODEXP Gas Cost Increase](https://eips.ethereum.org/EIPS/eip-7883) | |
| [7910 eth_config Endpoint](https://eips.ethereum.org/EIPS/eip-7910)\* | |
| [7939 CLZ Opcode](https://eips.ethereum.org/EIPS/eip-7939) | |
| [7951 secp256r1 Precompile](https://eips.ethereum.org/EIPS/eip-7951) | Existing L2 P256Verify precompile gas cost is updated |

\* Can technically be implemented outside of a hardfork, but we intend to ship them as part of a hardfork since they require multi-client compatibility.

## Failure Modes and Recovery Paths

### FM1: EIP-7825 Tx Gas Limit Cap Affects L2CM Deployment or Operation

- **Description:** If L2CM deployment happens after EIP-7825 is activated and costs more than 2^24 gas, it will fail. Any operations after deployment must also execute within by the new transaction gas limit cap.
- **Risk Assessment:** Low severity, medium likelihood.
- **Mitigations:**
  1. L2CM tests verifying that deployment and L2CM operation is unaffected by EIP-7825.
- **Detection:** An L2CM deployment or operation reverts.
- **Recovery Path(s):** Refactor the L2CM to execute within the transaction gas limit cap.

### FM2: EIP-7825 Tx Gas Limit Cap Breaks Existing On-Chain Usage

- **Description:** EIP-7825 introduces a transaction gas limit cap of 2^24 (16,777,216). Any existing contracts or dApps on OP chains that rely on transactions exceeding this cap would break after activation. Additionally, deposit transactions from L1 or other system transactions could be affected if they exceed this cap.
- **Risk Assessment:** Medium severity, low likelihood.
- **Mitigations:**
  1. Analysis of on-chain transaction gas usage across OP Mainnet chains to confirm no transactions exceed 2^24 gas.
  2. Verification that deposit transactions and all system transactions execute within the cap.
- **Detection:** User transactions or deposit transactions reverting unexpectedly after activation.
- **Recovery Path(s):** Affected contracts would need to be refactored to use multiple transactions. If deposit transactions are affected, an emergency client update would be required.

### FM3: Osaka EVM Changes Not Correctly Implemented in Fault Proof Programs

- **Description:** Osaka introduces several EVM changes that fault proof programs (op-program, Kona, etc.) must implement correctly: the new CLZ opcode (EIP-7939), updated MODEXP gas cost and input bounds (EIP-7883/EIP-7823), and the secp256r1 precompile gas cost change (EIP-7951). If any of these are missing or diverge from on-chain behavior, blocks containing affected transactions could become unprovable, potentially allowing invalid state transitions.
- **Risk Assessment:** High severity, low likelihood.
- **Mitigations:**
  1. Implement the CLZ opcode in all fault proof programs.
  2. Verify that FP programs correctly implement the updated MODEXP gas cost and input bounds.
  3. Verify that FP programs correctly implement the updated secp256r1 precompile gas cost.
  4. End-to-end proof tests exercising each changed opcode/precompile in FP programs.
- **Detection:** op-program trace runner failure, potentially triggering an alert.
- **Recovery Path(s):** Emergency update to fault proof programs to fix the divergent behavior.

### FM4: EIP-7951 secp256r1 Gas Cost Increase Breaks Existing Contracts

- **Description:** The P256Verify precompile gas cost doubles from 3,450 (RIP-7212, shipped in Fjord) to 6,900 (EIP-7951). Contracts that call P256Verify with a hardcoded gas stipend between 3,450 and 6,900 would fail silently after activation, since the forwarded gas would be insufficient for the precompile to execute.
- **Risk Assessment:** Medium severity, medium likelihood.
- **Mitigations:**
  1. Communication of the breaking change to developers ahead of activation.
- **Detection:** User transactions calling P256Verify reverting or returning unexpected results after activation.
- **Recovery Path(s):** Affected contracts would need to be upgraded to forward sufficient gas. No protocol-level fix needed.

### FM5: EIP-7883/EIP-7823 MODEXP Changes Break Existing Contracts

- **Description:** EIP-7883 increases the gas cost of the MODEXP precompile, and EIP-7823 introduces an upper bound on MODEXP input sizes. Together, these changes could (a) break contracts that call MODEXP with hardcoded gas stipends that are now insufficient, or (b) cause previously valid MODEXP calls with inputs exceeding the new upper bound to revert.
- **Risk Assessment:** Low severity, medium likelihood.
- **Mitigations:**
  1. Communication of the breaking change to developers ahead of activation.
- **Detection:** User transactions calling MODEXP reverting after activation.
- **Recovery Path(s):** Affected contracts would need to be upgraded.
