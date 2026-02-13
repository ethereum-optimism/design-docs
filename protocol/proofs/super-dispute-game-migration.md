# SuperDisputeGame Migration: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Steven Nevins                                      |
| Created at         | 2026-02-02                                         |
| Initial Reviewers  | _TBD_                                              |
| Need Approval From | Adrian Sutton                                      |
| Status             | Draft                                              |

## Summary

Migrate existing single-chain FaultDisputeGame chains to SuperDisputeGame as a prerequisite for interop. This unifies all chains onto one game type, enabling future interop sets to share a SuperDisputeGame. Changes must be backward compatible with OptimismPortal2 to avoid invalidating existing withdrawals.

## Problem Statement + Context

- **Current state**: Each chain has its own dispute game (FaultDisputeGame/PermissionedDisputeGame)
- **Interop requirement**: Chains need to share dispute infrastructure
- **Step 1**: Migrate individual chains to SuperDisputeGame (this doc)
- **Step 2 (future)**: Interop sets share a single SuperDisputeGame

### Key Constraints

- **Backward compatibility**: OptimismPortal2 must continue working with in-flight withdrawals
- **No withdrawal invalidation**: Existing proven withdrawals must remain valid
- Migration path must be safe for live chains

## Proposed Solution

Migration happens in two aspects to maintain backward compatibility.

### Change 1: Update OptimismPortal2 (Backward Compatible)

**Problem**: Old FaultDisputeGame contracts don't have `rootClaimByChainId()`. SuperDisputeGame uses this method to return chain-specific output roots from the SuperRoot. Calling it on old games reverts.

**Solution**: Add `_isSuperGameType()` helper to OptimismPortal2. Before extracting the output root, check the game type:

```solidity
function _isSuperGameType(GameType _gameType) internal pure returns (bool) {
    uint32 rawType = GameType.unwrap(_gameType);
    return rawType == GameType.unwrap(GameTypes.SUPER_CANNON)           // 4
        || rawType == GameType.unwrap(GameTypes.SUPER_PERMISSIONED_CANNON) // 5
        || rawType == GameType.unwrap(GameTypes.SUPER_ASTERISC_KONA)    // 7
        || rawType == GameType.unwrap(GameTypes.SUPER_CANNON_KONA);     // 9
}
```

Use it in the prove flow:

```solidity
Claim outputRootClaim;
if (_isSuperGameType(disputeGameProxy.gameType())) {
    outputRootClaim = disputeGameProxy.rootClaimByChainId(systemConfig.l2ChainId());
} else {
    outputRootClaim = disputeGameProxy.rootClaim();
}
```

**Note**: SuperDisputeGame's `rootClaim()` returns the SuperRoot hash (aggregate of all chains), NOT an individual OutputRoot. The `rootClaimByChainId()` method extracts the correct chain-specific OutputRoot from the SuperRoot preimage.

**Alternative Approaches Considered**:

1. **Try/catch fallback**: Call `rootClaimByChainId()` first, fallback to `rootClaim()` on revert. More future-proof (no update needed for new game types), but adds gas overhead and complexity.

2. **Update FaultDisputeGame first**: Ship `rootClaimByChainId()` to old games before OptimismPortal change. Older games prior to upgrade still couldn't prove new withdrawals, but finalization would still work.

**Chosen approach**: Hardcoded `_isSuperGameType()` check for simplicity. Tradeoff is requiring updates when new super game types are added.

### Change 2: Update OPCM, upgrade existing contracts, and update game type (via OPCM)

OPContractsManager v2 orchestrates the migration via `OPCM.upgrade()`. The upgrade executes atomically in a single transaction via `DELEGATECALL` from the Upgrade Controller Safe—if any step fails, all revert.

See [OPCM v2 Design Doc](../opcm-v2.md) for details.

High level Overview of Steps:
1. Deploy SuperDisputeGame implementation (by OPCM)
2. Upgrade AnchorStateRegistry implementation via proxy
    - Use target super-root game type for `_startingRespectedGameType` in the initializer
    - Set `_startingAnchorRoot` to a valid super root (existing anchor is an output root, not compatible)
    - Clear `anchorGame` to `address(0)` in the reinitializer — on live chains `anchorGame` is already set, and `getAnchorRoot()` checks it first. Without clearing, the new `startingAnchorRoot` is ignored entirely.
    - **Implementation note**: The reinitializer must accept a flag to opt into clearing `anchorGame`. This clearing must only happen during the initial super game migration — subsequent upgrades must NOT clear `anchorGame`, as it would reset the anchor to the original starting root.
3. Upgrade OptimismPortal2 implementation via proxy
4. Register SDG in DisputeGameFactory with new game type
5. Disable legacy game types in DisputeGameFactory (set implementation to `address(0)`)

Do NOT call `updateRetirementTimestamp()` or update the retirementTimestamp during migration.
    - This would retire ALL games created before the call, invalidating in-flight withdrawals.

#### OPCM V2 Migration Mode

Migration support is gated behind a feature flag (`DevFeatures.SUPER_ROOT_GAMES_MIGRATION`) to allow isolated development and testing.

**Feature Flag Behavior**:
- **Flag OFF**: Existing upgrade behavior unchanged (requires 4 game configs)
- **Flag ON**: Super-root migration mode active with constraints below

**Mode-Safety Constraints**:

The migration must preserve the chain's permission mode. OPCM reads the original `respectedGameType` directly from AnchorStateRegistry (not from overrides) to determine the chain's current mode:

| Original Chain Mode | Allowed Target | Rejected Target |
|---------------------|----------------|-----------------|
| Permissionless (CANNON, CANNON_KONA) | `SUPER_CANNON_KONA` | `SUPER_PERMISSIONED_CANNON`, `SUPER_CANNON` |
| Permissioned (PERMISSIONED_CANNON) | `SUPER_PERMISSIONED_CANNON` | `SUPER_CANNON_KONA`, `SUPER_CANNON` |

Cross-mode migration (permissionless ↔ permissioned) is explicitly prevented.

**Config Requirements (flag ON)**:

| Original Mode | Required Configs |
|---------------|-----------------|
| Permissionless | `SUPER_CANNON_KONA` + `SUPER_PERMISSIONED_CANNON` |
| Permissioned | `SUPER_PERMISSIONED_CANNON` |

- `SUPER_PERMISSIONED_CANNON` config is always required regardless of mode
- `SUPER_CANNON_KONA` config is required for permissionless chains, ignored/rejected for permissioned chains
- `startingRespectedGameType` override must match `SUPER_CANNON_KONA` (permissionless) or `SUPER_PERMISSIONED_CANNON` (permissioned)
- For permissionless chains, `SUPER_PERMISSIONED_CANNON` is registered but NOT set as the respected game type (enables fallback switch if needed)
- Cross-mode respected game type (permissionless ↔ permissioned) still explicitly prevented

**Legacy Type Cleanup (flag ON)**:

Post-migration, OPCM disables legacy game types in DisputeGameFactory (sets implementation to zero address, init bond to zero):
- `CANNON`
- `PERMISSIONED_CANNON`
- `CANNON_KONA`
- `SUPER_CANNON`

Note: `SUPER_PERMISSIONED_CANNON` is NOT disabled — it is always registered.

Only `SUPER_PERMISSIONED_CANNON` and (for permissionless chains) `SUPER_CANNON_KONA` remain registered.

**Note on Permissioned Games**:

There is no `SUPER_PERMISSIONED_CANNON_KONA` game type. While permissioned games are technically configured with a specific VM (cannon) and program (op-program), the permission mechanism prevents games from being played out to the point where VM/program differences matter. For super games, `SUPER_PERMISSIONED_CANNON` can use cannon/kona since it hasn't been deployed yet.

#### Operator Usage

To execute migration via `OPCM.upgrade()`:
1. Generate and verify the starting super root (see [Starting Super Root & Governance Verification](#starting-super-root--governance-verification))
2. Enable `DevFeatures.SUPER_ROOT_GAMES_MIGRATION` in development environment
3. Provide dispute game config(s):
   - Permissionless: configs for both `SUPER_CANNON_KONA` and `SUPER_PERMISSIONED_CANNON`
   - Permissioned: config for `SUPER_PERMISSIONED_CANNON`
4. Provide `extraInstructions` with:
   - `startingRespectedGameType` override matching target
   - `startingAnchorRoot` set to the generated super root
5. OPCM validates mode-safety automatically

#### Starting Super Root & Governance Verification

The starting super root is passed as an `OutputRoot` to the ASR reinitializer, where:

- `OutputRoot.root` = `keccak256(encode(SuperRootProof))`
- `OutputRoot.l2SequenceNumber` = `SuperRootProof.timestamp`

For a single-chain migration, the super root wraps exactly one output root:

```
SuperRootProof {
    version:     0x01
    timestamp:   <L2 timestamp of the output root>
    outputRoots: [{
        chainId: <SystemConfig.l2ChainId()>
        root:    <current anchor output root>
    }]
}
```

**What governance must verify (pre-upgrade, from the upgrade calldata)**:

1. **Decode and verify `version`** — Decode the `OutputRoot.root` preimage (provided alongside the upgrade transaction) via `Encoding.decodeSuperRootProof()`. Verify `version == 0x01`.
2. **Verify `chainId`** — The decoded `outputRoots[0].chainId` must match `SystemConfig.l2ChainId()` for the chain being migrated. Cross-reference against the superchain registry.
3. **Verify the super root** — Use the [`check-super-root`](https://github.com/ethereum-optimism/optimism/tree/develop/op-chain-ops/cmd/check-super-root) script to verify the super root can be recomputed and is correct.
4. **Verify `l2SequenceNumber`** — Must equal `SuperRootProof.timestamp`.
5. **Verify only one chain** — `outputRoots.length == 1` for single-chain migration.

### Why Withdrawals Stay Safe

The `wasRespectedGameTypeWhenCreated` flag protects existing withdrawals:

- Flag is set ONCE during game initialization, stored in game contract
- `AnchorStateRegistry.isGameRespected()` returns this stored flag
- Changing `respectedGameType` does NOT retroactively invalidate old games

### Assumptions

- OptimismPortal2 (NOT OptimismPortalInterop, which already has `superRootsActive`)
- **ChainId**: `SystemConfig.l2ChainId()` must be set correctly before migration
## Impact on Developer Experience

**Application developers**: Function signatures unchanged, but game selection logic changes:
- **Before (FaultDisputeGame)**: Find game where `l2BlockNumber` > withdrawal initiation block
- **After (SuperDisputeGame)**: Find game where `l2SequenceNumber` (timestamp) > withdrawal initiation block timestamp

SDKs and proving services need updates to query by timestamp instead of block number.

## Alternatives Considered

### 1. Replace Implementation in DisputeGameFactory

Register SDG at the same game type ID as FDG, replacing the old implementation.

**Rejected**: Breaks existing games. DisputeGameFactory uses game type + root claim to look up games. Changing the implementation for an existing type would cause `isGameRegistered()` to fail for old games, bricking in-flight withdrawals.

### 2. Always Use `rootClaimByChainId()`

Assume all games have `rootClaimByChainId()` and call it unconditionally.

**Rejected**: Old deployed FaultDisputeGame contracts don't have this method. The interface was added later, but deployed bytecode is immutable. Calling `rootClaimByChainId()` on old games reverts, breaking withdrawals.

## Risks & Uncertainties

### Multi-Version Jump Risk

**Risk**: Teams that skip intermediate upgrade versions can brick their withdrawals.

### ChainId Dependency

**Risk**: If `SystemConfig.l2ChainId()` is incorrect, `rootClaimByChainId()` returns wrong data or reverts.

**Mitigation**: OPCM should add a precondition check that `SystemConfig.l2ChainId()` matches the chain's expected chain ID from the superchain registry.

### Open Questions

- Are there special edge cases for deploy vs upgrade that we need to consider?

- Details for merging into interop sets and shared games?