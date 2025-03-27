# [Project Name]: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Author Name_                                      |
| Created at         | _YYYY-MM-DD_                                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_                 |
| Need Approval From | _Reviewer Name_                                    |
| Status             | _Draft / In Review / Implementing Actions / Final_ |

## Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

The primary purpose of this design document is to outline a refactor of the governance process aimed at reducing the likelihood of a catastrophic governance action and reducing the burden on the Security Council. This document carefully considers the potential scenarios in which an attacker would be able to execute a malicious governance action with the proposed process.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

The proposed solution is to introduce a timelock to the governance process. The Foundation submits governance actions to the timelock once the vote passes, and a low quorum of the Security Council can remove proposals from the timelock if found to be malicious. While in the timelock, proposals are eligible for bug bounty rewards to encourage community verification.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Our governance process faces two problems:

1. The **existential risk of a catastrophic governance action** introduced by a malicious party and made possible by compromising the tools used by our multisigs. The Bybit hack stole $1.4B by compromising the Safe frontend. We have identified a similar risk on our dependencies to Github, Tenderly and Forge.
2. A **rising pressure on our multisig members to verify governance actions**. Coordinating signers requires an enormous effort from Op Labs, which slows the speed at which we can ship safely. The rising requirements from our signers to sign more are in contradiction with our requirements for them to carefully verify each governance action and pose a security risk.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

EVM Safety proposes to solve these two problems in the following way:

1. We propose to uniquely **identify governance actions as a commit** in the superchain-ops github repository. Governance actions would published in [gov.optimism.io](http://gov.optimism.io) a period before execution, and include hashed identifiers which allow for verification using a multitude of tools. **Published governance actions would be eligible for a bounty** exactly as other smart contract code. This protects us from supply chain attacks and Op Labs staff being compromised, and enables the points below.
2. We will add a new governance pipeline in which the **Foundation introduces governance actions to a timelock**, and a much reduced quorum of the **Security Council has veto powers**. The maximum impact from compromising a quorum of signers in **the new model is downgraded from catastrophic loss of assets to temporary denial of service**. The published and delayed execution of governance actions enables the point below.
3. **The Security Council won’t be required on signing ceremonies**. The Security Council will acquire a reactive role that reduces the effort required from them, while preserving their duty and power to protect our assets. This will in turn reduce the effort we require to ship each feature, and will enable us to ship smaller and safer features more often.

The following two sections describe the governance pipeline in detail, and a Failure Mode Analysis for the governance pipeline.

### Governance Pipeline

The proposed governance pipeline is described here in detail. For the significance of each step please refer to the Failure Mode Analysis in the next section.

1. A governance action is prepared in the superchain-ops repository, including an exact safe proposal id, target and calldata.
2. EVM Safety reviews the governance action **at a specific commit.**
3. A governance action is proposed in [gov.optimism.io](http://gov.optimism.io)
    1. The proposal includes **the same commit** that was reviewed by EVM Safety
    2. The proposal also repeats the safe proposal id, target and calldata
    3. Goes through whatever voting process we have
    4. Action is eligible for a bounty at the time it’s proposed
4. Once the proposal passes, the Foundation submits the governance action into a timelock
    1. Only the Foundation is allowed to use the timelock
    2. Timelock transactions include the safe proposal id, target and calldata and are identified by commit to prevent replaying the same proposal twice
5. Security Council, EVM Safety and Bounty Hunters have tooling and incentives to detect on the timelock any proposals not approved by governance, as well as proposals approved by governance but with unintended effects.
6. We have some mechanism that prevents the timelock from being abused if the Foundation is compromised and is frequently proposing malicious transactions
7. We have some mechanism that allows the Security Council and potentially also the EVM Safety team to cancel pending governance actions in the timelock
    1.  Any member or potentially any 2 members
8. The current 2/2 multisig could still be used as a backup without a timelock

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The only resource usage in consideration is time required to ship a governance action.
- The use of a timelock adds to the time required.
- Removing the need to obtain signatures from the Security Council subtracts to the time required.

Depending on final implementation, the time required to ship a governance action will be modified between -1 week and +1 week.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

There is no impact on the clients.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

The main failure mode is the execution of a malicious governance action. This scenario and other less severe ones are discussed in the [Failure Mode Analysis](../security/fma-timelock.md).

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

Publishing the proposal as a commit in [gov.optimism.io](http://gov.optimism.io), adding proposals to the bug bounty and using tooling to make sure that only the posted commit is submitted to the multisig would provide some benefits from community verification. The effort required to coordinate and secure multisig members would remain the same. This option requires less effort in the short term, but more effort in the long term.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

1. A malicious proposal would be submitted to the timelock by the Foundation despite our tooling. At that point the bug bounty and OP Labs monitoring would have a limited time to detect it and mobilize a member of the Security Council to cancel the proposal.
2. The Security Council might not be able to cancel a proposal in time. If the Security Council doesn't exercise their power regularly, they might become slow in reacting in an emergency.
3. The Foundation could be compromised and spam the timelock with malicious proposals. In that case the old 2/2 multisig would would need to halt the timelock.
4. An error in the execution of the proposal to introduce a timelock could affect the governance pipeline and let a malicious proposal be executed in an unexpected way.


