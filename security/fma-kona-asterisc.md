<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
  - [[Name of Failure Mode 1]](#name-of-failure-mode-1)
  - [[Name of Failure Mode 2]](#name-of-failure-mode-2)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)
- [Appendix](#appendix)
  - [Appendix A: This is a Placeholder Title](#appendix-a-this-is-a-placeholder-title)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Stage 1.4: Kona and Asterisc FMA (Failure Modes and Recovery Path Analysis)

| | |
|--------|--------------|
| Author | Andreas Bigger |
| Created at | 2024-10-17 |
| Initial Reviewers | *Reviewer Name 1, Reviewer Name 2* |
| Need Approval From | Ben Clabby, Mark Tyneway |
| Status | Draft |

> [!TIP]
> Guidelines for writing a good analysis, and what the reviewer will look for:
>
> - Show your work: Include steps and tools for each conclusion.
> - Completeness of risks considered.
> - Include both implementation and operational failure modes
> - Provide references to support the reviewer.
> - The size of the document will likely be proportional to the project's complexity.
> - The ultimate goal of this document is to identify action items to improve the security of the  project. The FMA review process can be accelerated by proactively identifying action items during the writing process.

## Introduction

This document covers the initial deployment of [the Asterisc fault proof VM](https://github.com/ethereum-optimism/asterisc) with [the Kona fault proof program](https://github.com/anton-rs/kona).

Together, Kona and Asterisc form an alternative proof stack to the [op-program](https://github.com/ethereum-optimism/optimism/tree/develop/op-program) and [cannon](https://github.com/ethereum-optimism/optimism/tree/develop/cannon). A secondary dispute game will be created that uses kona and asterisc, with the `op-challenger` supporting playing _both_ game types - `op-program` + `cannon` as well as `kona` + `asterisc`.

Below are references for this project:

- ["Stage 1.4" internal project document](https://www.notion.so/oplabs/Stage-1-4-Partway-a6f57ad777b148dda01488f2646cff17)
- [Kona Documentation](https://anton-rs.github.io/kona/).
- [Kona Repository](https://github.com/anton-rs/kona).
- [Asterisc Repository](https://github.com/ethereum-optimism/asterisc)


## Failure Modes and Recovery Paths


### Breaking Backwards Compatibility for the op-challenger

- **Description:** The `op-challenger` has been updated to support the Kona + Asterisc Fault Dispute Game type ([Game Type 3](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/types/types.go#L30)).

- **Risk Assessment:** Low likelihood as the action tests that are run in CI very frequently run the `op-challenger` with the kona + asterisc fault dispute game. Medium impact since the off-chain `op-challenger` service can be updated with enough buffer time due to the large game time and nature of the chess clock.

- **Mitigations:** The `op-challenger` is architected in such a way as to isolate game types and moreso the inididual game players. To allow the challenger to scale and play many different game types at once, a list of game types to play is passed into the `op-challenger` as a CLI flag. When games are detected on-chain, the `op-challenger` will create a "game player" for this game type. Players are run in individual threads so a breaking kona-asterisc game player in the challenger will not cause a liveliness issue for the `op-challenger` to play other game types.

- **Detection:** The Dispute Monitor ([`op-dispute-mon`](https://github.com/ethereum-optimism/optimism/tree/develop/op-dispute-mon)) service provides an isolated monitoring service that will detect if a game is not being played by the honest `op-challenger`. It will alert in this case.

- **Recovery Path(s):** If the `op-challenger` fails to play Kona + Asterisc fault dispute games, a fix will need to be made to the `op-challenger`. It does not require governance, but would need a new release of the `op-challenger` to be published and rolled out to infrastructure.


### [Name of Failure Mode 2]

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
- **Detection:** *How do we detect if this occurs?*
- **Recovery Path(s)**: *How do we resolve this? Is it a simple, quick recovery or a big effort? Would recovery require a governance vote or a hard fork?*

### Generic items we need to take into account:

- [X] [./failure-modes-analysis.md](./failure-modes-analysis.md).


## Audit Requirements

*Given the failure modes and action items, will this project require an audit? See [OP Labs Audit Framework: When to get external security review and how to prepare for it](https://gov.optimism.io/t/op-labs-audit-framework-when-to-get-external-security-review-and-how-to-prepare-for-it/6864) for a reference decision making framework. Please explain your reasoning.*

## Appendix

### Appendix A: op-challenger

[op-challenger](https://github.com/ethereum-optimism/optimism/tree/develop/op-challenger) contains CLI flags to demonstrate specifying game types it should play.
