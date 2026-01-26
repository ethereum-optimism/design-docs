# FMA: OPCMv2

|  |  |
| --- | --- |
| Author | Kelvin Fichter |
| Created at | 2026-01-23 |
| Needs Approval From | maurelian, mds1 |
| Status | Draft |

## Introduction

OPCMv2 is a redesign of the OP Contracts Manager contract that manages deployments and upgrades for OP Stack chains. This FMA covers failure modes specific to the v2 architecture changes.

**Key architectural changes from v1:**

- Consolidates 5 functions into 2: `deploy()` and `upgrade()`
- Dynamic dispute game type support (was hardcoded to 2 types, now arbitrary array)
- Unified initialization pattern - both deploy and upgrade clear `initialized` slot and call `initialize()`
- Load-or-deploy pattern for chain world assembly
- Config overrides for upgrades (only changed values need to be provided)

**References:**

- [Design Doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/opcm-v2.md)
- [Spec](https://github.com/ethereum-optimism/specs/blob/main/specs/experimental/op-contracts-manager.md)

For generic smart contract failure modes, see the [Generic Contract FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md). For hardfork-related failure modes, see the [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md).

- [x]  Check this box to confirm that generic items have been considered and updated if necessary.

## Failure Modes

### FM1: Re-initialization State Corruption

#### Description

OPCMv2 uses a unified initialization pattern where both `deploy()` and `upgrade()` clear the `initialized` storage slot before calling `initialize()`. If the slot is cleared but `initialize()` fails or is called with incorrect parameters, the contract could be left in an uninitialized or partially initialized state. Additionally, if an attacker can front-run the `initialize()` call after the slot is cleared, they could initialize with malicious parameters.

#### Risk Assessment

Critical severity / Low likelihood

#### Mitigations

- The clear-and-initialize sequence must be atomic within a single transaction
- Use `initializer` modifier that prevents re-initialization once complete
- Have functions that use `initialize` always use `onlyProxyAdminOrProxyAdminOwner`

#### Detection

- Some script should check for uninitalized storage slots after OPCM executes

#### Recovery Path

- If contract is left uninitialized: re-call `initialize()` with correct parameters (if still possible)
- Worst case can be critical severity, should never be allowed to happen that `initialize()` is left open and the function can be called by anyone

#### Action Items

- [ ] Check for uninitalized initializer slot after OPCM execution
- [ ] Semgrep or other check that confirms all contracts use `onlyProxyAdminOrProxyAdminOwner`

### FM2: Storage Layout Corruption During Upgrade

#### Description

The re-initialization pattern for upgrades writes to storage slots that may have different meanings between contract versions. If a new version introduces storage variables that conflict with existing slot positions, or if the initialization logic assumes empty slots that contain old data, storage corruption can occur.

#### Risk Assessment

High+ severity / Low likelihood

#### Mitigations

- Careful review of storage layout changes during PRs
- Process-level checks for storage layout changes (e.g., during smart contracts release review)

#### Detection

- Fork testing will catch clearly broken storage layouts
- Challenging to detect if subtle

#### Recovery Path

- If storage layout issue causes broken-but-not-unsafe behavior: requires bug fix release ASAP
- If causes unsafe behavior: potentially critical, maybe pause

#### Action Items

- [ ] Improve review checks for storage layout checks
- [ ] Runbook if issues of this class are detected onchain after upgrade execution

### FM3: Dynamic Game Type Misconfiguration

#### Description

OPCMv2 moves from hardcoded 2 dispute game types to a dynamic array. Some failure modes could arise if OPCM is not careful about how it handles these parameters.

#### Risk Assessment

Medium severity / Low likelihood

#### Mitigations

- Inputs should be validated either in OPCM or via StandardValidator
- Fork testing and acceptance testing

#### Detection

- Acceptance testing for dispute games in betanets
- Existing monitoring for invalid dispute games

#### Recovery Path

- Game types can be added/modified through `DisputeGameFactory.setImplementation()` by owner
- Pause mechanism can handle incorrect dispute game results

#### Action Items

- [ ] Validate that existing StandardValidator checks will be sufficient

### FM4: Load-or-Deploy Pattern Failures

#### Description

The load-or-deploy pattern attempts to load an existing contract and deploys a new one if not found. Creates a new potential failure mode where a new proxy is deployed unintentionally.

#### Risk Assessment

High+ severity / Low likelihood

#### Mitigations

- Refuse to deploy new proxies for upgrades unless the user explicitly requests that creation
- Tie the extra instructions that allow proxy creations to specific OPCM versions so they cannot be accidentally set for future releases

#### Detection

- Superchain Ops should clearly highlight any new proxies that have been created (using event logs)

#### Recovery Path

- Impact depends on contract being created
- If impact is frozen funds, upgrade to fix

#### Action Items

- [ ] Have Superchain Ops highlight new proxies created by OPCMv2 upgrades

### FM5: Config Override Injection

#### Description

Upgrades can override specific config values rather than providing full config. Overrides must be carefully considered so that users cannot override things in a dangerous way.

#### Risk Assessment

High+ severity / Medium likelihood

#### Mitigations

- OPCMv2 limits overrides to specific variables and specific OPCM versions
- Careful review of any overrides supplied for any given upgrade

#### Detection

- Superchain Ops already displays state diffs, LLMs can help review

#### Recovery Path

- Recovery depends on impact
- If impact causes freeze, upgrade, otherwise may be too critical to recover

#### Action Items

- [ ] OPCMv2 Superchain Ops validation instructions should instruct careful review of overrides

### FM6: Gas Limit Exceeded

#### Description

OPCM operations must fit within per-transaction gas limits. The consolidated `deploy()` function deploys multiple contracts atomically; if total gas exceeds limit, deployment fails entirely. Fusaka hardfork gas limits apply. Large dispute game arrays or complex initialization logic could push gas usage over limits. Applies to `upgrade()` too.

#### Risk Assessment

Medium severity / Low likelihood

#### Mitigations

- Unit and upgrade testing for upgrade gas cost

#### Detection

- Betanet testing will catch for the 1 chain case

#### Recovery Path

- Would almost certainly be caught at Betanet stage
- If caught later, requires update to OPCM to fix gas issues

#### Action Items

- [ ] Guarantee we have strong bounds in place for deploy and upgrade functions

### FM7: Missing v1 Functionality

#### Description

OPCMv2 consolidates v1 functions. Edge cases from v1 may not be covered by the new interface.

#### Risk Assessment

Low severity / Medium likelihood

#### Mitigations

- Comprehensive mapping document: v1 function -> v2 equivalent call
- Integration tests that replicate all v1 use cases using v2 interface
- Maintain v1 interface in `op-deployer` for now

#### Detection

- User reports of unsupported operations
- Failed upgrade attempts with specific configurations
- Comparison testing: v1 and v2 output equivalence for same inputs

#### Recovery Path

- Add missing functionality and release a fixed OPCM
- Extend v2 interface to support discovered edge cases

#### Action Items

- [ ] Thorough LLM review of functional differences

### FM8: Upgrade Atomicity Failures

#### Description

A single `upgrade()` call may upgrade multiple contracts together. If something reverts partway through BUT the entire transaction somehow succeeds, some contracts may be upgraded while others are not, leaving the system in an inconsistent state.

#### Risk Assessment

High+ severity / Low likelihood

#### Mitigations

- Auditing and review to clearly guarantee that state transitions are all-or-nothing

#### Detection

- Potentially challenging to detect other than via acceptance testing

#### Recovery Path

- If detected and does not immediately cause a critical issue, can recover via upgrade

## Audit Requirements

**This component requires audit.**
