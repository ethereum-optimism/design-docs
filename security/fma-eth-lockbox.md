# ETH Lockbox: Failure Modes and Recovery Path Analysis

| Author              | Joxes, Agusduha              |
| ------------------- | ---------------------------- |
| Created at          | 2025-01-28                   |
| Initial Reviewers   | Josep Bove                   |
| Needs Approval From | Matt Solomon, Kelvin Fichter |
| Status              | Final                        |

## Introduction

This document covers the changes introduced by the addition of the ETH Lockbox design, a singleton contract that stores all ETH liquidity for a given set of interoperable chains. It addresses the [ETH withdrawals problem](https://github.com/ethereum-optimism/specs/issues/362) by the introduction of [SuperchainETHBridge](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/superchain-weth.md). The following components are:

- **Contracts**:
  - Introducing `ETHLockbox`: Stores all ETH liquidity for an interoperable graph. Only authorized `OptimismPortal` addresses can call `lockETH` and `unlockETH`.
  - Updates to the `OptimismPortal2`: Contain the `lockETH` and `unlockETH` functions to handle migrating its ETH liquidity to the `ETHLockbox` and locking/unlocking.

Below are references for this project:

- Specs PR: https://github.com/ethereum-optimism/specs/blob/main/specs/interop/eth-lockbox.md
- Implementation: https://github.com/defi-wonderland/optimism/blob/develop/packages/contracts-bedrock/src/L1/ETHLockbox.sol

Note that the inclusion of the dependency set in the fault-proof mechanism is relevant for the `ETHLockbox`, and it must function prior to the migration process of the contracts (analyzed here) to secure state transitions and thus withdrawals when a chain is added to any dependency set.

> 🗂️
> **For more context about the Interop project, refer to the following docs:**
>
> 1. [Interop System Diagram](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 2. [Interop PID](https://www.notion.so/16c8052fcbb24b93ad1a539b5f8db4c1?pvs=21)
> 3. [Interop Audit Request](https://docs.google.com/document/d/1Rcuzbsguh7koT2jFru5ft9T8zAvjBEzbt0zF5LNQQ08/edit?tab=t.0)

## Failure Modes and Recovery Paths

### FM1: Unauthorized access to `unlockETH` function

- **Description:** If an attacker bypasses access controls for `unlockETH`, the deposit and withdrawal invariants can be broken, potentially resulting in the worst-case scenario of draining all ETH liquidity from the `ETHLockbox`.
- **Risk Assessment:** High.
  - Potential Impact: Critical. Unauthorized ETH unlocking would result in a direct loss of ETH from the `ETHLockbox`.
  - Likelihood: Low. Access controls are specified in `if (!authorizedPortals[msg.sender]) revert Unauthorized()`, but errors or misconfigurations remain possible.
- **Mitigation:** There should be tests for `lockETH` and `unlockETH` checks. Access control must be strictly enforced, permitting only approved `OptimismPortal` contracts via the `authorizedPortals` mapping.
- **Detection:** Monitor `ETHUnlocked` events to verify consistency with authorized portal addresses. Set up alerts for suspected unauthorized activity from non-approved addresses.
- **Recovery Path(s):** Pause the system through `SuperchainConfig` to halt the `ETHLockbox` upon detection of unauthorized access.

### FM2: Solidity Compiler generates an exploitable Bytecode

- **Description:** Smart contracts are typically audited based on their source code, assuming that the Solidity compiler (`solc`) produces a correct and verifiable output. Relying solely on source code audits, without verifying the actual bytecode, could introduce undetected vulnerabilities.
- **Risk Assessment:** High.
  - Potential Impact: Critical. If the Solidity compiler generates incorrect or exploitable bytecode, it could introduce a hidden attack vector.
  - Likelihood: Low. The Solidity compiler is widely used and battle-tested, but compiler bugs have occurred in the past.
- **Mitigations:** Use a Solidity version that has been in production for at least 6 months per the [Solidity Upgrades guidelines](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/meta/SOLIDITY_UPGRADES.md). Actively track Solidity compiler CVEs and from existing resources such as [Solidity Security Alerts](https://soliditylang.org/blog/category/security-alerts/) and [Solidity Bugs Viewer](https://00xsev.github.io/solidityBugsByVersion/) to ensure the chosen compiler version remains robust and secure.
- **Detection:** Monitor for new vulnerabilities in the compiler and for emerging Solidity bugs.
- **Recovery Path(s):** Pause the system if a Solidity compiler bug is identified.

### FM3: Buggy upgrade over the `ETHLockbox`

- **Description:** The `ETHLockbox` is a proxied contract, which means its implementation can be upgraded. While this provides flexibility for fixes and improvements, a buggy or malicious upgrade could introduce vulnerabilities that compromise all ETH liquidity.
- **Risk Assessment:** High.
  - Potential Impact: Critical. If an upgrade introduces a bug or an exploitable backdoor, all ETH stored in the `ETHLockbox` could be drained.
  - Likelihood: Low to Medium. Upgrades are controlled by Security Council, after governance approval. However, upgrade mistakes or compromised keys remain a risk.
- **Mitigations:** Refer to [fma-generic-contracts](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md), items applies to this case. Any new implementation must go through the audit and testing. Upgrades should not be regular in this contract and it should maintain minimal code. Upgrade procedures and keys must follows the proper security practices.
- **Detection:** Verify proxy and implementation contracts through automated checks in the superchain-ops task
- **Recovery Path(s):** Refer to [fma-generic-contracts](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md). Pause the system through `SuperchainConfig` to halt the `ETHLockbox`.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [x] FM1: Provide tests. ([ETHLockbox tests](https://github.com/ethereum-optimism/optimism/blob/5f003211aed7469eed7df666291a62c025d1c46c/packages/contracts-bedrock/test/L1/ETHLockbox.t.sol#L22) and [unlockETH test](https://github.com/ethereum-optimism/optimism/blob/5f003211aed7469eed7df666291a62c025d1c46c/packages/contracts-bedrock/test/L1/ETHLockbox.t.sol#L211))
- [x] FM1: Provide monitoring solutions. (tracked in [Interop Monitoring issue](https://github.com/ethereum-optimism/optimism/issues/15178))
- [x] FM2: Ensure that the security team is aware of how sensitive `ETHLockbox` is to potential compiler bugs.
- [x] FM1, FM2, FM3: Consider potential options for implementing automated pause mechanisms for the `ETHLockbox`, given the impact of these failures modes. (`ETHLockbox` uses `SuperchainConfig` pause mechanism for automated pause)
- [x] Confirm the interop fault proofs are consistent with the ETH Lockbox implementation so that FM discussed are aligned with it and new ones aren’t expected. (`ETHLockbox` is integrated into the portal with the superproofs)

## Audit Requirements

The `OptimismPortal2.sol` and `ETHLockbox.sol` require an audit before production.
