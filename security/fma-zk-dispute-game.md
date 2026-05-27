# [ZK Dispute Game]: Failure Modes and Recovery Path Analysis

**Table of Contents**

- [Introduction](#introduction)
- [Failure Modes and Recovery Paths](#failure-modes-and-recovery-paths)
    - [FM1: ZK Verifier Soundness or Completeness Break](#fm1-zk-verifier-soundness-or-completeness-break)
    - [FM2: Unchallenged Fraudulent Proposal](#fm2-unchallenged-fraudulent-proposal)
    - [FM3: Missed Child Blacklist After Parent Invalidation](#fm3-missed-child-blacklist-after-parent-invalidation)
    - [FM4: Bond Accounting Failure and DelayedWETH Insolvency](#fm4-bond-accounting-failure-and-delayedweth-insolvency)
    - [FM5: Prestate Mismatch](#fm5-prestate-mismatch)
    - [FM6: CWIA Game Args Layout and extraData Offset Mismatch](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch)
    - [FM7: Bond and Duration Misconfiguration](#fm7-bond-and-duration-misconfiguration)
    - [FM8: Self-Challenge Front-Running to Recover Bonds](#fm8-self-challenge-front-running-to-recover-bonds)
    - [FM9: Challenge Griefing](#fm9-challenge-griefing)
    - [FM10: Anchor State Type Mismatch on First Super Game](#fm10-anchor-state-type-mismatch-on-first-super-game)
    - [FM11: Partial Chain Monitoring Allows Fraudulent Output Root to Stand](#fm11-partial-chain-monitoring-allows-fraudulent-output-root-to-stand)
    - [FM12: Shared Parameter Drift as Superchain Scales](#fm12-shared-parameter-drift-as-superchain-scales)
- [Audit Requirements](#audit-requirements)
- [Action Items](#action-items)

|  |  |
| --- | --- |
| Author | Wonderland |
| Created at | 2026-05-18 |
| Need Approval From |  |
| Status | Draft |

## Introduction

This document covers the `ZKDisputeGame`, a single-round ZK-proof-based dispute game for the OP Stack adapted for the interop superchain. Instead of committing to a single chain's output root, a proposer posts a super root, a commitment over all chains in the superchain simultaneously, with a bond. Anyone can challenge it by depositing a challenger bond, and a prover submits a ZK proof to defend the claim or lets the proving window expire. The system integrates into the existing OP Stack dispute infrastructure: `DisputeGameFactory`, `AnchorStateRegistry`, `DelayedWETH`, and `OPContractsManager`.

Below are references for this project:

- [Design Doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/zk-dispute-game.md)
- [ZK Dispute Game Spec](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/zk/zk-dispute-game.md)
- [Game Mechanics Spec](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/zk/game-mechanics.md)
- [IZKVerifier Interface Spec](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/zk/zk-interface.md)
- [ZK Fault Proof VM Spec](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/zk-fault-proof-vm.md)

## Failure Modes and Recovery Paths

### FM1: ZK Verifier Soundness or Completeness Break

- **Description:** Three sub-cases of ZK proving system malfunction (aZKG-001):

    **A -- Verifier soundness break:** A bug in the verifier or the `IZKVerifier` adapter causes `verify()` to return without reverting for an incorrect state transition. Since `prove()` treats any non-reverting return as proof acceptance, the game transitions to `DEFENDER_WINS` with a fraudulent `rootClaim`, enabling withdrawal of funds that don't exist on any chain in the superchain. This covers both cryptographic flaws (e.g., broken field arithmetic, pairing checks) and input validation gaps (e.g., `verify()` silently returns on empty proof bytes, zero `programId`, or gas-limited subcalls instead of reverting).

    **B -- Completeness break:** The verifier rejects valid proofs. All challenged games resolve as `CHALLENGER_WINS`, proposers lose bonds, and withdrawals stall.

    **C -- ZK program soundness break:** A bug in the ZK program allows a prover to generate a cryptographically valid proof for an invalid state transition by manipulating private witness values. Because the ZK program covers all chains in the superchain simultaneously, a constraint bug enables fraudulent proofs for the entire super root. Unlike sub-case A, the verifier and adapter work correctly and the proof is genuinely valid, but the program's constraint logic fails to properly constrain the relationship between public inputs and private witnesses. The impact is identical to sub-case A: `DEFENDER_WINS` with a fraudulent `rootClaim`, with blast radius across every committed chain.

- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. `IZKVerifier` interface enables verifier swap without redeploying the game.
    2. `prove()` constructs `publicValues` from immutable on-chain state. The super root hash (keccak256 of `SuperRootProof`) and super root timestamp are derived from `extraData` and on-chain context. Only `_proofBytes` is caller-supplied.
    3. Verifier contract will be audited before mainnet deployment.
    4. The ZK program validates that private prover inputs are derived directly from or cryptographically linked to the public inputs, including the full set of `(chainId, outputRoot)` pairs.
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

- **Description:** If nobody challenges a fraudulent super root within the challenge window, the game resolves as `DEFENDER_WINS` by default. The fraudulent root becomes eligible to finalize withdrawals after the finality delay and the `DelayedWETH` delay elapse. The system's safety depends on at least one honest, always-online challenger. `ZKDisputeGame` requires only a single `challenge()` call, but the challenger must verify every `(chainId, outputRoot)` tuple embedded in the super root against a trusted node for each chain. The monitoring surface scales linearly with superchain size. See FM11 for the sub-variant where a challenger is online but only monitors a subset of chains.
- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond economics incentivize challengers: a successful challenge nets the challenger the proposer's `initBond` as profit, so `initBond` must be high enough to justify running multi-chain challenger infrastructure.
    2. `maxChallengeDuration` provides a configurable time window. It should be long enough to account for L1 congestion, censorship scenarios, and the time required to verify all chains in the superchain.
    3. The Guardian can blacklist fraudulent games during the `DISPUTE_GAME_FINALITY_DELAY_SECONDS` airgap, even if the challenge window was missed.
    4. `DelayedWETH` provides an additional freeze window after `closeGame()`.
    5. Multiple independent challengers can run concurrently for redundancy (though only one challenge per game is accepted, see FM8). Redundancy only holds if each challenger covers the full chain set. Partial-chain monitoring is a sub-variant of this failure mode (see FM11).
- **Detection:**
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly.
- **Recovery Path(s):**
    1. If the challenge window is missed, the Guardian blacklists the game before the finality delay expires.
    2. If the finality delay has also passed, the Guardian pauses the system to prevent withdrawal finalization.
    3. As a last resort, governance can upgrade contracts to recover.
- **Action Item(s):**
    - [ ]  FM2: Ensure `maxChallengeDuration` accounts for L1 congestion/censorship, multi-chain verification time, and that bond economics incentivize challengers at superchain scale.

---

### FM3: Missed Child Blacklist After Parent Invalidation

- **Description:** Blacklisting a game does not automatically propagate to its children. When a parent game is blacklisted, the Guardian should also blacklist any descendants with invalid claims (aZKG-003). If the Guardian misses a child with an invalid claim, that child could resolve as `DEFENDER_WINS` and finalize fraudulent withdrawals.

    **Safeguards:** iZKG-005 propagates `CHALLENGER_WINS` automatically. iZKG-009 prevents children from resolving before parents. `isGameRespected`/`wasRespectedGameTypeWhenCreated` prevents retired game types from finalizing withdrawals. But none of these help when a parent is individually blacklisted after resolving as `DEFENDER_WINS`.

- **Risk Assessment:** Medium severity / Low likelihood
- **Mitigations:**
    1. `resolve()` propagates `CHALLENGER_WINS` from parent to child (iZKG-005), automatic for resolution-based invalidity.
    2. iZKG-009 enforces child-waits-for-parent ordering, giving the Guardian time to act on parents first.
    3. `isGameClaimValid()` checks blacklist/retirement status before allowing withdrawal finalization.
    4. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` + `DelayedWETH` delay provide Guardian intervention windows.
- **Detection:**
    - `op-dispute-mon` already detects games that are forecast to or do resolve incorrectly. A child with an invalid claim that the Guardian missed will be flagged regardless of its parent's blacklist status.
- **Recovery Path(s):**
    1. Guardian calls `updateRetirementTimestamp()` to retire all games. This sets the retirement timestamp to the current block, invalidating all in-flight games.

---

### FM4: Bond Accounting Failure and DelayedWETH Insolvency

- **Description:** Multiple `ZKDisputeGame` instances share the same per-chain `DelayedWETH`. The spec requires (iZKG-011) that for every game: `sum(distributions) + sum(burns) == initBond + challengerBond`. An accounting bug in `closeGame()` that violates this invariant by distributing more than deposited, double-crediting, or mishandling the burn path could drain `DelayedWETH` or permanently lock funds.

    The burn path deserves attention: when a parent resolves as `CHALLENGER_WINS` and the child was unchallenged, the child's `initBond` is sent to `address(0)`. A bug here could create unbacked withdrawals or lock residual funds. The bond distribution table has 10 NORMAL + 3 REFUND scenarios, each with distinct logic paths.

- **Risk Assessment:** Critical severity / Low likelihood
- **Mitigations:**
    1. Bond distribution is fully specified in the [game mechanics](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/zk/game-mechanics.md#bond-distribution) with a clear table of every scenario.
    2. The `DelayedWETH` pattern is shared with `FaultDisputeGame` and has been audited in that context.
    3. REFUND mode returns each bond to its original depositor, which is always balanced by definition.
    4. Bond distribution is deterministic, with no external inputs or oracle dependencies.
- **Detection:**
    - Invariant tests asserting `sum(distributions) + sum(burns) == initBond + challengerBond` across all scenarios.
    - Fuzz testing with randomized game sequences to detect conservation violations.
    - Monitoring `DelayedWETH` balance against total outstanding credits.
- **Recovery Path(s):**
    1. Guardian pauses the system and blacklists affected games (REFUND mode).
    2. If `DelayedWETH` is insolvent, governance must deploy a replacement, since the contract is immutable.
- **Action Item(s):**
    - [ ]  FM4: Implement iZKG-011 conservation invariant tests across all bond distribution scenarios.
    - [ ]  FM4: Fuzz test bond accounting across randomized game lifecycles, including the burn path.

---

### FM5: Prestate Mismatch

- **Description:** `absolutePrestate` is the ZK program verification key (VKey) for all chains in the superchain. Any divergence between the on-chain value and the program actually running breaks proving for every chain simultaneously. Three root causes:

    **A -- Off-chain config lag:** `absolutePrestate` is updated on-chain but provers still run the old binary (or vice versa) (aZKG-002). All challenged games time out as `CHALLENGER_WINS` and proposers across the entire superchain lose bonds.

    **B -- New chain added without program update:** When a new chain joins the superchain, the ZK program must be updated to include that chain's state transition function (STF) before games referencing that `chainId` are created. If not, the program either rejects all proofs for those games (completeness break) or silently validates fraudulent state transitions for the new chain.

    **C -- Existing chain changes its STF without program update:** A hardfork or EVM config change on any superchain member without a corresponding ZK program and `absolutePrestate` update causes proof failures for that chain. There is no on-chain enforcement of this coordination, so an unmanaged STF change is indistinguishable from a fraudulent claim. Chains under a self-managed plan that modify core code without coordinating a program update could break proving for the entire superchain.

- **Risk Assessment:** High severity / Medium likelihood
- **Mitigations:**
    1. In-progress games use old configuration immutably (CWIA args fixed at clone creation). Off-chain software selects the correct prestate by hash, supporting multiple prestates concurrently.
    2. Verifier contracts are immutable, so old verifiers remain functional indefinitely.
    3. Off-chain software updates for all chains must be coordinated simultaneously when the ZK program changes.
    4. Any chain addition or STF change on a superchain member must trigger a coordinated ZK program upgrade and `absolutePrestate` update via OPCM before the change takes effect, as a precondition of the governance process.
    5. OP governance must enforce that only chains with program-incorporated STFs are permitted in the superchain set.
- **Detection:**
    - Alerts when a challenged game fails to receive a proof within a reasonable time.
    - Alerts when the proposer generates a proof found invalid on-chain.
    - `op-dispute-mon` detects games that are forecast to or do resolve incorrectly.
    - Alert when a new `chainId` appears in a super root not covered by the current `absolutePrestate`'s known chain set.
    - Alert when a chain's known hardfork block is reached without a corresponding `absolutePrestate` update.
- **Recovery Path(s):**
    1. For sub-case A: fix the off-chain software configuration. Proposers lose `initBond` on challenged games that couldn't be proven. Guardian can blacklist affected games for REFUND mode.
    2. For sub-case B: block game creation for super roots containing the new `chainId` until the ZK program is updated and a new `absolutePrestate` is deployed via OPCM.
    3. For sub-case C: Guardian blacklists in-flight games for the affected chain. OPCM deploys updated ZK program and `absolutePrestate`. In-flight games that cannot be proven resolve via REFUND mode.
- **Action Item(s):**
    - [ ]  FM5: Add monitoring alert for a new `chainId` in a super root not covered by the current `absolutePrestate` known chain set.
    - [ ]  FM5: Enforce in the chain governance process that any hardfork or STF change triggers a ZK program upgrade and `absolutePrestate` update before the hardfork activates.

---

### FM6: CWIA Game Args Layout and extraData Offset Mismatch

- **Description:** Each game's Clone-With-Immutable-Args (CWIA) payload contains a fixed pre-extraData section followed by a variable-length `extraData` of `9 + n×64 bytes` (1 version byte, 8-byte super root timestamp, n `(chainId:32, outputRoot:32)` pairs), with the implementation argument region appended after. All implementation argument getters (`verifier`, `absolutePrestate`, `anchorStateRegistry`, `weth`, bond amounts, durations) derive their byte offsets at runtime via `_preExtraDataByteCount() + _extraDataByteCount()`. `OPContractsManager._makeGameArgs()` produces the encoded payload, and `initialize()` enforces two invariants on the decoded values: `l2ChainId == 0`, and `l2SequenceNumber` (uint64) is the super root timestamp. A mismatch anywhere in this encode / decode / validate chain — off-by-one offsets, stale hardcoded constants, wrong field order, incorrect type sizes, or missing validation — corrupts every implementation argument simultaneously for all games created from that implementation, or silently accepts malformed games.
- **Risk Assessment:** Medium severity / Low likelihood
- **Mitigations:**
    1. Port `_preExtraDataByteCount()` and `_extraDataByteCount()` directly from `SuperFaultDisputeGame` rather than reimplementing them. Any deviation from the established pattern must be explicitly justified.
    2. The encoding is defined in the spec ([Game Args Layout](https://github.com/ethereum-optimism/specs/blob/main/specs/fault-proof/stage-one/zk/zk-dispute-game.md#game-args-layout)) and implemented in `OPContractsManager._makeGameArgs()`.
    3. Round-trip tests that encode via `_makeGameArgs()` and decode via the game's accessor functions run at n=1, n=2, and n=max supported chain counts.
- **Detection:**
    - Unit tests round-tripping all fields through encode/decode at variable extraData lengths.
    - Fuzz tests verifying accessor output matches encoding input for random values and chain counts.
    - Specific tests asserting `l2ChainId == 0` and correct super root timestamp decoding.
    - Integration test: create a game via the factory and verify all accessor functions return expected values.
- **Recovery Path(s):**
    1. If detected before deployment, fix the encoding, offset helpers, and validation then redeploy.
    2. If detected after deployment, OPCM deploys a new implementation with corrected logic and updates the factory. In-progress games with incorrect decoding or missing validation are blacklisted.
- **Action Item(s):**
    - [ ]  FM6: Port `_preExtraDataByteCount()` and `_extraDataByteCount()` from `SuperFaultDisputeGame` without reimplementation, and document any intentional deviations.
    - [ ]  FM6: Implement round-trip encoding/decoding tests for all `gameArgs` fields at n=1, n=2, and n=max chain counts, including edge-case values (zero, max, addresses with leading zeros).
    - [ ]  FM6: Add fuzz tests verifying `_makeGameArgs()` output is correctly decoded by the game's accessor functions across random chain counts.
    - [ ]  FM6: Review the `Duration` type's packed size and ensure it matches the offset calculations in the dynamic CWIA decoding logic.
    - [ ]  FM6: Add tests asserting `l2ChainId == 0` enforcement and correct super root timestamp `uint64` decoding.

---

### FM7: Bond and Duration Misconfiguration

- **Description:** Economic and temporal parameter calibration risks at initial deployment (aZKG-004, aZKG-007). All parameters (`initBond`, `challengerBond`, `maxChallengeDuration`, `maxProveDuration`) are shared across the entire superchain. A single misconfigured value degrades security for every chain simultaneously. This FM covers the static surface: choosing the right values at deployment and accounting for lifecycle-level pressures that apply to any single game. The sub-cases:
    - **Bonds too low:** Spam proposals and frivolous challenges are cheap. Challenging costs only `challengerBond`, while defending requires expensive ZK proof generation for all chains. If proving cost > `challengerBond`, defending is economically irrational.
    - **Bonds too high:** Honest proposers and challengers priced out, undermining permissionless participation.
    - **Durations too short:** Provers may not generate the proof in time for all chains. L1 censorship of `prove()` also becomes feasible (cost proportional to `maxProveDuration`). Short `maxChallengeDuration` compounds FM2 risk.
    - **Durations too long:** Withdrawal finality delayed on the unchallenged path. The latency benefit of ZK proofs is only realized if someone eagerly proves, which has its own cost.
    - **Super root timestamp selection:** The proposer is responsible for selecting a super root timestamp that all included chains can prove within `maxProveDuration`. Proving cost scales with chain count, not a single block range.
    - **L1 gas pressure:** ZK proof verification can cost several hundred thousand gas units depending on the backend. During extreme gas spikes, `prove()` inclusion becomes expensive, not technically unfeasible given the margin against the block gas limit.

    These values are not set once. FM12 covers the recurring recalibration triggered by changes in the superchain composition over time.

- **Risk Assessment:** Medium severity / Medium likelihood
- **Mitigations:**
    1. All parameters are in `gameArgs` and tunable per deployment by OPCM without redeploying.
    2. `challengerBond` should exceed expected full superchain proving cost so defending is always profitable.
    3. Third-party provers can earn `challengerBond`, creating a market incentive for proving services.
    4. `maxProveDuration` must be long enough that L1 censorship of `prove()` is economically infeasible (aZKG-007).
    5. Rational proposers won't create superroots for unnecessarily large timestamp deltas (aZKG-006).
- **Detection:**
    - Monitoring challenge rates, proving costs vs `challengerBond`, and `prove()` inclusion failures.
    - Alerts when challenged games approach `maxProveDuration` without proof submission.
- **Recovery Path(s):**
    1. OPCM adjusts `challengerBond` and/or `maxProveDuration` for new games. Existing in-flight games continue to play out with their existing rules.
    2. Guardian can blacklist exploitative games, update the retirement timestamp to invalidate all in-flight games, or change the respected game type.
- **Action Item(s):**
    - [ ]  FM7: Establish a minimum `challengerBond` policy such that `challengerBond > expected full superchain proving cost` to ensure defending is always profitable.
    - [ ]  FM7: Calibrate `maxProveDuration` per aZKG-007 with analysis of L1 censorship costs and proof generation time on reference hardware for the full chain set.
    - [ ]  FM7: Monitor challenge rates in production and have a runbook for adjusting bond and duration parameters if griefing is detected.
    - [ ]  FM7: Add gas consumption regression tests for the verifier.

---

### FM8: Self-Challenge Front-Running to Recover Bonds

- **Description:** Only one challenge per game (iZKG-010). A malicious proposer who posts a fraudulent `rootClaim` can monitor the mempool for incoming `challenge()` transactions and front-run them by self-challenging from a different address. The honest challenger's transaction reverts. The proposer then lets the prove deadline expire. The game resolves as `CHALLENGER_WINS` and the proposer's challenger address receives `initBond + challengerBond`. Net cost to the proposer: gas only.

    This neutralizes the `initBond` penalty for fraud attempts. The proposer can spam fraudulent proposals hoping one goes unchallenged (compounding FM2), and front-run any challenge that comes in to recover their bond. The fraud itself doesn't succeed (the claim is rejected), but the economic deterrent is eliminated.

- **Risk Assessment:** Low severity / Low likelihood
- **Mitigations:**
    1. Challengers can use private mempools (e.g., Flashbots Protect) to submit challenges without exposing them, preventing the proposer from front-running.
    2. The fraud still doesn't succeed, as the claim resolves as `CHALLENGER_WINS` and cannot finalize withdrawals.
    3. The proposer still pays gas for both `create()` and `challenge()`, plus the capital lockup for `initBond + challengerBond` during the game lifecycle.
- **Detection:**
    - Monitoring for games where the challenger address is linked to the proposer (same EOA, same deployer, funded from the same source).
    - Monitoring for repeated fraudulent proposals from the same proposer that are always self-challenged.
- **Recovery Path(s):**
    1. If systematic self-challenge front-running is detected, increase `initBond` to raise the capital cost of the attack.

---

### FM9: Challenge Griefing

- **Description:** A malicious actor challenges every proposal, forcing the proposer to generate and submit a ZK proof over all chains on every game. Because `ZKDisputeGame` requires a single proof over all chains simultaneously, the proving cost per challenge is materially higher than in the single-chain game. The cost to the attacker is `challengerBond` per challenge (forfeited to the proposer on successful proof). In practice this likely speeds up withdrawal finality rather than delaying it, since the proposer proves the game immediately instead of waiting for `maxChallengeDuration` to elapse unchallenged. In the worst case, the attacker challenges right before `maxChallengeDuration` expires, adding one full proof generation time of delay.
- **Risk Assessment:** Low severity / Medium likelihood
- **Mitigations:**
    1. The attacker forfeits `challengerBond` for every challenge, making sustained griefing expensive.
    2. The proposer profits from each griefing challenge, covering proving costs and then some (assuming `challengerBond > full superchain proving cost`). The calibration of this invariant is more critical than in the single-chain case.
    3. Challenged games resolve faster since the proposer can prove immediately rather than waiting for the challenge window.
- **Detection:**
    - No specific detection required. Games resolve correctly, and the proposer simply proves and collects bonds.
- **Recovery Path(s):**
    1. No action required if `challengerBond` covers superchain proving costs, since the proposer profits from the griefing.
    2. If `challengerBond` does not cover proving costs, OPCM increases `challengerBond` to restore profitability. If superchain size grows and proving cost rises, OPCM must recalibrate `challengerBond` upward proactively.

---

### FM10: Anchor State Type Mismatch on First Super Game

- **Description:** When migrating from `SuperFaultDisputeGame` or `SuperPermissionedFaultDisputeGame`, the `AnchorStateRegistry` must hold a valid super root anchor rather than a single-chain output root before any `ZKDisputeGame` game is created. The ZK program derives its starting state from this anchor, so a type mismatch either causes every proof to fail or, if the program does not validate the anchor type, silently accepts an invalid starting point, allowing the prover to construct a fraudulent state transition. Note: this is handled by the SuperRoots migration that updates the anchor registry to use a super root as anchor (see [OPCM v2 Migration Mode](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/proofs/super-dispute-game-migration.md#opcm-v2-migration-mode)), but the migration must be executed correctly for this invariant to hold.
- **Risk Assessment:** High severity / Low likelihood
- **Mitigations:**
    1. OPCM migration scripts for Super Games must set the `AnchorStateRegistry` to a valid super root anchor atomically, in the same transaction as or immediately before `setRespectedGameType(ZKDisputeGame)` is called.
- **Detection:**
    - The `AnchorStateRegistry` and the game cannot detect this on `initialize`. After migration, perform an on-chain sanity check that the root returned by the `AnchorStateRegistry` is super root compatible (check the version byte in the decoded `SuperRootProof`).
- **Recovery Path(s):**
    1. Roll back `respectedGameType` to the previous game type.
    2. Re-run OPCM migration with a correctly formatted super root anchor.
- **Action Item(s):**
    - [ ]  FM10: Verify OPCM migration script sets a valid super root anchor atomically before or in the same transaction as `setRespectedGameType(ZKDisputeGame)`.

---

### FM11: Partial Chain Monitoring Allows Fraudulent Output Root to Stand

- **Description:** Unlike FM2 (no challenger at all), this failure mode involves a challenger that is online but only monitors a subset of the superchain. A fraudulent `(chainId, outputRoot)` tuple embedded in an otherwise-valid super root goes undetected, the game resolves `DEFENDER_WINS`, and a fraudulent withdrawal on the unmonitored chain finalizes. The challenger must be updated to parse every `(chainId, outputRoot)` tuple in `extraData`, verify each against a trusted RPC, and derive the SuperRoot hash from all monitored chains to assess the validity of a claim.
- **Risk Assessment:** Medium severity / Low likelihood
- **Mitigations:**
    1. Off-chain challengers must parse every `(chainId, outputRoot)` tuple in `extraData` and verify each against a trusted RPC before deciding on claim validity.
    2. Bond economics from FM2 still apply: `initBond` must cover the cost of running multi-chain verification infrastructure. The bond should generally scale as more chains are added to the superchain.
- **Detection:**
    - Off-chain monitor cross-checks every tuple against trusted nodes before accepting any game as valid. `op-challenger` must be updated to verify all chains in every super root.
- **Recovery Path(s):**
    1. Blacklist the resolved game before the finality delay expires.
    2. Challenge any open games containing the fraudulent tuple via `challenge()`.

---

### FM12: Shared Parameter Drift as Superchain Scales

- **Description:** FM7 covers the initial calibration of the shared parameters. This FM covers the recurring problem: values that were safe at deployment can drift out of range as the superchain composition evolves. Each chain addition, hardfork, or per-chain config change (gas limit, block time, EVM version) is a trigger for revisiting all four values. Three drift dimensions:

    **A -- Proof duration drift:** ZK proof generation time grows with chain count and chain complexity. `maxProveDuration` must cover the worst-case chain at the worst-case superchain size. If any chain is added or upgrades its configuration without a parameter review and OPCM upgrade, in-flight challenged games may expire before a proof can be submitted, resolving as `CHALLENGER_WINS`.

    **B -- Challenge duration drift:** `maxChallengeDuration` must cover the time needed to verify games for every chain in the superchain. A duration calibrated for a two-chain superchain may be inadequate for a ten-chain one.

    **C -- Bond-to-TVL drift:** Two metrics evolve independently of each other and of `initBond` / `challengerBond`. Aggregate superchain Total Value Locked (TVL) grows, raising the required deterrent that `initBond` must provide. Per-chain proving cost grows as chains are added, raising the floor that `challengerBond` must exceed. Bonds that were correct at deployment can become too low (fraud profitable) or too high (honest participants priced out) without anyone changing them.

    Unlike FM7, this FM is process-driven rather than value-driven: the failure is not picking the wrong number at setup, but not picking a new number when the conditions change.

- **Risk Assessment:** Medium severity / High likelihood
- **Mitigations:**
    1. Parameters must be calibrated to the worst-case chain in the current superchain set. Any chain addition or per-chain config change must trigger a full parameter review and OPCM upgrade if the worst case changes.
    2. `initBond` policy must factor in aggregate superchain TVL. As a baseline, `initBond` should exceed the expected gain from a successful fraudulent proposal on the highest-TVL chain, and `challengerBond` must exceed expected proving cost for the full chain set on reference hardware.
- **Detection:**
    - Monitor observed proof time vs `maxProveDuration` and alert when margin falls below 20%.
    - Monitor challenger spin-up and response time vs `maxChallengeDuration` and alert on margin reduction.
    - Alert when any chain in the superchain set updates gas limit, block time, or EVM version, flagging for immediate parameter review.
    - Periodic review of proof time benchmarks and aggregate TVL as superchain composition changes.
- **Recovery Path(s):**
    1. OPCM upgrade with recalibrated `maxProveDuration`, `maxChallengeDuration`, and bond values. Existing in-flight games continue under their original values.
- **Action Item(s):**
    - [ ]  FM12: Establish a parameter review process triggered by any chain addition or per-chain config change (gas limit, block time, EVM version).
    - [ ]  FM12: Calibrate `maxProveDuration` and `maxChallengeDuration` to the worst-case chain in the current superchain set and document the required margin (≥20%).
    - [ ]  FM12: Calibrate `initBond` to aggregate superchain TVL and `challengerBond` to full superchain proving cost on reference hardware, and document the methodology.

---

## Audit Requirements

The following contracts require an audit before production deployment:

| Contract | Rationale |
| --- | --- |
| `ZKDisputeGame.sol` (implementation) | Core game logic: state machine, bond accounting, parent validation, proof verification call, variable-length extraData decoding. |
| `IZKVerifier` adapter | Wraps the proving system verifier. A bug here is equivalent to a verifier soundness break (FM1). |
| ZK program | Defines the constrained state transition proven across all chains in the superchain. A constraint bug enables invalid proofs for the entire super root (FM1-C, FM5). |
| `OPContractsManager` (ZK-related changes) | `_makeGameArgs()` encoding for the ZK dispute game. Packing errors would cause FM6. |
| `DisputeGameFactory` (if modified) | Clone deployment and CWIA injection. Changes to support `ZKDisputeGame` must be reviewed. |
| `AnchorStateRegistry` (if modified) | Changes to support `ZKDisputeGame` lifecycle (e.g., `isGameClaimValid`, `isFinalized()` for the new game type). Correctness of `isFinalized()` is covered in Generic Failure Modes (External Contract Dependencies). |
| `SuperRootProof` decoding | Variable-length extraData parsing and runtime offset derivation. Offset errors corrupt all implementation args (FM6). |

## Action Items

Below is a consolidated list of all action items from the failure modes above.

| Action Item | Description | Source |
| --- | --- | --- |
| FM1-1 | Fuzz test `IZKVerifier.verify()` with malformed inputs to confirm it always reverts. | [FM1](#fm1-zk-verifier-soundness-or-completeness-break) |
| FM2-1 | Ensure `maxChallengeDuration` accounts for L1 congestion/censorship, multi-chain verification time, and that bond economics incentivize challengers at superchain scale. | [FM2](#fm2-unchallenged-fraudulent-proposal) |
| FM4-1 | Implement iZKG-011 conservation invariant tests across all bond distribution scenarios. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM4-2 | Fuzz test bond accounting across randomized game lifecycles, including the burn path. | [FM4](#fm4-bond-accounting-failure-and-delayedweth-insolvency) |
| FM5-1 | Add monitoring alert for a new `chainId` in a super root not covered by the current `absolutePrestate` known chain set. | [FM5](#fm5-prestate-mismatch) |
| FM5-2 | Enforce in the chain governance process that any hardfork or STF change triggers a ZK program upgrade and `absolutePrestate` update before the hardfork activates. | [FM5](#fm5-prestate-mismatch) |
| FM6-1 | Port `_preExtraDataByteCount()` and `_extraDataByteCount()` from `SuperFaultDisputeGame` without reimplementation, and document any intentional deviations. | [FM6](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch) |
| FM6-2 | Implement round-trip encoding/decoding tests for all `gameArgs` fields at n=1, n=2, and n=max chain counts, including edge-case values (zero, max, addresses with leading zeros). | [FM6](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch) |
| FM6-3 | Add fuzz tests verifying `_makeGameArgs()` output is correctly decoded by the game's accessor functions across random chain counts. | [FM6](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch) |
| FM6-4 | Review the `Duration` type's packed size and ensure it matches the offset calculations in the dynamic CWIA decoding logic. | [FM6](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch) |
| FM6-5 | Add tests asserting `l2ChainId == 0` enforcement and correct super root timestamp `uint64` decoding. | [FM6](#fm6-cwia-game-args-layout-and-extradata-offset-mismatch) |
| FM7-1 | Establish a minimum `challengerBond` policy such that `challengerBond > expected full superchain proving cost` to ensure defending is always profitable. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-2 | Calibrate `maxProveDuration` per aZKG-007 with analysis of L1 censorship costs and proof generation time on reference hardware for the full chain set. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-3 | Monitor challenge rates in production and have a runbook for adjusting bond and duration parameters if griefing is detected. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM7-4 | Add gas consumption regression tests for the verifier. | [FM7](#fm7-bond-and-duration-misconfiguration) |
| FM10-1 | Verify OPCM migration script sets a valid super root anchor atomically before or in the same transaction as `setRespectedGameType(ZKDisputeGame)`. | [FM10](#fm10-anchor-state-type-mismatch-on-first-super-game) |
| FM12-1 | Establish a parameter review process triggered by any chain addition or per-chain config change (gas limit, block time, EVM version). | [FM12](#fm12-shared-parameter-drift-as-superchain-scales) |
| FM12-2 | Calibrate `maxProveDuration` and `maxChallengeDuration` to the worst-case chain in the current superchain set and document the required margin (≥20%). | [FM12](#fm12-shared-parameter-drift-as-superchain-scales) |
| FM12-3 | Calibrate `initBond` to aggregate superchain TVL and `challengerBond` to full superchain proving cost on reference hardware, and document the methodology. | [FM12](#fm12-shared-parameter-drift-as-superchain-scales) |
