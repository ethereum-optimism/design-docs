# FMA: Interop Support in OptimismPortal

|---------------------|-------------------------|
| Author              | Kelvin Fichter          |
| Created at          | 2025-04-03              |
| Needs Approval From | Maurelian, Matt Solomon |
| Other Reviewers     |                         |
| Status              | Final                   |

## Introduction

This Failure Mode Analysis discusses changes made to the `OptimismPortal` and a number of
surrounding contracts to support interop between two OP Stack chains.

### AnchorStateRegistry as source of truth

The `OptimismPortal` is updated to use the `AnchorStateRegistry` as the source of truth for the
validity of Dispute Game instances. This means that the `AnchorStateRegistry` now decides whether a
given Dispute Game is valid and holds important state like the retirement timestamp and the
respected game type.

## ETHLockbox for every OptimismPortal

The `OptimismPortal` is updated to use the `ETHLockbox` contract to store ETH. The `ETHLockbox`
supports multiple `OptimismPortal` contracts and has a function that allows an admin account to
allow new portals to use the lockbox and to migrate funds from one lockbox to another. Migration
functions in the `OptimismPortal` and the `ETHLockbox` and the overall migration process defined by
these functions are in scope for this contest.

## OptimismPortal proving function for Super Roots

The `OptimismPortal` is updated to have a new version of `proveWithdrawalTransaction` that uses the
Super Root proving path. The Super Root proving path is toggled by the variable `superRootsActive`
which is meant to be triggered when a chain migrates to using Super Roots.

## Failure Modes

### FM1: Incorrect State Reporting by `AnchorStateRegistry`

#### Description

A bug within the `AnchorStateRegistry` causes it to return incorrect data about a dispute game
(e.g., resolves incorrectly, reports wrong `respectedGameType`, incorrect `retirementTimestamp`).
Since the `OptimismPortal` now relies entirely on the `AnchorStateRegistry` as the source of truth
for game validity and state, incorrect data leads directly to `OptimismPortal` malfunction.

#### Risk Assessment

- Impact: CRITICAL
  - Reasoning: Could lead to a **Withdrawal Safety Failure** or **Withdrawal Liveness Failure**.
- Likelihood: LOW
  - Reasoning: Assumes the `AnchorStateRegistry` is well-tested and audited as a core component.

#### Mitigations

- Rigorous testing (unit, integration, fuzzing) of `AnchorStateRegistry`.
- Multiple independent audits of `AnchorStateRegistry`.

#### Detection

- Monitoring already exists for the correctness and age of the Anchor State.
- Monitoring already exists for the correctness of resolved dispute games.

#### Action Items

- [x] Update monitoring to take `SystemConfig` or `OptimismPortal` instead of `AnchorStateRegistry`
      or `DisputeGameFactory` as input because these contracts will change, leading to a mismatch
      between the monitor and the actual onchain state.

#### Recovery Path(s)

- Depends heavily on the nature of the bug.
- If possible, upgrade the `AnchorStateRegistry` contract to fix the bug.
- May require manual intervention via governance or admin roles to correct state or pause affected
  portal functions temporarily.
- Possibly not recoverable if Withdrawal Safety is violated.

### FM2: Bug in `ETHLockbox` / `OptimismPortal` Migration Logic

#### Description

The processes for migrating ETH funds from the `OptimismPortal` to the `ETHLockbox` or to migrate
from an old `ETHLockbox` to a new one, coordinated by the `OptimismPortal` and `ETHLockbox`,
involves complex logic. A bug in this logic could leave the system in a state that impacts either
safety or liveness.

#### Risk Assessment

- Impact: HIGH/CRITICAL
  - Reasoning: Failure could result in funds being partially transferred, locked indefinitely in
    either contract, or lead to other inconsistencies between the Portal and the Lockbox,
    potentially causing either a **Loss of Funds** or a **Withdrawal Liveness Failure**.
- Likelihood: LOW
  - Reasoning: Cross-contract interactions for state transitions like migration can be complex, but
    the migration process is well-tested.

#### Mitigations

- Rigorous testing focused on migration scenarios, reusing existing upgrade testing.
- Clear documentation of the expected state after the migration integrated into the superchain-ops
  task that actually executes this migration.

#### Recovery Path(s)

- Depends on the specific bug.
- May require deploying a fixed version of the contracts and performing a subsequent migration.
- Manual fund recovery via governance intervention might be necessary if funds are locked.

#### Action Items

- [x] Ensure that betanet will include testing that (1) a withdrawal after the migration works and
      (2) lockbox balances to confirm that entire balance is transferred properly.
- [x] Follow up with Proofs to understand what's being solved for with a migration that does not
      rely on a trusted input.

### FM3: Disruptions During `superRootsActive` Transition

#### Description

The `OptimismPortal` has a flag `superRootsActive` to switch between legacy and Super Root
withdrawal proving paths. Issues can occur during the transition period when this flag is flipped.
Users might attempt proofs with the wrong mechanism, or bugs could exist in the logic that handles
proofs submitted near the transition time. The system intentionally invalidates old-style proofs
after the switch.

#### Risk Assessment

- Impact: MEDIUM
  - Reasoning: Primarily causes user friction (**Temporary Liveness Failure**) if their proof
    transactions fail due to using the wrong mechanism. Primarily impacts the user experience.
- Likelihood: MEDIUM
  - Reasoning: Transitions are inherently risky periods. User confusion is likely without clear
    communication. Logic handling the boundary needs careful implementation.

#### Mitigations

- Clear, advance communication to users and infrastructure providers about the switch timing.
- Robust testing of proof validation logic specifically around the `superRootsActive` flag change.
- Update and test tools like Viem ahead of time and test that they function properly as part of the
  betanet process.

#### Detection

- Check with partners for reports of failures while proving withdrawals after the switch.

#### Recovery Path(s)

- User education and support to ensure they use the correct proving path.
- Minor bugs might be worked around by tweaking specific proving parameters or pushing updates to
  offchain tooling.

#### Action Items

- [x] ~~Update Viem to auto-switch between proof modes.~~ Tracked separately as part of U17.
- [x] ~~Make sure that devrel/comms is loud about the changes.~~ Tracked separately as part of U17

### FM4: Incorrect Toggling of `superRootsActive`

#### Description

A chain incorrectly toggles the `superRootsActive` flag (e.g., enables it before L2 supports Super
Roots).

#### Risk Assessment

- Impact: HIGH
  - Reasoning: Will likely lead to a **Withdrawal Liveness Failure** as proofs will mismatch the
    required path for the L2 state. Requires intervention to fix.
- Likelihood: LOW
  - Reasoning: Assumes administrative actions are controlled (e.g., multisig, governance) and
    operators are competent. Very unlikely that a migration action would be taken without
    explicitly understanding why the migration exists.

#### Mitigations

- Clear operational procedures and checklists for the transition.
- Pre-flight checks to confirm L2 readiness before toggling the flag.
- Make the function difficult to call accidentally (e.g., require specific confirmation steps).

#### Detection

- Generally hard to detect automatically, would very quickly be surfaced by partners.

#### Recovery Path(s)

- Corrective administrative action to toggle the flag back to the appropriate state.
- Communication to users about the temporary disruption and resolution.
- Investigation into how the incorrect toggling occurred.

#### Action Items

- [x] Add a warning to the `portal.migrate()` function.

### FM5: Further Contract Changes Required for Full Interop

#### Description

Despite the goal of these contract changes being to enable interop support, unforeseen
requirements, design gaps, or edge cases discovered during the development and testing of the full
end-to-end interop system might necessitate additional contract modifications beyond this upgrade.
The current changes might prove insufficient to launch the desired interop functionality.

#### Risk Assessment

- Impact: MEDIUM
  - Reasoning: This failure doesn't directly risk user funds or core system safety/liveness.
    However, it incurs significant operational overhead (design, development, audit, deployment)
    for another upgrade cycle, delays the launch of full interop capabilities, and potentially
    requires revisiting architectural decisions.
- Likelihood: MEDIUM
  - Reasoning: Interop systems are complex, involving interactions between multiple chains and
    components. Discovering unforeseen requirements or integration challenges during the build-out
    phase is relatively common.

#### Mitigations

- Thorough end-to-end interop design and specification review involving all stakeholders *before*
  finalizing and deploying this set of changes.
- Comprehensive integration testing of interop scenarios using these upgraded contracts in
  realistic multi-chain test environments (devnets, testnets).
- Allocating buffer time in the interop project plan for potential follow-up contract adjustments.
- Clear documentation of the *intended scope* of interop functionality enabled by this specific
  upgrade.

#### Detection

- We'll know it when we see it.

#### Recovery Path(s)

- Initiate a new contract development, audit, and deployment cycle to implement the missing or
  corrected functionality.
- Evaluate if a reduced scope for the initial interop launch is feasible, deferring features that
  require the missing contract changes.
- Delay the launch of interop features until the necessary follow-up contract upgrades are completed.

#### Action Items

- [x] Make sure that Proofs is aware of the requirements/goals here.
