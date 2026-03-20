# [ZK Dispute Game]: Failure Modes and Recovery Path Analysis

**Table of Contents**

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
    - [FM1: ZK Verifier Soundness or Completeness Break](#fm1-zk-verifier-soundness-or-completeness-break)
    - [FM2: Unchallenged Fraudulent Proposal](#fm2-unchallenged-fraudulent-proposal)
    - [FM3: Parent Chain Invalidation and Propagation Failure](#fm3-parent-chain-invalidation-and-propagation-failure)
    - [FM4: Bond Accounting Failure and DelayedWETH Insolvency](#fm4-bond-accounting-failure-and-delayedweth-insolvency)
    - [FM5: Upgrade Lifecycle Mismatch](#fm5-upgrade-lifecycle-mismatch)
    - [FM6: CWIA Game Args Encoding Mismatch](#fm6-cwia-game-args-encoding-mismatch)
    - [FM7: Bond and Duration Misconfiguration](#fm7-bond-and-duration-misconfiguration)
    - [FM8: Self-Challenge Front-Running to Recover Bonds](#fm8-self-challenge-front-running-to-recover-bonds)
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

- **Description:** Two sub-cases of `IZKVerifier` malfunction (aZKG-001):
    
    **A -- Soundness break:** A bug in the verifier or the `IZKVerifier` adapter causes `verify()` to return without reverting for an incorrect state transition. Since `prove()` treats any non-reverting return as proof acceptance, the game transitions to `DEFENDER_WINS` with a fraudulent `rootClaim`, enabling withdrawal of funds that don't exist on L2. This covers both cryptographic flaws (e.g., broken field arithmetic, pairing checks) and input validation gaps (e.g., `verify()` silently returns on empty proof bytes, zero `programId`, or gas-limited subcalls instead of reverting).
    
    **B -- Completeness break:** The verifier rejects valid proofs. All challenged games resolve as `CHALLENGER_WINS`, proposers lose bonds, and withdrawals stall.
    
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. `IZKVerifier` interface enables verifier swap without redeploying the game.
    2. `prove()` constructs `publicValues` from immutable on-chain state — only `_proofBytes` is caller-supplied.
    3. Verifier contract will be audited before mainnet deployment.
    4. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` airgap + `DelayedWETH` delay give the Guardian two windows to intervene.
- **Detection:**
    - Off-chain monitoring that re-executes L2 state transitions and alerts on `DEFENDER_WINS` for incorrect claims.
    - Fuzz testing verifier with edge-case inputs (empty bytes, zero values, gas-limited calls).
    - Monitoring for challenged games that never receive a valid proof (sub-case B indicator).
- **Recovery Path(s):**
    1. Guardian pauses system, blacklists affected games, retires game type updating the retirement timestamp if systemic.
    2. OPCM deploys patched verifier and updates `gameArgs`.
    3. For sub-case B: Guardian blacklists affected games (REFUND mode) while corrected verifier is deployed.
- **Action Item(s):**
    - [ ]  FM1: Off-chain monitoring that verifies every game's `rootClaim` against trusted L2 node.
    - [ ]  FM1: Guardian runbook for verifier compromise (pause/blacklist/retire sequence).
    - [ ]  FM1: Fuzz test `IZKVerifier.verify()` with malformed inputs to confirm it always reverts.

---

### FM2: Unchallenged Fraudulent Proposal

- **Description:** If nobody challenges a fraudulent output root within `maxChallengeDuration`, the game resolves as `DEFENDER_WINS` by default. The fraudulent root becomes a Valid Claim eligible to finalize withdrawals. The system's safety depends on at least one honest, well-funded, always-online challenger. The ZK game requires only a single `challenge()` call (simpler than multi-round bisection), but every proposal must still be independently re-executed to verify its `rootClaim`.
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond economics incentivizes challengers, a successful challenge can earn `initBond + challengerBond`, so `initBond` must be high enough to justify the cost of running a challenger infrastructure. At the same time, `challengerBond` must remain above proving cost but low enough to not discourage participation.
    2. `maxChallengeDuration` provides a configurable time window. It should be long enough to account for L1 congestion and censorship scenarios.
    3. The Guardian can blacklist fraudulent games during the `DISPUTE_GAME_FINALITY_DELAY_SECONDS` airgap, even if the challenge window was missed.
    4. `DelayedWETH` provides an additional freeze window after `closeGame()`.
    5. Multiple independent challengers can run concurrently for redundancy (though only one challenge per game is accepted).
- **Detection:**
    - Off-chain challenger software (`op-challenger` or equivalent) that monitors all new games, re-derives the correct output root, and challenges any incorrect claims.
    - Independent monitoring that alerts when a game's `rootClaim` does not match the correct output root and no challenge has been submitted.
    - Alerts when `maxChallengeDuration` is approaching for an unchallenged game with an incorrect claim.
- **Recovery Path(s):**
    1. If the challenge window is missed, the Guardian blacklists the game before `closeGame()`.
    2. If `closeGame()` was already called, the Guardian pauses the system to prevent withdrawal finalization.
    3. As a last resort, governance can upgrade contracts to recover.
- **Action Item(s):**
    - [ ]  FM2: Implement monitoring that alerts when an unchallenged game has an incorrect `rootClaim` with time remaining in the challenge window.
    - [ ]  FM2: Ensure `maxChallengeDuration` accounts for L1 congestion/censorship and bond economics incentivize challengers.
    - [ ]  FM2: Document the Guardian fallback procedure for games that were not challenged in time.

---

### FM3: Parent Chain Invalidation and Propagation Failure

- **Description:** When a parent game is blacklisted or retired after children exist, the Guardian MUST individually blacklist each descendant (aZKG-003). Blacklisting does not change a parent's resolution status — a blacklisted parent that already resolved as `DEFENDER_WINS` does not propagate invalidity to children via `resolve()`. Chain depth is unbounded.
    
    **Safeguards:** iZKG-005 propagates `CHALLENGER_WINS` automatically. iZKG-009 prevents children from resolving before parents. `isGameRespected`/`wasRespectedGameTypeWhenCreated` prevents retired game types from finalizing withdrawals. But none of these help when a parent is individually blacklisted after resolving as `DEFENDER_WINS`.
    
    **Cross-prestate amplification:** Parents don't need the same `absolutePrestate` as children (intentional — avoids 7-day re-proofs). This means retirement of old-prestate games doesn't cover children with different prestates.
    
    **Resolution ordering deadlock:** iZKG-009 requires the parent to be resolved before any child can resolve. Challenging a root game blocks all descendants until the root's proof is submitted or `maxProveDuration` expires. In practice the delay equals the proof generation time (the prover can submit immediately after challenge), not the full `maxProveDuration` — the worst case only applies if proving infrastructure is unavailable. Cost to the attacker is `challengerBond` per challenge (forfeited to the prover). Proposers mitigate by maintaining parallel chains from the anchor state.
    
- **Risk Assessment:** High severity / Medium likelihood
- **Mitigations:**
    1. `resolve()` propagates `CHALLENGER_WINS` from parent to child (iZKG-005) — automatic for resolution-based invalidity.
    2. iZKG-009 enforces child-waits-for-parent ordering, giving the Guardian time to act on parents first.
    3. `isGameClaimValid()` checks blacklist/retirement status before allowing withdrawal finalization.
    4. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` + `DelayedWETH` delay provide Guardian intervention windows.
    5. Proposers can maintain parallel chains to mitigate resolution ordering deadlocks.
- **Detection:**
    - Off-chain tooling that monitors blacklist events and automatically enumerates all descendant games of a blacklisted game by traversing `parentIndex` references, including across prestate boundaries.
    - Alerts when a game whose ancestor is blacklisted reaches a state where `closeGame()` could be called.
    - Monitoring for games that are challenged but remain unresolved close to `maxProveDuration`.
    - Alerts when descendant games are blocked on parent resolution for extended periods.
    - Tracking the depth and structure of game chains.
- **Recovery Path(s):**
    1. Guardian blacklists all identified descendant games individually, tracing across prestate boundaries.
    2. If some descendants were missed and `closeGame()` was called, Guardian pauses the system to prevent further damage.
    3. Guardian retires the entire game type as a last resort to put all games into REFUND mode.
    4. For resolution ordering deadlocks: provers submit proofs for challenged games to unblock the chain. Proposers create new parallel chains from the anchor state to bypass blocked chains.
    5. If deadlock griefing is systematic, OPCM can increase `challengerBond` to raise the cost.
- **Action Item(s):**
    - [ ]  FM3: Build tooling that, given a blacklisted game, automatically enumerates all descendant games by querying `DisputeGameFactory` for all `ZKDisputeGame` instances and filtering by `parentIndex`, including cross-prestate ancestry.
    - [ ]  FM3: Document the Guardian runbook for cascading blacklists, including cross-prestate tracing, tooling, and verification steps.
    - [ ]  FM3: Add monitoring that alerts when any game with a blacklisted ancestor approaches the finality delay window.
    - [ ]  FM3: Analyze the expected chain depth under normal operation and the maximum delay an attacker can impose per `challengerBond` spent via resolution ordering deadlock.

---

### FM4: Bond Accounting Failure and DelayedWETH Insolvency

- **Description:** Multiple `ZKDisputeGame` instances share the same per-chain `DelayedWETH`.
The spec requires (iZKG-011) that for every game: `sum(distributions) + sum(burns) == initBond + challengerBond`. An accounting bug in `closeGame()` that violates this invariant — distributing more than deposited, double-crediting, or mishandling the burn path — could drain `DelayedWETH` or permanently lock funds.
    
    The burn path deserves attention: when a parent resolves as `CHALLENGER_WINS` and the child was unchallenged, the child's `initBond` is sent to `address(0)`. A bug here could create unbacked withdrawals or lock residual funds. The bond distribution table has 10 NORMAL + 3 REFUND scenarios, each with distinct logic paths.
    
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond distribution is fully specified in the [game mechanics](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/game-mechanics.md#bond-distribution) with a clear table of every scenario.
    2. The `DelayedWETH` pattern is shared with `FaultDisputeGame` and has been audited in that context.
    3. REFUND mode returns each bond to its original depositor, which is always balanced by
    definition.
    4. Bond distribution is deterministic — no external inputs or oracle dependencies.
- **Detection:**
    - Invariant tests asserting `sum(distributions) + sum(burns) == initBond + challengerBond` across all scenarios.
    - Fuzz testing with randomized game sequences to detect conservation violations.
    - Monitoring `DelayedWETH` balance against total outstanding credits.
- **Recovery Path(s):**
    1. Guardian pauses the system and blacklists affected games (REFUND mode).
    2. If `DelayedWETH` is insolvent, governance must deploy a replacement — the contract is
    immutable.
- **Action Item(s):**
    - [ ]  FM4: Implement iZKG-011 conservation invariant tests across all bond distribution
    scenarios.
    - [ ]  FM4: Fuzz test bond accounting across randomized game lifecycles, including the burn
    path.
    - [ ]  FM4: Add `DelayedWETH` balance monitoring alert.

---

### FM5: Upgrade Lifecycle Mismatch

- **Description:** Covers the full upgrade lifecycle (aZKG-002, [IZKVerifier upgrade path](https://github.com/defi-wonderland/specs/blob/feat/op-zk-proofs/specs/fault-proof/stage-one/zk/zk-interface.md#verifier-upgrade-path)):
    
    **A -- absolutePrestate mismatch:** The `absolutePrestate` is updated but provers still run the old binary (or vice versa). Proofs fail verification; challenged games time out as `CHALLENGER_WINS` and proposers lose bonds. Also applies when governance delays mean the prestate isn't updated before a hard fork activates on L2.
    
    **B -- Verifier upgrade race:** The old verifier is deprecated while in-flight games still reference it via immutable CWIA args. Those games become unprovable.
    
    **C -- Aggressive retirement timestamp:** `updateRetirementTimestamp()` set too close to present forces in-flight games into REFUND mode unnecessarily.
    
    **D -- Cross-prestate parent chaining:** Parent games intentionally don't need the same `absolutePrestate` as children (avoids 7-day re-proofs on every update). But retiring old-prestate games doesn't cover children that chained off them with a different prestate — the Guardian must trace across prestate boundaries.
    
- **Risk Assessment:** High severity / Low-to-medium likelihood
- **Mitigations:**
    1. In-progress games use old configuration immutably (CWIA args fixed at clone creation).
    2. Upgrade sequencing: update `absolutePrestate` on-chain first, then distribute new binary to provers, then resume proposing. For hard forks, prestate update MUST precede activation.
    3. Old verifier contracts MUST remain functional until all in-flight games resolve or are blacklisted.
    4. Retirement timestamps should allow at least `maxChallengeDuration + maxProveDuration + DISPUTE_GAME_FINALITY_DELAY_SECONDS` lead time.
- **Detection:**
    - Monitoring for `gameArgs` changes in the `DisputeGameFactory`.
    - Alerts when a challenged game fails to receive a proof within a reasonable time (indicating prover/prestate mismatch).
    - Pre-deployment verification that the `absolutePrestate` matches the program binary hash.
    - Monitoring for games referencing verifier addresses that are no longer functional.
    - Alerts when retirement timestamp is set within the in-flight game window.
- **Recovery Path(s):**
    1. If games were created with a mismatched `absolutePrestate`, they cannot be proven. If challenged, the proposer loses `initBond`. Guardian can blacklist these games to trigger REFUND mode instead.
    2. Correct the `absolutePrestate` via OPCM and resume normal operation.
    3. If a hard fork activated before the prestate update, Guardian retires the old game type and OPCM deploys with the correct prestate.
    4. If the old verifier was made non-functional prematurely, Guardian blacklists affected games to trigger REFUND mode, then OPCM re-deploys the old verifier or migrates to a new one.
- **Action Item(s):**
    - [ ]  FM5: Document a program upgrade runbook that specifies the exact sequencing of `absolutePrestate` update, prover binary distribution, and proposer resumption.
    - [ ]  FM5: For hard fork upgrades, the `absolutePrestate` update MUST be included in the governance proposal alongside the L2 activation timestamp, with the on-chain update applied before activation.
    - [ ]  FM5: Add automated verification in the deployment pipeline that the `absolutePrestate` in `gameArgs` matches the hash of the deployed program binary.
    - [ ]  FM5: Document the requirement that old verifier contracts must remain functional until all in-flight games resolve, and add monitoring for verifier liveness.
    - [ ]  FM5: Establish a minimum retirement timestamp lead time policy and enforce it in Guardian tooling.
    - [ ]  FM5: Document the cross-prestate parent chaining implications for retirement cascades in the Guardian runbook.

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
    - **Durations too long:** Withdrawal finality delayed, eliminating the latency benefit of ZK proofs.
    - **Block range selection:** The proposer is responsible for creating games with block ranges they can prove within `maxProveDuration`. Proving larger ranges in a single proof is generally cheaper per-block in zkVMs however a proposer who creates an unprovable range loses their own `initBond`.
    - **L1 gas pressure:** ZK proof verification can cost several hundred thousand gas units depending on the backend. During extreme gas spikes, `prove()` inclusion becomes expensive, not technically unfeasible though given the margin against the block gas limit.
- **Risk Assessment:** Medium severity / Medium likelihood
- **Mitigations:**
    1. All parameters are in `gameArgs` and tunable per chain by OPCM without redeploying.
    2. `challengerBond` should exceed 2x expected proving cost so defending is always profitable.
    3. Third-party provers can earn `challengerBond`, creating a market incentive for proving services.
    4. `maxProveDuration` must be long enough that L1 censorship of `prove()` is economically infeasible (aZKG-007).
    5. Rational proposers won't create unnecessarily large block ranges (aZKG-006).
- **Detection:**
    - Monitoring challenge rates, proving costs vs `challengerBond`, and `prove()` inclusion failures.
    - Alerts when challenged games approach `maxProveDuration` without proof submission.
- **Recovery Path(s):**
    1. OPCM adjusts `challengerBond` and/or `maxProveDuration`. "Game type changed mid-play" ensures in-flight games get REFUND treatment.
    2. Guardian can blacklist exploitative games or retire the game type.
- **Action Item(s):**
    - [ ]  FM7: Establish a minimum `challengerBond` policy such that `challengerBond > 2x expected proving cost` to ensure defending is always profitable.
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
- **Action Item(s):**
    - [ ]  FM8: Analyze the economic viability of self-challenge front-running given gas costs and capital lockup.
    - [ ]  FM8: Evaluate protocol mitigations: multiple challenges or challenger priority mechanisms.

---

## Audit Requirements

The following contracts require an audit before production deployment:

| Contract | Rationale |
| --- | --- |
| `ZKDisputeGame.sol` (implementation) | Core game logic: state machine, bond accounting, parent validation, proof verification call. |
| `IZKVerifier` adapter | Wraps the proving system verifier. A bug here is equivalent to a verifier soundness break (FM1). |
| `OPContractsManager` (ZK-related changes) | `_makeGameArgs()` encoding for the ZK dispute game. Packing errors would cause FM6. |
| `DisputeGameFactory` (if modified) | Clone deployment and CWIA injection. Changes to support the ZK dispute game must be reviewed. |
| `AnchorStateRegistry` (if modified) | Changes to support `ZKDisputeGame` lifecycle (e.g., `isGameClaimValid`, `isFinalized()` for the new game type). Correctness of `isFinalized()` is covered in Generic Failure Modes (External Contract Dependencies). |

## Action Items

Below is a consolidated list of all action items from the failure modes above.

| Action Item | Description | Source |
| --- | --- | --- |
| FM1-1 | Off-chain monitoring that verifies every game's `rootClaim` against trusted L2 node. | [FM1](#fm1-zk-verifier-soundness-or-completeness-break) |
| FM1-2 | Guardian runbook for verifier compromise (pause/blacklist/retire sequence). | [FM1](#fm1-zk-verifier-soundness-or-completeness-break) |
| FM1-3 | Fuzz test `IZKVerifier.verify()` with malformed inputs to confirm it always reverts. | [FM1](#fm1-zk-verifier-soundness-or-completeness-break) |
| FM2-1 | Implement monitoring that alerts when an unchallenged game has an incorrect `rootClaim` with time remaining in the challenge window. | [FM2](#fm2-unchallenged-fraudulent-proposal) |
| FM2-2 | Ensure `maxChallengeDuration` accounts for L1 congestion/censorship and bond economics incentivize challengers. | [FM2](#fm2-unchallenged-fraudulent-proposal) |
| FM2-3 | Document the Guardian fallback procedure for games that were not challenged in time. | [FM2](#fm2-unchallenged-fraudulent-proposal) |
| FM3-1 | Build tooling that, given a blacklisted game, automatically enumerates all descendant games by querying `DisputeGameFactory` for all `ZKDisputeGame` instances and filtering by `parentIndex`, including cross-prestate ancestry. | [FM3](#fm3-parent-chain-invalidation-and-propagation-failure) |
| FM3-2 | Document the Guardian runbook for cascading blacklists, including cross-prestate tracing, tooling, and verification steps. | [FM3](#fm3-parent-chain-invalidation-and-propagation-failure) |
| FM3-3 | Add monitoring that alerts when any game with a blacklisted ancestor approaches the finality delay window. | [FM3](#fm3-parent-chain-invalidation-and-propagation-failure) |
| FM3-4 | Analyze the expected chain depth under normal operation and the maximum delay an attacker can impose per `challengerBond` spent via resolution ordering deadlock. | [FM3](#fm3-parent-chain-invalidation-and-propagation-failure) |
| FM4-1 | Implement iZKG-011 conservation invariant tests across all bond distribution scenarios. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM4-2 | Fuzz test bond accounting across randomized game lifecycles, including the burn path. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM4-3 | Add `DelayedWETH` balance monitoring alert. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM5-1 | Document a program upgrade runbook that specifies the exact sequencing of `absolutePrestate` update, prover binary distribution, and proposer resumption. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM5-2 | For hard fork upgrades, the `absolutePrestate` update MUST be included in the governance proposal alongside the L2 activation timestamp, with the on-chain update applied before activation. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM5-3 | Add automated verification in the deployment pipeline that the `absolutePrestate` in `gameArgs` matches the hash of the deployed program binary. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM5-4 | Document the requirement that old verifier contracts must remain functional until all in-flight games resolve, and add monitoring for verifier liveness. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM5-5 | Establish a minimum retirement timestamp lead time policy and enforce it in Guardian tooling. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM5-6 | Document the cross-prestate parent chaining implications for retirement cascades in the Guardian runbook. | [FM5](#fm5-upgrade-lifecycle-mismatch) |
| FM6-1 | Implement comprehensive round-trip encoding/decoding tests for all 8 `gameArgs` fields, including edge-case values (zero, max, addresses with leading zeros). | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-2 | Add fuzz tests that verify `_makeGameArgs()` output is correctly decoded by the game's accessor functions. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-3 | Review the `Duration` type's packed size and ensure it matches the offset calculations in the CWIA decoding logic. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM6-4 | Add specific tests for `l2SequenceNumber` `uint64` encoding in `_extraData` to verify correct packing and offset alignment. | [FM6](#fm6-cwia-game-args-encoding-mismatch) |
| FM7-1 | Establish a minimum `challengerBond` policy such that `challengerBond > 2x expected proving cost` to ensure defending is always profitable. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-2 | Calibrate `maxProveDuration` per aZKG-007 with analysis of L1 censorship costs and proof generation time on reference hardware. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-3 | Monitor challenge rates in production and have a runbook for adjusting bond and duration parameters if griefing is detected. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-4 | Add gas consumption regression tests for the verifier. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM8-1 | Analyze the economic viability of self-challenge front-running given gas costs and capital lockup. | [FM8](#fm8-self-challenge-front-running-to-recover-bonds) |
| FM8-2 | Evaluate protocol mitigations: multiple challenges or challenger priority mechanisms. | [FM8](#fm8-self-challenge-front-running-to-recover-bonds) |