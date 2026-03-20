# [ZK Dispute Game]: Failure Modes and Recovery Path Analysis

**Table of Contents**

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
    - [FM1: ZK Verifier Soundness or Completeness Break](#fm1-zk-verifier-soundness-or-completeness-break)
    - [FM2: Unchallenged Fraudulent Proposal](#fm2-unchallenged-fraudulent-proposal)
    - [FM3: Missed Child Blacklist After Parent Invalidation](#fm3-missed-child-blacklist-after-parent-invalidation)
    - [FM4: Bond Accounting Failure and DelayedWETH Insolvency](#fm4-bond-accounting-failure-and-delayedweth-insolvency)
    - [FM5: Prestate Mismatch](#fm5-prestate-mismatch)
    - [FM6: CWIA Game Args Encoding Mismatch](#fm6-cwia-game-args-encoding-mismatch)
    - [FM7: Bond and Duration Misconfiguration](#fm7-bond-and-duration-misconfiguration)
    - [FM8: Self-Challenge Front-Running to Recover Bonds](#fm8-self-challenge-front-running-to-recover-bonds)
    - [FM9: Challenge Griefing](#fm9-challenge-griefing)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

|  |  |
| --- | --- |
| Author | Wonderland |
| Created at | 2026-03-12 |
| Need Approval From |  |
| Status | Draft |

## Introduction

This document covers the `ZKDisputeGame`, a single-round ZK-proof-based dispute game for the OP Stack. It replaces the multi-round bisection protocol with a single cryptographic proof that verifies an L2 state transition from a parent output root to a claimed output root. The system integrates into the existing OP Stack dispute infrastructure: `DisputeGameFactory`, `AnchorStateRegistry`, `DelayedWETH`, and `OPContractsManager`.

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/zk-dispute-game.md)
- [ZK Dispute Game Spec](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/zk-dispute-game.md)
- [Game Mechanics Spec](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/game-mechanics.md)
- [IZKVerifier Interface Spec](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/zk-interface.md)
- [ZK Fault Proof VM Spec](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/zk-fault-proof-vm.md)

## Failure Modes and Recovery Paths

### FM1: ZK Verifier Soundness or Completeness Break

- **Description:** Three sub-cases of ZK proving system malfunction (aZKG-001):

    **A -- Verifier soundness break:** A bug in the verifier or the `IZKVerifier` adapter causes `verify()` to return without reverting for an incorrect state transition. Since `prove()` treats any non-reverting return as proof acceptance, the game transitions to `DEFENDER_WINS` with a fraudulent `rootClaim`, enabling withdrawal of funds that don't exist on L2. This covers both cryptographic flaws (e.g., broken field arithmetic, pairing checks) and input validation gaps (e.g., `verify()` silently returns on empty proof bytes, zero `programId`, or gas-limited subcalls instead of reverting).

    **B -- Completeness break:** The verifier rejects valid proofs. All challenged games resolve as `CHALLENGER_WINS`, proposers lose bonds, and withdrawals stall.

    **C -- ZK program soundness break:** A bug in the ZK program allows a prover to generate a cryptographically valid proof for an invalid state transition by manipulating private witness values. Unlike sub-case A, the verifier and adapter work correctly — the proof is genuinely valid — but the program's constraint logic fails to properly constrain the relationship between public inputs and private witnesses. The impact is identical to sub-case A: `DEFENDER_WINS` with a fraudulent `rootClaim`.
    
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. `IZKVerifier` interface enables verifier swap without redeploying the game.
    2. `prove()` constructs `publicValues` from immutable on-chain state — only `_proofBytes` is caller-supplied.
    3. Verifier contract will be audited before mainnet deployment.
    4. The ZK program validates that private prover inputs are derived directly from or cryptographically linked to the public inputs.
    5. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` airgap + `DelayedWETH` delay give the Guardian two windows to intervene.
- **Detection:**
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly, covering all three sub-cases: an invalid claim that gets proven or goes unchallenged (A, C), and a valid claim whose proof is never submitted (B).
    - Fuzz testing verifier with edge-case inputs (empty bytes, zero values, gas-limited calls).
- **Recovery Path(s):**
    1. Guardian pauses system and blacklists affected games. If the issue is systemic, the Guardian updates the retirement timestamp to invalidate all in-flight games.
    2. OPCM deploys patched verifier and updates `gameArgs`.
    3. For sub-case B: Guardian blacklists affected games (REFUND mode) while corrected verifier is deployed.
- **Action Item(s):**
    - [ ]  FM1: Fuzz test `IZKVerifier.verify()` with malformed inputs to confirm it always reverts.

---

### FM2: Unchallenged Fraudulent Proposal

- **Description:** If nobody challenges a fraudulent output root within the challenge window, the game resolves as `DEFENDER_WINS` by default. The fraudulent root becomes eligible to finalize withdrawals after the finality delay and the `DelayedWETH` delay elapse. The system's safety depends on at least one honest, always-online challenger. The ZK game requires only a single `challenge()` call (simpler than multi-round bisection), but every proposal's `rootClaim` must still be compared against the canonical output root from a trusted node.
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond economics incentivize challengers: a successful challenge nets the challenger the proposer's `initBond` as profit, so `initBond` must be high enough to justify running challenger infrastructure.
    2. `maxChallengeDuration` provides a configurable time window. It should be long enough to account for L1 congestion and censorship scenarios.
    3. The Guardian can blacklist fraudulent games during the `DISPUTE_GAME_FINALITY_DELAY_SECONDS` airgap, even if the challenge window was missed.
    4. `DelayedWETH` provides an additional freeze window after `closeGame()`.
    5. Multiple independent challengers can run concurrently for redundancy (though only one challenge per game is accepted — see FM8).
- **Detection:**
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly.
- **Recovery Path(s):**
    1. If the challenge window is missed, the Guardian blacklists the game before the finality delay expires.
    2. If the finality delay has also passed, the Guardian pauses the system to prevent withdrawal finalization.
    3. As a last resort, governance can upgrade contracts to recover.
- **Action Item(s):**
    - [ ]  FM2: Ensure `maxChallengeDuration` accounts for L1 congestion/censorship and bond economics incentivize challengers.

---

### FM3: Missed Child Blacklist After Parent Invalidation

- **Description:** Blacklisting a game does not automatically propagate to its children. When a parent game is blacklisted, the Guardian should also blacklist any descendants with invalid claims (aZKG-003). If the Guardian misses a child with an invalid claim, that child could resolve as `DEFENDER_WINS` and finalize fraudulent withdrawals.

    **Safeguards:** iZKG-005 propagates `CHALLENGER_WINS` automatically. iZKG-009 prevents children from resolving before parents. `isGameRespected`/`wasRespectedGameTypeWhenCreated` prevents retired game types from finalizing withdrawals. But none of these help when a parent is individually blacklisted after resolving as `DEFENDER_WINS`.

- **Risk Assessment:** High severity / Low likelihood
- **Mitigations:**
    1. `resolve()` propagates `CHALLENGER_WINS` from parent to child (iZKG-005) — automatic for resolution-based invalidity.
    2. iZKG-009 enforces child-waits-for-parent ordering, giving the Guardian time to act on parents first.
    3. `isGameClaimValid()` checks blacklist/retirement status before allowing withdrawal finalization.
    4. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` + `DelayedWETH` delay provide Guardian intervention windows.
- **Detection:**
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly. A child with an invalid claim that the Guardian missed will be flagged regardless of its parent's blacklist status.
- **Recovery Path(s):**
    1. Guardian calls `updateRetirementTimestamp()` to retire all games. This sets the retirement timestamp to the current block, invalidating all in-flight games.

---

### FM4: Bond Accounting Failure and DelayedWETH Insolvency

- **Description:** Multiple `ZKDisputeGame` instances share the same per-chain `DelayedWETH`. The spec requires (iZKG-011) that for every game: `sum(distributions) + sum(burns) == initBond + challengerBond`. An accounting bug in `closeGame()` that violates this invariant — distributing more than deposited, double-crediting, or mishandling the burn path — could drain `DelayedWETH` or permanently lock funds.
    
    The burn path deserves attention: when a parent resolves as `CHALLENGER_WINS` and the child was unchallenged, the child's `initBond` is sent to `address(0)`. A bug here could create unbacked withdrawals or lock residual funds. The bond distribution table has 10 NORMAL + 3 REFUND scenarios, each with distinct logic paths.
    
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond distribution is fully specified in the [game mechanics](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/game-mechanics.md#bond-distribution) with a clear table of every scenario.
    2. The `DelayedWETH` pattern is shared with `FaultDisputeGame` and has been audited in that context.
    3. REFUND mode returns each bond to its original depositor, which is always balanced by definition.
    4. Bond distribution is deterministic — no external inputs or oracle dependencies.
- **Detection:**
    - Invariant tests asserting `sum(distributions) + sum(burns) == initBond + challengerBond` across all scenarios.
    - Fuzz testing with randomized game sequences to detect conservation violations.
    - Monitoring `DelayedWETH` balance against total outstanding credits.
- **Recovery Path(s):**
    1. Guardian pauses the system and blacklists affected games (REFUND mode).
    2. If `DelayedWETH` is insolvent, governance must deploy a replacement — the contract is immutable.
- **Action Item(s):**
    - [ ]  FM4: Implement iZKG-011 conservation invariant tests across all bond distribution scenarios.
    - [ ]  FM4: Fuzz test bond accounting across randomized game lifecycles, including the burn path.

---

### FM5: Prestate Mismatch

- **Description:** The `absolutePrestate` is updated on-chain but provers still run the old binary (or vice versa) (aZKG-002). Proofs fail verification; challenged games time out as `CHALLENGER_WINS` and proposers lose bonds.
- **Risk Assessment:** Medium severity / Low likelihood
- **Mitigations:**
    1. In-progress games use old configuration immutably (CWIA args fixed at clone creation). Off-chain software automatically selects the correct prestate based on the absolute prestate hash, supporting multiple prestates concurrently.
    2. Verifier contracts are immutable — old verifiers remain functional indefinitely and cannot be deprecated.
- **Detection:**
    - Alerts when a challenged game fails to receive a proof within a reasonable time (indicating prover/prestate mismatch).
    - Alerts when a `prove()` transaction for a valid state root reverts.
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly.
- **Recovery Path(s):**
    1. Fix the off-chain software configuration. Proposers lose `initBond` on any challenged games that couldn't be proven, but the system is otherwise unaffected.
    2. Guardian can blacklist affected games to trigger REFUND mode if needed.

---

### FM6: CWIA Game Args Encoding Mismatch

- **Description:** The MCP clone pattern packs 8 fields into `gameArgs` via `abi.encodePacked`. A mismatch between `OPContractsManager._makeGameArgs()` encoding and `ZKDisputeGame` CWIA offset decoding causes the game to read wrong values — wrong `verifier` address, wrong `challengerBond`, wrong durations, etc. Note: `l2SequenceNumber` type changed from `uint256` to `uint64` in `_extraData`, relevant to packing correctness.
- **Risk Assessment:** Medium severity / Low likelihood
- **Mitigations:**
    1. The encoding is defined in the spec ([Game Args Layout](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/zk-dispute-game.md#game-args-layout)) and implemented in `OPContractsManager._makeGameArgs()`.
    2. Round-trip tests that encode via `_makeGameArgs()` and decode via the game's accessor functions verify consistency.
    3. The CWIA pattern is used extensively in the existing OP Stack (e.g., `FaultDisputeGame`), so there is prior art and tooling for validating it.
- **Detection:**
    - Unit tests that round-trip all 8 fields through encode/decode.
    - Integration tests that create a game via the factory and verify all accessor functions return expected values.
    - Invariant/fuzz tests that verify accessor output matches encoding input for random values.
    - Specific tests for the `l2SequenceNumber` `uint64` encoding in `_extraData`.
- **Recovery Path(s):**
    1. If detected before deployment, fix the encoding/decoding and redeploy.
    2. If detected after deployment, OPCM must deploy a new implementation with corrected decoding and update the factory. In-progress games with incorrect decoding would need to be blacklisted.
- **Action Item(s):**
    - [ ]  FM6: Implement comprehensive round-trip encoding/decoding tests for all 8 `gameArgs` fields, including edge-case values (zero, max, addresses with leading zeros).
    - [ ]  FM6: Add fuzz tests that verify `_makeGameArgs()` output is correctly decoded by the game's accessor functions.
    - [ ]  FM6: Review the `Duration` type's packed size and ensure it matches the offset calculations in the CWIA decoding logic.
    - [ ]  FM6: Add specific tests for `l2SequenceNumber` `uint64` encoding in `_extraData` to verify correct packing and offset alignment.

---

### FM7: Bond and Duration Misconfiguration

- **Description:** Economic parameter calibration risks (aZKG-004, aZKG-007). All sub-cases are fundamentally about calibrating `initBond`, `challengerBond`, `maxChallengeDuration`, and `maxProveDuration`:
    - **Bonds too low:** Spam proposals and frivolous challenges are cheap. Challenging costs only `challengerBond`; defending requires expensive ZK proof generation. If proving cost > `challengerBond`, defending is economically irrational.
    - **Bonds too high:** Honest proposers and challengers priced out, undermining permissionless participation.
    - **Durations too short:** Provers may not generate the proof in time. L1 censorship of `prove()` also becomes feasible (cost proportional to `maxProveDuration`). Short `maxChallengeDuration` compounds FM2 risk.
    - **Durations too long:** Withdrawal finality delayed on the unchallenged path. The latency benefit of ZK proofs is only realized if someone eagerly proves, which has its own cost.
    - **Block range selection:** The proposer is responsible for creating games with block ranges they can prove within `maxProveDuration`. Proving larger ranges in a single proof is generally cheaper per-block in zkVMs however a proposer who creates an unprovable range loses their own `initBond`.
    - **L1 gas pressure:** ZK proof verification can cost several hundred thousand gas units depending on the backend. During extreme gas spikes, `prove()` inclusion becomes expensive, not technically unfeasible though given the margin against the block gas limit.
- **Risk Assessment:** Medium severity / Medium likelihood
- **Mitigations:**
    1. All parameters are in `gameArgs` and tunable per chain by OPCM without redeploying.
    2. `challengerBond` should exceed expected proving cost so defending is always profitable.
    3. Third-party provers can earn `challengerBond`, creating a market incentive for proving services.
    4. `maxProveDuration` must be long enough that L1 censorship of `prove()` is economically infeasible (aZKG-007).
    5. Rational proposers won't create unnecessarily large block ranges (aZKG-006).
- **Detection:**
    - Monitoring challenge rates, proving costs vs `challengerBond`, and `prove()` inclusion failures.
    - Alerts when challenged games approach `maxProveDuration` without proof submission.
- **Recovery Path(s):**
    1. OPCM adjusts `challengerBond` and/or `maxProveDuration` for new games. Existing in-flight games continue to play out with their existing rules.
    2. Guardian can blacklist exploitative games, update the retirement timestamp to invalidate all in-flight games, or change the respected game type.
- **Action Item(s):**
    - [ ]  FM7: Establish a minimum `challengerBond` policy such that `challengerBond > expected proving cost` to ensure defending is always profitable.
    - [ ]  FM7: Calibrate `maxProveDuration` per aZKG-007 with analysis of L1 censorship costs and proof generation time on reference hardware.
    - [ ]  FM7: Monitor challenge rates in production and have a runbook for adjusting bond and duration parameters if griefing is detected.
    - [ ]  FM7: Add gas consumption regression tests for the verifier.

---

### FM8: Self-Challenge Front-Running to Recover Bonds

- **Description:** Only one challenge per game (iZKG-010). A malicious proposer who posts a fraudulent `rootClaim` can monitor the mempool for incoming `challenge()` transactions and front-run them by self-challenging from a different address. The honest challenger's transaction reverts. The proposer then lets the prove deadline expire — the game resolves as `CHALLENGER_WINS` and the proposer's challenger address receives `initBond + challengerBond`. Net cost to the proposer: gas only.
    
    This neutralizes the `initBond` penalty for fraud attempts. The proposer can spam fraudulent proposals hoping one goes unchallenged (compounding FM2), and front-run any challenge that comes in to recover their bond. The fraud itself doesn't succeed (the claim is rejected), but the economic deterrent is eliminated.
    
- **Risk Assessment:** Medium severity / Medium likelihood
- **Mitigations:**
    1. Challengers can use private mempools (e.g., Flashbots Protect) to submit challenges without exposing them, preventing the proposer from front-running.
    2. The fraud still doesn't succeed — the claim resolves as `CHALLENGER_WINS` and cannot finalize withdrawals.
    3. The proposer still pays gas for both `create()` and `challenge()`, plus the capital lockup for `initBond + challengerBond` during the game lifecycle.
- **Detection:**
    - Monitoring for games where the challenger address is linked to the proposer (same EOA, same deployer, funded from the same source).
    - Monitoring for repeated fraudulent proposals from the same proposer that are always self-challenged.
- **Recovery Path(s):**
    1. If systematic self-challenge front-running is detected, increase `initBond` to raise the capital cost of the attack.

---

### FM9: Challenge Griefing

- **Description:** A malicious actor challenges every proposal, forcing the proposer to generate and submit ZK proofs for the entire chain. The cost to the attacker is `challengerBond` per challenge (forfeited to the proposer on successful proof). In practice this likely speeds up withdrawal finality rather than delaying it, since the proposer proves the game immediately instead of waiting for `maxChallengeDuration` to elapse unchallenged. In the worst case, the attacker challenges right before `maxChallengeDuration` expires, adding one proof generation time of delay.
- **Risk Assessment:** Low severity / Medium likelihood
- **Mitigations:**
    1. The attacker forfeits `challengerBond` for every challenge, making sustained griefing expensive.
    2. The proposer profits from each griefing challenge, covering proving costs and then some (assuming `challengerBond > proving cost`).
    3. Challenged games resolve faster since the proposer can prove immediately rather than waiting for the challenge window.
- **Detection:**
    - No specific detection required. Games resolve correctly; the proposer simply proves and collects bonds.
- **Recovery Path(s):**
    1. No action required if `challengerBond` covers proving costs — the proposer profits from the griefing.
    2. If `challengerBond` does not cover proving costs, OPCM increases `challengerBond` to restore profitability.

---

## Audit Requirements

The following contracts require an audit before production deployment:

| Contract | Rationale |
| --- | --- |
| `ZKDisputeGame.sol` (implementation) | Core game logic: state machine, bond accounting, parent validation, proof verification call. |
| `IZKVerifier` adapter | Wraps the proving system verifier. A bug here is equivalent to a verifier soundness break (FM1). |
| ZK program | Defines the constrained state transition proven inside the zkVM. A bug in constraint logic could allow valid proofs for invalid state transitions (FM1). |
| `OPContractsManager` (ZK-related changes) | `_makeGameArgs()` encoding for the ZK dispute game. Packing errors would cause FM6. |
| `DisputeGameFactory` (if modified) | Clone deployment and CWIA injection. Changes to support the ZK dispute game must be reviewed. |
| `AnchorStateRegistry` (if modified) | Changes to support `ZKDisputeGame` lifecycle (e.g., `isGameClaimValid`, `isFinalized()` for the new game type). Correctness of `isFinalized()` is covered in Generic Failure Modes (External Contract Dependencies). |

## Action Items

Below is a consolidated list of all action items from the failure modes above.

| Action Item | Description | Source |
| --- | --- | --- |
| FM1-1 | Fuzz test `IZKVerifier.verify()` with malformed inputs to confirm it always reverts. | [FM1](#fm1-zk-verifier-soundness-or-completeness-break) |
| FM2-1 | Ensure `maxChallengeDuration` accounts for L1 congestion/censorship and bond economics incentivize challengers. | [FM2](#fm2-unchallenged-fraudulent-proposal) |
| FM4-1 | Implement iZKG-011 conservation invariant tests across all bond distribution scenarios. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM4-2 | Fuzz test bond accounting across randomized game lifecycles, including the burn path. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM6-1 | Implement comprehensive round-trip encoding/decoding tests for all 8 `gameArgs` fields, including edge-case values (zero, max, addresses with leading zeros). | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-2 | Add fuzz tests that verify `_makeGameArgs()` output is correctly decoded by the game's accessor functions. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-3 | Review the `Duration` type's packed size and ensure it matches the offset calculations in the CWIA decoding logic. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-4 | Add specific tests for `l2SequenceNumber` `uint64` encoding in `_extraData` to verify correct packing and offset alignment. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM7-1 | Establish a minimum `challengerBond` policy such that `challengerBond > expected proving cost` to ensure defending is always profitable. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-2 | Calibrate `maxProveDuration` per aZKG-007 with analysis of L1 censorship costs and proof generation time on reference hardware. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-3 | Monitor challenge rates in production and have a runbook for adjusting bond and duration parameters if griefing is detected. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-4 | Add gas consumption regression tests for the verifier. | [FM7](#fm7-bond-and-duration-misconfiguration) |