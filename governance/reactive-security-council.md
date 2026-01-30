# Reactive Security COuncil: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Alberto Cuesta Cañada                              |
| Created at         | 2025-08-25                                         |
| Initial Reviewers  | Kelvin Fichter, Matt Solomon                       |
| Need Approval From | _Reviewer Name_                                    |
| Status             | Draft                                              |


## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Our upgrade places a **rising pressure on our multisig members to verify upgrades**. Coordinating signers requires an enormous effort from Op Labs, which slows the speed at which we can ship safely. The rising requirements from our signers to sign more are in contradiction with our requirements for them to carefully verify each upgrade and pose a security risk.

## Customer Description

<!-- Provide a brief summary of the customers for this design document. -->

The customers for this design doc are the Optimism Foundation multisig owners, Security Council and OP Labs personnel working to coordinate upgrades. Also all other chain operators for which OP Labs manages upgrades.

### Customer Reviewers

<!-- Identify at least one customer who should be involved in the review of this document. -->

- Optimism Foundation (`LivenessModule` user)
- Optimism Security Council (`LivenessModule` user)
- Coinbase (Base)
- Uniswap (Unichain)

## Requirements and Constraints

<!-- Identify the solution requirements and any additional design constraints from the Context and
Problem Statement section in a bulleted list. -->

- Do not increase the risk of a bad upgrade

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

The proposed solution depends on the proposed TimelockGuard being enabled on the L1PAO, with a delay of at least a week. The L1PAO would be configured as a 1/2 multisig and the Optimism Foundation would regularly propose protocol upgrades. The Security Council would only intervene in emergencies to cancel an erroneous or malicious upgrade using the existing TimelockGuard cancellation feature. Oversight of upgrades would be reinforced using OP Labs resources, additional encouragement of bounty hunters, and third party contractors.

1. The L1PAO TimelockGuard delay will be verified to be at least 1 week.
2. The L1PAO will be set as a 1/2.
3. Hexagate monitoring will be set to alert of every transaction scheduled in the L1PAO, to protect against a Optimism Foundation multisig or timelock compromise.
4. Partners would be encouraged to monitor the L1PAO for scheduled transactions.
5. All transactions scheduled in the L1PAO timelock will be simulated and verified by EVM Safety to match the pre-scheduling behaviour.
6. Bug bounty hunters will be encouraged to do the same by the existing bug bounty rewards.
7. Third party contractors will be paid to do the same.

The impact of this feature would be that:
- **The Security Council wouldn’t be required on signing ceremonies**. The Security Council will acquire a reactive role that reduces the effort required from them, while preserving their duty and power to protect our assets. This will in turn reduce the effort we require to ship each feature, and will enable us to ship more often.


### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The only resource usage in consideration is time required to ship an upgrade. Removing the need to obtain signatures from the Security Council subtracts to the time required. Possibly 3 days.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

There is no impact on the clients.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

The alternative is to do nothing. The effort required to coordinate and secure multisig members would remain the same.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

### Loss of Security Council Reviews
The Security Council currently has a duty to verify all transactions. Currently they do this with the tools and guidance provided by EVM Safety, using no more than a few hours on each upgrade and often much less. While removing the upgrade reviewing duties from the SC weakens our security posture, this can be amply compensated by hiring third party professionals willing to develop custom tooling and spend a larger amount of focused time.

### Bug Bounty Failure
Reviewing proposals is complex, and bounty hunters are not used to the task. A persistent marketing and devrel campaign can be used to maintain interest and facilitate this work.

### Malicious Optimism Foundation
A compromise of the Optimism Foundation multisig would most likely lead to malicious upgrade proposals being scheduled. Redundant monitoring alerting of scheduled upgrades will be needed. Requesting from partners that they also monitor the L1PAO would be even better.


