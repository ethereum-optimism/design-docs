# Shared Lockbox: Failure Modes and Recovery Path Analysis

| Author | Joxes, Agusduha |
| --- | --- |
| Created at | 2025-01-28 |
| Initial Reviewers | Josep Bove |
| Needs Approval From | Matt Solomon, Kelvin Fichter |
| Status | In Review |

## Introduction

This document covers the changes introduced by the addition of the Shared Lockbox design, a singleton contract that stores all ETH liquidity for a given set of interoperable chains. It addresses the [ETH withdrawals problem](https://github.com/ethereum-optimism/specs/issues/362) by the introduction of [SuperchainWETH](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/superchain-weth.md). The following components are:

- **Contracts**:
    - Introducing `SharedLockbox`: Stores all ETH liquidity for an interoperable graph. Only authorized `OptimismPortal` addresses can call `lockETH` and `unlockETH`.
    - Introducing `DependencyManager`: A predeploy proxy and implementation contract. Manages the local chain’s dependency set. Triggers an L2 → L1 withdrawal to request adding a new chain to the dependency set on L1 in `SuperchainConfigInterop`.
    - Introducing `SuperchainConfigInterop`: A proxy and implementation contract in L1. Processes add-dependency requests, track the official “mesh” of authorized chains and their portals, and authorizes each chain’s `OptimismPortal` in the `SharedLockbox`.
    - Updates to the `OptimismPortal2`: Contain the `lockETH` and `unlockETH` functions, enabled when the `Sharedlockbox` is used.
    - Updates to the `OptimismPortalInterop`: Extended functionality on top of `OptimismPortal2` to handle migrating its ETH liquidity to the `SharedLockbox` and locking/unlocking.

Below are references for this project:

- Specs PR: https://github.com/ethereum-optimism/specs/pull/465
- Implementation: Refer to this branch https://github.com/defi-wonderland/optimism/tree/sc-feat/add-shared-lockbox

Note that the inclusion of the dependency set in the fault-proof mechanism is relevant for the `SharedLockbox`, and it must function prior to the migration process of the contracts (analyzed here) to secure state transitions and thus withdrawals when a chain is added to any dependency set. The discussion around its design and implementation is still ongoing.

> 🗂️
> **For more context about the Interop project, refer to the following docs:**
> 1. [Interop System Diagram](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: Unauthorized access to `unlockETH` function

- **Description:** If an attacker bypasses access controls for `unlockETH`, the deposit and withdrawal invariants can be broken, potentially resulting in the worst-case scenario of draining all ETH liquidity from the `SharedLockbox`.
- **Risk Assessment:** High.
    - Potential Impact: Critical. Unauthorized ETH unlocking would result in a direct loss of ETH from the `SharedLockbox`.
    - Likelihood: Low. Access controls are specified in `if (!superchainConfig().authorizedPortals(msg.sender)) revert Unauthorized()`, but errors or misconfigurations remain possible.
- **Mitigation:** There should be tests for `lockETH` and `unlockETH` checks. Access control must be strictly enforced, permitting only approved `OptimismPortal` contracts via the `authorizedPortals` mapping in `SuperchainConfigInterop`.
- **Detection:** Monitor `ETHUnlocked` events to verify consistency with authorized portal addresses. Set up alerts for suspected unauthorized activity from non-approved addresses.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox` upon detection of unauthorized access.

### FM2: Compromised or poorly implemented `clusterManager` on L1

- **Description:** The `clusterManager` role in `SuperchainConfigInterop` can directly call `addDependency` on L1, bypassing the normal L2 → L1 flow. If compromised, an attacker could add malicious or arbitrary chain IDs and authorize their own portal.
- **Risk Assessment:** High.
    - Potential Impact: Critical. A malicious addition to the dependency set could allow an attacker to `unlockETH` from the `SharedLockbox`.
    - Likelihood: Low. The `clusterManager` should be a highly secured role (e.g., a multi-sig or governance key).
- **Mitigations:** Tests to ensure `addDependency` cannot be called by unauthorized accounts. Strictly secure the `clusterManager` key.
- **Detection:** Track the `DependencyAdded` event for calls from `clusterManager`. Compare newly added chain IDs to an approved list.
- **Recovery Path(s):** Pause the system through `SuperchainConfig`. Since removal functions are unavailable, an upgrade directly over **`SuperchainConfigInterop`** to clear storage slots will be required.

### FM3: Unauthorized or incorrectly initialized `superchainConfig` address in `OptimismPortalInterop`.

- **Description:** In both `SharedLockbox` and `OptimismPortalInterop`, the address of the `SuperchainConfig` is stored in a special storage slot. If it’s set incorrectly or to a malicious address, the entire access-control scheme breaks.
- **Risk Assessment:** High.
    - Potential Impact: Critical. Since the entire access-control scheme depends on `SuperchainConfig`, all funds could potentially be extracted from the **OptimismPortal**.
    - Likelihood: Low. Storage slots and values are carefully set during protocol upgrades.
- **Mitigations:** There should be tests for the initialization step, and ensuring that `superchainConfig` is only set once.
- **Detection:** Verify the final `superchainConfig` address in newly deployed or upgraded contracts.
- **Recovery Path(s):** Pause the system, to upgrade the contract to set `superchainConfig` correctly.

### FM4: Solidity Compiler generates an exploitable Bytecode

- **Description:** Smart contracts are typically audited based on their **source code**, assuming that the Solidity compiler (`solc`) produces a correct and verifiable output. Relying solely on source code audits, without verifying the actual bytecode, could introduce undetected vulnerabilities.
- **Risk Assessment:** High.
    - Potential Impact: Critical. If the Solidity compiler generates incorrect or exploitable bytecode, it could introduce a hidden attack vector.
    - Likelihood: Low. The Solidity compiler is widely used and battle-tested, but compiler bugs have occurred in the past.
- **Mitigations:** Use a widely audited Solidity version. ****Compare the Keccak-256 hash of the compiled bytecode against an expected reference. Track Solidity compiler CVEs to be aware of past and ongoing issues.
- **Detection:** Verify deployed bytecode against expected hash values before proxy to target the implementation contract.
- **Recovery Path(s):** Pause the system if a Solidity compiler bug is identified.

### FM5: Buggy upgrade over the `SharedLockbox`

- **Description:** The `SharedLockbox` is a proxied contract, which means its implementation can be upgraded. While this provides flexibility for fixes and improvements, a buggy or malicious upgrade could introduce vulnerabilities that compromise all ETH liquidity.
- **Risk Assessment:** High.
    - Potential Impact: Critical. If an upgrade introduces a bug or an exploitable backdoor, all ETH stored in the `SharedLockbox` could be drained.
    - Likelihood: Low to Medium. Upgrades are controlled by Security Council, after governance approval. However, upgrade mistakes or compromised keys remain a risk.
- **Mitigations:** Any new implementation must go through the audit and testing. Upgrades should not be regular in this contract and it should maintain minimal code. Upgrade procedures and keys must follows the proper security practices.
- **Detection:** Verify proxy and implementation contracts
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `SharedLockbox`.

### FM6: Unauthorized use of `DEPOSITOR_ACCOUNT` to call `addDepedency` in `DependencyManager`

- **Description:** On L2, only the `DEPOSITOR_ACCOUNT` can call `addDependency` in `DependencyManager`. If attackers can spoof or impersonate that account, they can push arbitrary additions to the dependency set, eventually letting malicious portals into the `SharedLockbox`.
- **Risk Assessment:** Medium.
    - Potential Impact: High. Attackers could create fake or malicious chain entries that, if get accepted on L1, would authorize an attacker-controlled portal in `SharedLockbox`.
    - Likelihood: Low. The `DEPOSITOR_ACCOUNT` is typically used by the derivation pipeline or otherwise restricted. Previously, there were specific deposit calls that used this account, but those are no longer in use. If the `DEPOSITOR_ACCOUNT` becomes callable in other ways under future upgrades, the attack surface would increase.
- **Mitigations:** There should be tests that ensure `DEPOSITOR_ACCOUNT` only can be called as part of a coordinated upgrade. Review `DEPOSITOR_ACCOUNT` privileges if it is used for other features in the future. Ensure interop fault proof is properly set to reject states that add a non-expected chain.
- **Detection:** Monitor `DependencyAdded` events to catch unexpected additions. Do periodical checks to `dependencySet` to review the existing dependency set recorded in the chain.
- **Recovery Path(s):** Because removing a chain from the dependency set is currently irreversible, a hard fork will be the only way to revert an unauthorized addition. Patch any potential issue around `DEPOSITOR_ACCOUNT`.

### FM7: Unable to add new Chains by Reaching the `uint8` limit in `addDependency`

- **Description:** Both `DependencyManager` and `SuperchainConfigInterop` limit the number of dependencies to `type(uint8).max` (= 255). Once the set size reaches 255, any further `addDependency` call will revert.
- **Risk Assessment:** Low.
    - Potential Impact: Medium. Future expansions of the interoperable set are blocked.
    - Likelihood: Low. 255 chains are already quite large, but could be a problem if the interoperable set wants to grow beyond that.
- **Mitigations:** There should be a defined path to upgrade to increase the type from `uint8` to `uint16` or another approach. Current dependency set size should be monitored.
- **Detection:** A simple periodic check of `dependencySetSize` in both `DependencyManager` and `SuperchainConfigInterop` should be sufficient to ensure the limit is not approaching. Reverts on calls to `addDependency` would indicate this issue.
- **Recovery Path(s):** An upgrade over `SuperchainConfigInterop` and `DependencyManager` is needed to support a larger capacity.

### FM8: `ETHMigrated` flag remains `false` in the `OptimismPortalInterop` after migration

- **Description:** During the `migrateLiquidity` call, the `OptimismPortalInterop` sets an internal `migrated = true` flag. If that flag is never set, the portal incorrectly tries to handle ETH locally (even though liquidity is already in the `SharedLockbox`).
- **Risk Assessment:** Low.
    - Potential Impact: Medium. This situation will cause deposits and withdrawals to fail.
    - Likelihood: Very Low. The logic is straightforward, so only an upgrade or a code bug might cause it.
- **Mitigations:** There should be tests for `migrateLiquidity` to set as `true`.
- **Detection:** Check whether the `migrated` function is marked as `false` after `ETHMigrated` event is emitted. Watch for unexpected reverts on user withdrawals.
- **Recovery Path(s):** Perform an upgrade to set the flag correctly.

### FM9: `addDependency` completed in L2 but `SharedLockbox` not activated, causing withdrawal failures

- **Description:** After an L2 call to `addDependency`, the message must be proven and finalized on L1 to “activate” the chain’s membership in the `SuperchainConfigInterop` and trigger `migrateLiquidity`. If users are allowed to send ETH to the new chain through interop and perform a withdrawal before the liquidity migration is complete, they might see reverts.
- **Risk Assessment:** Low.
    - Potential Impact: Low. ETH withdrawals may fail if they are executed before the migration is completed.
    - Likelihood: Very Low. Withdrawals should be able to finalize in order, so the chance of an ETH withdrawal request being finalized before the migration is executed is quite minimal.
- **Mitigations:** Migration procedures should ensure monitoring, proving, and finalizing the withdrawal transaction once it is possible to do so. Alternatively, chains may choose to delay finalizing cross-chain transfers until the migration is completed.
- **Detection:** Up to the upgrade procedures.
- **Recovery Path(s):** Prove and finalize the withdrawal that executes the migration.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x]  Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [ ]  Resolve all the comments.
- [ ]  FM1: Provide tests.
- [ ]  FM1: Provide monitoring solutions.
- [ ]  FM2: Provide tests.
- [ ]  FM2: Provide monitoring solutions.
- [ ]  FM3: Provide tests.
- [ ]  FM6: Provide tests.
- [ ]  FM6: Provide monitoring solutions.
- [ ]  FM8: Provide tests.
- [ ]  Confirm the interop fault proofs are consistent with the Shared Lockbox and dependency set management implementation so that FM discussed are aligned with it and new ones aren’t expected.

## Audit Requirements

The `OptimismPortal2.sol`, `OptimismPortalInterop.sol`, `DependencyManager.sol` `SharedLockbox.sol`, `SystemConfigInterop.sol`, and `SuperchainConfigInterop.sol` require an audit before production.
