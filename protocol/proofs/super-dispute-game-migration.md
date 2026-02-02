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

### Change 2: Update OPCM, upgrade existing contracts, and update game type (via OPCM)

OPContractsManager v2 orchestrates the migration via `OPCM.upgrade()`. The upgrade executes atomically in a single transaction via `DELEGATECALL` from the Upgrade Controller Safe—if any step fails, all revert.

See [OPCM v2 Design Doc](../opcm-v2.md) for details.

High level Overview of Steps:
1. Deploy SuperDisputeGame implementation (by OPCM)
2. Upgrade AnchorStateRegistry implementation via proxy
    - Use `SUPER_CANNON_KONA` for _startingRespectedGameType in the initializer
3. Upgrade OptimismPortal2 implementation via proxy
4. Register SDG in DisputeGameFactory with new game type

Do NOT call `updateRetirementTimestamp()` or update the retirementTimestamp during migration. 
    - This would retire ALL games created before the call, invalidating in-flight withdrawals.

### Why Withdrawals Stay Safe

The `wasRespectedGameTypeWhenCreated` flag protects existing withdrawals:

- Flag is set ONCE during game initialization, stored in game contract
- `AnchorStateRegistry.isGameRespected()` returns this stored flag
- Changing `respectedGameType` does NOT retroactively invalidate old games

### Assumptions

- OptimismPortal2 (NOT OptimismPortalInterop, which already has `superRootsActive`)
- All deployed FaultDisputeGame contracts have `wasRespectedGameTypeWhenCreated`. If older games exist, migration must wait for them to finalize first.
    - Potential issue if teams are upgrading from a version older than when this function was added that their inflight withdrawals would be bricked. I think this assumption would need to be handled via comms that you should keep up with the upgrade process. 
- **ChainId**: `SystemConfig.l2ChainId()` must be set correctly before migration

## Failure Mode Analysis

See [FMA: SuperDisputeGame Migration](../../security/fma-super-dispute-game-migration.md).

## Impact on Developer Experience

**Application developers**: No change. Withdrawal proving/finalizing works the same from user perspective. `OptimismPortal.proveWithdrawalTransaction()` and `finalizeWithdrawalTransaction()` signatures unchanged.

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

**Mitigation**: Add precondition check before migration. Verify chainId is set correctly in SystemConfig.

### Open Questions

- Should we think through a fallback to OPCM v1 ? Are we okay being coupled to OPCM v2

- Are we allowing upgrading to either the SuperPermissioned/SuperDisputeGame?

- SUPER_CANNON_KONA is the gameType for this?

- Should we enforce based upgrade logic like if you're upgrading a Permissionless chain that you upgrade to a Permissionless Super game

- Does anything happen with the OptimismPortalInterop?

- Should this doc cover deploy or only the upgrade