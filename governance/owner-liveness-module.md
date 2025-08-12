# Owner Liveness Module

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Alberto Cuesta Cañada                              |
| Created at         | 2025/08/12                                         |
| Initial Reviewers  | [ ] John Mardlin                                   |
|                    | [ ] Kelvin Fichter                                 |
|                    | [ ] Matt Solomon                                   |
| Need Approval From | _Reviewer Name_                                    |
| Status             | Draft                                              |

## Purpose

Replace the current fixed-schedule liveness requirements for governance multisig signers with an on-demand challenge system. Enable the community to remove inactive signers when they become unresponsive, while protecting active signers from unnecessary burden.

## Summary

We introduce a Safe-compatible module that allows anyone to challenge the liveness of multisig owners by posting a bond. Challenged owners have a fixed time window to prove they are active by submitting a timestamped signature. If they fail to respond, they are automatically removed from the Safe. If they successfully respond, the challenger loses their bond and the Safe receives the funds. This eliminates the need for scheduled liveness checks while maintaining the ability to remove inactive participants.

## Problem Statement + Context

Current governance multisigs require every signer to demonstrate liveness every 90 days by either signing regular governance transactions or signing purpose-built liveness transactions.

This creates unnecessary overhead for active signers who must perform routine liveness checks even when their availability is not in question. The fixed schedule generates constant reminders and monitoring alerts, creating operational noise that doesn't align with when we actually need to verify signer availability. Signers may be temporarily unavailable but still capable of participating when needed, leading to false positives in our liveness detection.

We know signers are generally active and available. The current system forces proof of life on a schedule rather than when there's genuine concern about availability.

## Proposed Solution

Implement an on-demand challenge system that assumes liveness by default, with no scheduled requirements for active signers. Anyone can challenge suspected inactive signers by posting a bond, giving challenged signers a reasonable window to prove they can still participate. Challengers lose their bond if they target active signers, while genuinely inactive signers are automatically removed while preserving minimum thresholds and Safe integrity.

## Scope

- Enable bonded challenges against individual Safe owners to verify their continued ability to participate.
- Automatically remove unresponsive owners while preserving Safe operability.
- Provide economic incentives to discourage frivolous challenges.

## Non-goals

This system is not intended to replace all forms of governance oversight or monitoring, prevent legitimate owner rotation through standard Safe mechanisms, or guarantee that challenged owners are actually unavailable since they may choose not to respond for various reasons.

## Resource Usage

On-chain costs include challenge bonds held in the module and gas costs for challenge creation, defense, and finalization. Operationally, this reduces ongoing maintenance compared to scheduled checks, focusing attention only when challenges arise.

## Alternatives Considered

The status quo 90-day schedule is predictable but creates unnecessary overhead and doesn't align with actual availability concerns.

We also considered a permissioned module, where bonds are not required but only signers can challenge other signers. While possibly simpler, this alternative would brick the multisig if all signers become unresponsive at the same time.

## Risks & Uncertainties

It is unclear which bond size would be optimal to discourage harrassment of signers.

By switching to an on-demand liveness challenge, we need some other process to gather information on which signers might be unresponsive.

## References

- [Sunny Days Hackathon Proposal](https://docs.google.com/presentation/d/1utbGigIbMRA7JGcKZ9ZcUgMCdIPpLDJSwWmxNF9hBJM/edit?slide=id.g3734216eca8_11_0#slide=id.g3734216eca8_11_0)
- [Exploratory Implementation](https://github.com/ethereum-optimism/optimism/blob/sc/sss-olm/packages/contracts-bedrock/src/safe/OwnerLivenessModule.sol)