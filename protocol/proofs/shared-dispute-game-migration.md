# Design Doc: Migrating Chains to Shared Super Dispute Game

| | |
|---|---|
| **Author(s)** | Steven Nevins |
| **Status** | Draft |
| **Created** | 2026-03-19 |

## Purpose

This document describes the on-chain migration that moves independent OP Stack chains from per-chain dispute games to a shared super dispute game.

## Summary

Today, each OP Stack chain runs its own dispute infrastructure: a per-chain `DisputeGameFactory`, `AnchorStateRegistry`, `OptimismPortal2`, and `ETHLockbox`. Each game validates a single chain's output root in isolation.

The migration consolidates N independent chains into a shared set of these contracts, backed by **super dispute games** whose `rootClaim` is a `SuperRootProof` containing output roots for ALL participating chains. This enables cross-chain state commitment at the dispute layer.

Key outcomes:
- **Portal unification**: `OptimismPortalInterop` features are merged into `OptimismPortal2`, making OP2 the single portal contract. OPInterop is deleted.
- **Shared infrastructure**: All migrated chains share one `DisputeGameFactory`, one `AnchorStateRegistry`, one `ETHLockbox`, and one `DelayedWETH`.
- **Two-phase execution**: (1) per-chain OPCM upgrade prepares each chain, then (2) an atomic multi-chain `migrate()` call deploys shared infra and wires everything together.
- **Proof routing via game type**: `_isSuperGameType()` in OP2 (`OptimismPortal2.sol:642-646`) determines whether to use `rootClaimByChainId()` or `rootClaim()` based on the game type enum. No `superRootsActive` storage flag needed.
- **CGT chain compatibility**: Not in v1 scope. Attempting to migrate a CGT chain will revert — this prevents CGT chains from being incorrectly pooled into the shared ETHLockbox. `OptimismPortal2` retains CGT support via the `Features` library so CGT chains can be added in a future iteration.

## Problem Statement + Context

Each OP Stack chain currently runs isolated dispute infrastructure. Its `DisputeGameFactory` creates games that validate a single chain's output root, its `AnchorStateRegistry` tracks anchor state for that chain only, and its `ETHLockbox` holds ETH for that chain's withdrawals. Nothing connects these chains at the dispute layer.

Interop breaks this model. When Chain A sends a message to Chain B, the system needs to prove that Chain A's state is valid *at the same time* as Chain B's state. A per-chain game that only commits to one chain's output root cannot provide this guarantee. The dispute layer must commit to the state of ALL participating chains simultaneously — a single game whose root claim covers every chain in the interop set.

This also means the infrastructure backing those games must be shared. A per-chain `AnchorStateRegistry` only knows about its own chain's games. A shared ASR is needed so that game validity checks (anchor state, respected game type, blacklisting) apply consistently across all chains resolving the same super root.

ETH liquidity must also be pooled. If users bridge ETH from Chain A to Chain B through interop messaging, Chain B's portal needs access to that ETH when finalizing withdrawals. Per-chain lockboxes cannot serve cross-chain withdrawals — a shared lockbox holding the combined ETH of all participating chains is required.

Finally, the current codebase has two portal contracts: `OptimismPortal2` (live, supports CGT chains) and `OptimismPortalInterop` (not live, has migration functions but lacks CGT support). Shipping interop through OPInterop would permanently fork the portal — CGT chains could never participate, and every security fix would need to land in both contracts. Merging OPInterop's migration features into OP2 eliminates this split.

## Proposed Solution

### 1. Portal Unification (Merge OPInterop into OP2)

**What changes in `OptimismPortal2`:**

Add from `OptimismPortalInterop`:
- `migrateToSharedDisputeGame(IETHLockbox _newLockbox, IAnchorStateRegistry _newAnchorStateRegistry)` (currently `migrateToSuperRoots` in code, proposed rename) — swaps lockbox and ASR references, emits `PortalMigrated` event (from `OptimismPortalInterop.sol:381-414`)
- `migrateLiquidity()` — transfers portal's ETH balance to its lockbox (from `OptimismPortalInterop.sol:359-367`)
- `upgrade(IAnchorStateRegistry, IETHLockbox)` — reinitializer for upgrading portal references (from `OptimismPortalInterop.sol:263-276`)
- `ETHMigrated` and `PortalMigrated` events
- `OptimismPortal_MigratingToSameRegistry` error
- Bump `ReinitializableBase(3)` to `ReinitializableBase(4)` in the constructor

**What OP2 already has that diverged from OptimismPortalInterop** (keep as-is):
- CGT support via `_isUsingCustomGasToken()` (`OptimismPortal2.sol:656-660`)
- `Features` library integration
- `_isUsingLockbox()` for conditional lockbox operations (`OptimismPortal2.sol:650-652`)
- `_assertValidLockboxState()` (`OptimismPortal2.sol:670-677`)
- `_isSuperGameType()` for stateless proof routing (`OptimismPortal2.sol:642-646`)

**Design Decision: SuperchainConfig Version Checks**

Unlike `deploy` and `upgrade`, migration does NOT enforce a SuperchainConfig version floor.

**OPContractsManagerMigrator update:**

The migrator currently casts portals to `IOptimismPortalInterop` (`OPContractsManagerMigrator.sol:244`). After unification, it casts to `IOptimismPortal2` instead, since the unified interface lives on OP2.

### 2. Pre-Migration Constraints

Before any migration step, several invariants must hold.

**Governance alignment.** All chains must share the same ProxyAdmin owner (`OPContractsManagerMigrator.sol:96`) and reference the exact same SuperchainConfig address (`OPContractsManagerMigrator.sol:102`). Shared infrastructure is initialized with a single ProxyAdmin and SuperchainConfig — the ETHLockbox enforces this at authorization time (`ETHLockbox.sol:220-224`).

**Portal version.** The portal MUST already be upgraded to the version with migration functions. If not, `migrateToSharedDisputeGame()` reverts (`OPContractsManagerMigrator.sol:277-279`).

**Feature flags.** The `OPTIMISM_PORTAL_INTEROP` dev feature flag must be enabled on the OPCM container (`OPContractsManagerMigrator.sol:80`).

### 3. Step 1: Per-Chain OPCM Upgrade (`OPContractsManagerV2.upgrade()`)

This step runs once per chain BEFORE the atomic migration.

**What happens:**
- Portal upgraded to the version with migration functions (`OPContractsManagerV2.sol:744-752`)
- ASR reinitialized with `_clearAnchorGame=true` — resets anchor game so `getAnchorRoot()` returns new `startingAnchorRoot`
- Game config validated via `assertValidSuperRootMigrationConfig()` (`OPContractsManagerUtils.sol:465-525`)
- Legacy game types (CANNON, PERMISSIONED_CANNON, CANNON_KONA, SUPER_CANNON) disabled on per-chain DGF

**Constraints:**
- **Game config must match chain's current mode.** Permissioned chains target `SUPER_PERMISSIONED_CANNON`, permissionless target `SUPER_CANNON_KONA`. No security model changes during migration.
- **Starting anchor root must differ from current.** Prevents accidental no-op.
- **Legacy game types disabled.** Prevents split-brain where both legacy and super games exist on the same DGF.
- **OPCM version sequence.** Cannot skip versions.

**Transitional state:** After this step, chains function independently with super game types on their per-chain DGF. This is functional but NOT the target state — shared infrastructure comes in Step 2.

### 4. Step 2: Atomic Multi-Chain Migrate (`OPContractsManagerMigrator.migrate()`)

This step runs ONCE, atomically, for ALL chains in the migration set.

**What happens:**

1. **Deploy shared infrastructure** (`OPContractsManagerMigrator.sol:125-156`): new ETHLockbox, DisputeGameFactory, and AnchorStateRegistry proxies. Reuses existing DelayedWETH.

2. **Initialize shared contracts** (`OPContractsManagerMigrator.sol:166-205`): ETHLockbox with first chain's SystemConfig, DGF with ProxyAdmin owner, ASR with starting anchor root and respected game type.

3. **Per-portal migration** (`_migratePortal` at `OPContractsManagerMigrator.sol:236-281`): For each chain:
   - Authorize portal on new lockbox (`OPContractsManagerMigrator.sol:248`)
   - Authorize old lockbox on new lockbox for liquidity transfer (`OPContractsManagerMigrator.sol:251`)
   - Migrate liquidity from old lockbox to new lockbox (`OPContractsManagerMigrator.sol:254`)
   - Clear all game type implementations on old DGF — sets all 6 types (CANNON, SUPER_CANNON, PERMISSIONED_CANNON, SUPER_PERMISSIONED_CANNON, CANNON_KONA, SUPER_CANNON_KONA) to `address(0)` (`OPContractsManagerMigrator.sol:261-266`)
   - Enable `ETH_LOCKBOX` feature on SystemConfig if not already enabled (`OPContractsManagerMigrator.sol:270-272`)
   - Call `migrateToSharedDisputeGame()` to swap portal's lockbox and ASR references (`OPContractsManagerMigrator.sol:279`)

4. **Register game types** on new shared DGF from caller-supplied `DisputeGameConfig[]` input (`OPContractsManagerMigrator.sol:221-228`).

**Constraints:**
- **Atomicity required.** Liquidity migration and portal migration must happen in the same tx (`ETHLockbox.sol:198-200`). If the portal points to a new lockbox but liquidity hasn't transferred, withdrawals revert.
- **System must NOT be paused.** `migrateToSharedDisputeGame()` calls `_assertNotPaused()` (`OptimismPortalInterop.sol:384`).
- **N independent pre-interop chains only.** Does not support partial migration, re-migration, or adding chains later. Re-calling on already-migrated portals corrupts the shared DGF (`OPContractsManagerMigrator.sol:69-73`).
- **Old DGF implementations cleared.** All 6 game types (CANNON, SUPER_CANNON, PERMISSIONED_CANNON, SUPER_PERMISSIONED_CANNON, CANNON_KONA, SUPER_CANNON_KONA) set to `address(0)` (`OPContractsManagerMigrator.sol:261-266`). In-progress games on old DGFs are orphaned.
- **One-way operation.** Requires a contract upgrade to revert (not controlled by the chain operator).

### 5. Post-Migration State

| Component | Before | After |
|---|---|---|
| DisputeGameFactory | Per-chain | Shared across all chains |
| OptimismPortal | Points to per-chain DGF/ASR/Lockbox | Points to shared DGF/ASR/Lockbox |
| AnchorStateRegistry | Per-chain | Shared across all chains |
| ETHLockbox | Per-chain (or none) | Shared across all chains |
| Game types | CANNON, PERMISSIONED_CANNON, CANNON_KONA | SUPER_CANNON_KONA, SUPER_PERMISSIONED_CANNON |
| Proof method | `rootClaim()` (single chain) | `rootClaimByChainId(chainId)` (per-chain from super root) |

**All prior withdrawal proofs are invalidated.** The ASR reference changes — old proofs reference games from the old DGF, which the new ASR does not track. Users must re-prove (`OPContractsManagerMigrator.sol:62-64`). Additionally, the new ASR sets `retirementTimestamp = block.timestamp` on initialization, which retires all games created before or at that block as an extra safety measure (`AnchorStateRegistry.sol:106-113`).

**Irreversible without full contract upgrade.** No `migrateBack()` exists.

### Resource Usage

**Gas**: `migrate()` performs O(N) per-portal operations in a single transaction. Each portal migration involves ~6 external calls plus storage writes.

**Multi-client**: Migration targets `SUPER_CANNON_KONA` (Kona/Rust) for permissionless and `SUPER_PERMISSIONED_CANNON` for permissioned games. `SUPER_CANNON` (Go/Cannon) is disabled during the per-chain upgrade.

### Single Point of Failure Considerations

| Component | Impact if compromised |
|---|---|
| Shared DGF | No chain can create new dispute games. In-progress games still resolve. |
| Shared ASR | Game validity checks (`isGameProper`, `isGameRespected`, `isGameClaimValid`) fail across all chains. |
| Shared ETHLockbox | All ETH across all chains at risk. Mitigated by authorized portals check and pause mechanism. |
| SuperchainConfig | Already shared pre-migration. Post-migration, pausing also blocks dispute resolution. |

## Failure Mode Analysis

See the existing FMA document for comprehensive failure mode analysis.

**Two-step failure gap:** Between Step 1 and Step 2, chains run super game types against per-chain infrastructure. If Step 2 is delayed, chains are stuck in a transitional state with no rollback function.

## Impact on Developer Experience

**Users must re-prove withdrawals.** SDKs and bridge UIs need to detect invalidated proofs and guide users through re-proving.

**New game types in tooling.** Off-chain tools must recognize `SUPER_CANNON_KONA` and `SUPER_PERMISSIONED_CANNON`. The `rootClaim` format changes to `SuperRootProof`. Critically, tools must call `rootClaimByChainId(chainId)` instead of `rootClaim()` when validating output roots or constructing withdrawal proofs — `rootClaim()` returns the full super root, not the per-chain output root.

**Consolidated monitoring.** Operators monitor one DGF instead of N. Simpler alerting, higher stakes per alert.

## Alternatives Considered

### Keep two portal contracts (OP2 + OPInterop)

Rejected. OPInterop is not live and lacks CGT support. Doubles audit surface and creates feature divergence.

### Use `superRootsActive` storage flag instead of `_isSuperGameType()`

Rejected. The game type enum already encodes whether a game is a super game — a storage flag is redundant state that must be kept in sync.

### Single-step atomic migration (no per-chain upgrade step)

Rejected. Per-chain upgrade validates each chain's game config independently. Combining everything into one step makes the transaction larger and harder to reason about.

### Gradual migration (migrate chains one at a time)

Rejected. The shared DGF and ASR must be initialized with the complete set from the start. Incremental migration would require re-migration support, which corrupts the shared DGF (`OPContractsManagerMigrator.sol:69-73`).

### Keep `OptimismPortalInterop` as a separate deployment for interop-only chains

Rejected. Creates a permanent fork where CGT chains can never participate in interop.

## Risks & Uncertainties

### Resolved

- **CGT chains and interop**: CGT is not required for the first release — no CGT chains are immediately lined up for interop. Future CGT support requires splitting lockbox migration from the shared dispute game migration so CGT chains can join without ETH pooling (they must not deploy ETH bridging contracts like `SuperchainETHBridge`/`ETHLiquidity`).

### Open questions

- **Withdrawal proof invalidation**: All prior withdrawal proofs are invalidated during migration. Is this accepted for v1, or do we need to design a way to preserve in-flight proofs? May have a different answer for initial release vs future chain additions.

### Trust assumptions

- **ProxyAdmin owner coordination**: All chains must trust the same ProxyAdmin owner to execute the migration correctly. A malicious or compromised owner can corrupt all chains simultaneously.

### Operational gaps

- **No rollback mechanism**: If the migration succeeds but a bug is discovered in the shared infrastructure, there is no `undo()`. Fixing requires deploying new shared contracts and migrating again (which the current migrator does not support — see re-migration constraint).
- **Orphaned in-progress games**: Games created on old per-chain DGFs before migration can still resolve, but their results are not tracked by the new shared ASR. Users who proved withdrawals against these games must re-prove against the new DGF.
- **No partial migration support (v1 constraint)**: There is no way to migrate N-1 chains and add the Nth later. Incremental chain addition is planned for a future iteration.

## Key Reference Files

| File | Description |
|---|---|
| `src/L1/OptimismPortal2.sol` | Current OP2 — target for merged migration features |
| `src/L1/OptimismPortalInterop.sol` | Source of migration features to merge |
| `src/L1/opcm/OPContractsManagerMigrator.sol` | Atomic multi-chain migration orchestrator |
| `src/L1/opcm/OPContractsManagerV2.sol` | Per-chain OPCM upgrade (Step 1) |
| `src/L1/opcm/OPContractsManagerUtils.sol` | Validation logic (`assertValidSuperRootMigrationConfig`) |
| `src/dispute/SuperFaultDisputeGame.sol` | Super game implementation with `rootClaimByChainId` |
| `src/L1/ETHLockbox.sol` | Shared ETH liquidity management |
| `src/dispute/AnchorStateRegistry.sol` | Game validation and anchor state |
| `src/libraries/Features.sol` | Feature flags (ETH_LOCKBOX, CUSTOM_GAS_TOKEN) |
| `src/libraries/DevFeatures.sol` | Dev feature flags (SUPER_ROOT_GAMES_MIGRATION, OPTIMISM_PORTAL_INTEROP) |
