# Shared Lockbox: Failure Modes and Recovery Path Analysis

| Author | Joxes, Agusduha, Gotzen |
| --- | --- |
| Created at | 2024-12-20 |
| Initial Reviewers | Pending |
| Needs Approval From | Pending |
| Status | Draft |

## Introduction

This document covers the changes introduced by the addition of the [Shared Lockbox design](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/eth-shared-lockbox.md), a singleton contract that stores all ETH liquidity given a set of interoperable chains. This design addresses the [ETH withdrawals problem](https://github.com/ethereum-optimism/specs/issues/362) by the introduction of [SuperchainWETH](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/superchain-weth.md). The following components are:

- **Contracts**:
    - Introduction of the `SharedLockbox`.
    - Updates to the `OptimismPortal` to lock and unlock ETH from the `Sharedlockbox`.
    - Updates to the `SuperchainConfig` to manage the dependency, thereby removing this functionality from `SystemConfig`.
    - Introduction of `LiquidityMigrator`, an intermediate contract for moving ETH liquidity into the `SharedLockbox`.
- **Scripts**:
    - Management of `OptimismPortal` upgrades that include the ETH liquidity migration process.

Below are references for this project:

- [Design doc](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/eth-shared-lockbox.md).
- [Specs PR](https://github.com/ethereum-optimism/specs/pull/465).
- [Implementation PR](https://github.com/ethereum-optimism/optimism/pull/13144).

## Failure Modes and Recovery Paths

### **Unauthorized access to `unlockETH` function**

- **Description:** if an attacker bypasses access controls for `unlockETH`, the deposit and withdrawal invariants can be broken, potentially resulting in the worst-case scenario of draining all ETH liquidity from the `SharedLockbox`.
- **Risk Assessment:** High
    - Potential Impact: High. Unauthorized unlocking ETH would result in a direct loss of ETH from the `SharedLockbox`.
    - Likelihood: Medium. While access controls are specified in the implementation, errors or misconfigurations remain possible.
- **Mitigation:** Access control must be strictly enforced, permitting only approved `OptimismPortal` contracts via the `authorizedPortals` mapping.
- **Detection:** Monitor `ETHUnlocked` events to verify consistency with authorized portal addresses. Set up alerts for suspected unauthorized activity from non-approved addresses.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox` upon detection of unauthorized access. Liquidity can be restored by depositing ETH directly in the `SharedLockbox`.

### Unauthorized access to the `addDependency` function and `authorizedPortals` mapping

- **Description:** The `addDependency` function is intended to be callable only by the `dependencyManager`. If bypassed, a malicious actor could register arbitrary addresses in the `dependencySet` within `SuperchainConfig`, and the `authorizedPortals` mapping, putting the ETH liquidity at risk.
- **Risk Assessment:** High
    - Potential Impact: High. Malicious addresses in `authorizedPortals` could trigger a `finalizeWithdrawalTransaction` function to drain all the ETH liquidity.
    - Likelihood: Low. The `dependencyManager` is the only entity permitted to update these mappings, but implementation flaws or misconfigurations could occur.
- **Mitigation:** Verify that access to the `addDependency` and `authorizePortal` functions are restricted exclusively to the designated `dependencyManager`.
- **Detection:** Monitor the `ConfigUpdate` to ensure the correct `dependencyManager` address is set, as this event happens in the `initialize` function of `SuperchainConfig`. Additionally, monitor  `DependencyAdded` events to verify that only expected addresses are registered and executed by the `dependencyManager`.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox` upon detecting unexpected addresses. As removal functions are unavailable, an upgrade directly over the `SuperchainConfig` and `SharedLockbox` to clear storage slots will be required.

### Mismanagement of `addDependency` function

- **Description:** The `dependencyManager` could accidentally or maliciously add incorrect addresses to the dependency set.
- **Risk Assessment:** High
    - Potential Impact: High. Similarly to unauthorized access, incorrect addresses could disrupt functionality or put funds at risk.
    - Likelihood: Low. Current security standards are enough to safeguard the process, so is expected to follow best practices.
- **Mitigation:** Up to the best practices of those in charge of the `dependencyManager` role.
- **Detection:** Monitor `DependencyAdded` events for unauthorized or erroneous additions.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox` upon detecting unexpected addresses. As removal functions are unavailable, an upgrade directly over the `SuperchainConfig` and `SharedLockbox` to clear storage slots will be required.

### Re-Entrancy Attacks on `finalizeWithdrawalTransaction` and `unlockETH` functions

- **Description:** A reentrancy vulnerability in `unlockETH` could allow an attacker to repeatedly call the function before the state is updated, extracting more liquidity than allowed.
- **Risk Assessment:** High.
    - Potential Impact: High. Direct loss of ETH from the `SharedLockbox`.
    - Likelihood: Medium. The `finalizeWithdrawalTransaction` function is modified to make it work using the `SharedLockbox`. Now `finalizeWithdrawalTransaction` make an internal call in `unlockETH` which transfers ETH back using the `donateETH` function. This flow must be carefully reviewed for potential vulnerabilities.
- **Mitigation:** Ensure state validation checks are implemented to prevent repeated calls within the same transaction.
- **Detection:** Implement scripts that monitor `unlockETH` and `finalizeWithdrawalTransaction` functions. A function called multiple times over any of both in the same transaction that breaks balance invariants should raise an alert.
- **Recovery Path(s):** Pause the system through SuperchainConfig to halt the `SharedLockbox`.

### Adding a chain from another cluster in `addDependency`

- **Description:** Adding a chain that belongs to another cluster can cause liquidity issues in the `SharedLockbox`, as ETH will remain in the old `SharedLockbox`. This could lead to withdrawal failures in the new one.
- **Risk Assessment:** High.
    - Potential Impact: High. The `SharedLockbox` may have less ETH liquidity than expected. ETH withdrawals are at risk to fail for any `OptimismPortal` in the cluster because a portion of the expected ETH was not migrated and remains in the old `SharedLockbox`.
    - Likelihood: Low. Current security standards are enough to safeguard the process, so is expected to follow best practices.
- **Mitigation:** Up to the best practices of those in charge of the `dependencyManager` role. This involves verifying if the chain belongs to another cluster, with two scenarios to consider:
    - **Independent cluster**: The chain is part of its own cluster without extra dependencies. In this case, it must be ensured that its `SharedLockbox` migrates the entire ETH liquidity to the new cluster. Note: Liquidity migration between `SharedLockbox`s is not yet possible in the current code.
    - **Shared cluster**: The chain is part of a cluster with other dependencies. In this case, the chain cannot be migrated individually; the entire cluster must migrate as a unit. This is because there is no on-chain mechanism to distinguish ETH liquidity for individual chains (although this could be done off-chain, it introduces additional security risks).
- **Detection:** Monitor `DependencyAdded` events for unauthorized or erroneous additions.  Implement security measures to ensure that chains added do not belong to a previously shared cluster.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox` upon detecting unexpected addresses. As removal functions are unavailable, an upgrade directly over the `SuperchainConfig` and `SharedLockbox` to clear storage slots will be required.

### Equivocation in batch execution

- **Description:** The batch transaction process for migrating a chain to adopt the `SharedLockbox` requires three sequential calls: the first to `SuperchainConfig` and the subsequent two to `ProxyAdmin`. If the transaction is executed incorrectly, the migration may remain incomplete, leading to erratic system behavior due to an unexpected resultant state. While the batched migration process assumes that if one call fails, the entire batch will fail, it does not enforce the inclusion or ordering of the necessary calls to ensure a successful migration. This limitation can result in multiple failure scenarios.
- **Risk Assessment:** Medium.
    - Potential Impact: Low to Medium. Several complications may arise depending on the case (see table at the end).
    - Likelihood: Low. Current security standards and adherence to best practices are expected to minimize the likelihood of incorrect execution.
- **Mitigations:** The process relies on the best practices followed by those responsible for the `dependencyManager` role to ensure it is always proposed and executed correctly.
- **Detection:** After the upgrade, verify that the `OptimismPortal` does not hold any balance. Also, the `SharedLockbox` must have included the `OptimismPortal` in its allow list. Implement scripts at every step of the process to track all related events and verify their completion.
- **Recovery Path(s):** The recovery approach depends on the specific scenario. The table below outlines possible outcomes and recovery strategies when a batch transaction maintains the expected order but omits certain calls:
    
    
    | Case N | Step 1: Allow new chain through `addDependency`  | Step 2: Call `upgradeAndCall` from `ProxyAdmin` to upgrade `OptimismPortal` to `LiquidityMigrator` to execute the migration of ETH. | Step 3: Call `upgrade` to update `OptimismPortal` to its final implementation. | Expected result | Recovery path |
    | --- | --- | --- | --- | --- | --- |
    | Case 1 | yes | yes | yes | Migration completed | - |
    | Case 2 | yes | no | no | ETH deposits will continue to be made into the `OptimismPortal`, but withdrawals may fail if the contract lacks sufficient funds. This issue can also arise in other `OptimismPortal`contracts within the op-interop set, as ETH flow between chains can cause imbalances, leaving some portals underfunded. | Execute a batch tx with the steps 2 and 3. |
    | Case 3 | no | yes | no | The transaction will revert because the `OptimismPortal` was not allow-listed since `addDependency` was not executed first. | Re-execute the complete batch process. No additional issues are expected. |
    | Case 4 | no | no | yes | Deposits and withdrawals will fail because `SharedLockbox`does not allow `OptimismPortal` to call `lockETH` and `unlockETH` since it was not authorized. Consequently, the existing ETH will remain stuck in `OptimismPortal`. | Re-execute the complete batch process. |
    | Case 5 | yes | yes | no | The `OptimismPortal` proxy implementation will remain incorrect (the latest applied implementation would be the `LiquidityMigrator`). As a result, none of the expected functions of the `OptimismPortal` will work. | Re-execute the step 3 only. |
    | Case 6 | yes |  no | yes | `OptimismPortal` will be able to lock and unlock ETH from the `SharedLockbox`, but it will still store its ETH in the `OptimismPortal`. ETH withdrawals are at risk to fail for any `OptimismPortal` in the cluster because a portion of the expected ETH was not migrated and remains in the latest added chain. | Execute a batch TX with the steps 2 and 3. |
    | Case 7 | no | yes | yes | Same as Case 3: The transaction will revert because the `OptimismPortal` was not allow-listed since `addDependency` was not executed first. | Re-execute the complete batch process. |
    
    It’s worth noting that the process is designed to revert if the step 2 (`upgradeAndCall`) is executed before the step 1 (`addDependency`). Consequently, only a few additional cases can be described when the order is also altered.
    
    | Case N | Type | Expected result | Recovery Path |
    | --- | --- | --- | --- |
    | Case 8 | Step 1 → Step 3 → Step 2 | Same as Case 5, the `OptimismPortal` proxy implementation will remain incorrect (the latest applied implementation would be the `LiquidityMigrator`). As a result, none of the expected functions of the `OptimismPortal` will work. | Re-execute the step 3 only. |
    | Case 9 | Step 3 → Step 1 → Step 2 | ETH will be migrated to the `SharedLockbox`, but the `OptimismPortal` proxy implementation will remain incorrect (the latest applied implementation would be the `LiquidityMigrator`). As a result, none of the expected functions of the `OptimismPortal` will work. | Re-execute the step 3 only. |
    | Case 10 | Step 3 → Step 1 (Step 2 missed) | Same as Case 6, `OptimismPortal` will be able to lock and unlock ETH from the `SharedLockbox`, but it will still store its ETH in the `OptimismPortal`. ETH withdrawals are at risk to fail for any `OptimismPortal` in the cluster because a portion of the expected ETH was not migrated and remains in the latest added chain. | Execute a batch TX with the steps 2 and 3. |

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x]  Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- Resolve all the comments.
- Proceed with the code implementation and make any necessary approved changes.
- Move into testing of all the components.
- Make sure there are monitoring solutions ready to cover all the cases.

## Audit Requirements

The `OptimismPortalInterop.sol`, `SharedLockbox.sol`, `LiquidityMigrator.sol`, `SystemConfigInterop.sol`, and `SuperchainConfig.sol` require an audit before going to production. 

## Additional Notes

- Most high-risk cases involving the `SharedLockbox` pertain to the unexpected extraction of ETH liquidity. There are a few possible approaches to minimize the risks of such events:
    - Adding up a timelock to the `finalizeWithdrawalTransaction` function, which would delay fund extraction.
    - Introducing a cooldown period for each newly authorized `OptimismPortal`. During the cooldown, the new portal would be unable to successfully call `unlockETH` and `lockETH`.
    
    In both cases, these time periods would allow the Guardian to pause the system before a malicious attempt is successful.
    
- In the future, the batch transaction process could be replaced by a dedicated batcher contract that predefines the calls and enforces their ordering. This would minimize the risks associated with equivocation in batch execution described above to the lowest possible level.
