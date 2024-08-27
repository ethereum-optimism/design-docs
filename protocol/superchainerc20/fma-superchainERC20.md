## **SuperchainERC20 FMA (Failure Modes and Recovery Path Analysis)**

| Author | Particle, Skeletor |
| --- | --- |
| Created at | 2024-08-27  |
| Needs Approval From | Matt Solomon, Mark Tyneway |
| Other Reviewers | - |
| Status | Review |

## Introduction

---

This document is intended to be shared publicly for reviews and visibility purposes. It covers the code introduced by

- Updates to the `L2StandardBridge` to allow token conversion.
- Updates to the `OptimismMintableERC20Factory`.
- The `OptimismSuperchainERC20Factory` predeploy.
- `OptimismSuperchainERC20` implementation.
- The `BeaconContract` predeploy used by the `OptimismSuperchainERC20` BeaconProxies.

Below are references for this project:

- The new contracts and updates are documented in the following specs:
    - [Predeploys specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/predeploys.md).
    - [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md)
- Reference implementation and tests can be found in the following PRs: 
    - [Convert and Factories](https://github.com/ethereum-optimism/optimism/pull/11479)
    - [OptimismSuperchainERC20](https://github.com/ethereum-optimism/optimism/pull/11256)

## Failure Modes and Recovery Paths

---

#### Grant access to a malicious or bugged implementation in the `convert` function on the `L2StandardBridge`

- **Description:** If someone can bypass the checks on the token's validity, it would be possible to drain locked liquidity in L1 (e.g., convert a malicious super token into a legit legacy and then withdraw to L1).
- **Risk Assessment:** Medium.
    - Potential Impact: High. The impact of such an attack would vary depending on the token, but significant liquidity is at risk.
    - Likelihood: Low. There are four ways this could happen.
        - A bug in the `_validatePair()` internal function that should only allow implementations deployed by the official factory.
        - A protocol upgrade that allows the writing of malicious or bugged addresses on the `deployments` mapping on the `OptimismMintableERC20Factory` or `OptimismSuperchainERC20Factory`.
        - A malicious or bugged implementation of the token stored in the `BeaconContract`. In particular, upgradable tokens or mintable by a party besides the `L2StandardBridge`.
        - An incorrect `creationCode` in the `OptimismSuperchainERC20Factory`, that does not deploy BeaconProxies that point to the correct `BeaconContract`.
        
        In any of the cases, the bugged implementation must bypass internal security checks. 
- **Mitigations:**
    - The `convert` function is unit, fuzz, e2e and invariant tested with full coverage.
    - Protocol upgrades pose the greatest danger. Before implementing an upgrade, it is essential to check the `_validatePair()` invariants.
- **Detection:** Any off-chain script should maintain its registry of allowed tokens and check for anomalies in the `Converted` even emission. It is possible to introduce a minimum liquidity threshold to minimize off-chain workload, but doing so would also allow for undetected slow drain. 
Moreover, withdrawals to L1 are one of the most commonly tracked variables in Optimism, so the bridge monitoring tools can also detect liquidity anomalies.
- **Recovery Path(s):** Recovery should be similar to any other attack on the `L2StandardBridge`. 
There will be scripts in place to deploy a new implementation, upgrade the implementation in the `BeaconContract`, upgrade the Factories and `L2StandardBridge` implementations.

#### Bypass security checks in `sendERC0` or `relayERC20` functions in `OptimismSuperchainERC20`

- **Description:** The `sendERC20` and `relayERC20` generate and process outbound/inbound cross-chain transactions. The `sendERC20` will burn tokens and generate a cross-chain message. The `relayERC20` will check that the message comes from the `L2toL2CrossDomainMessenger` and the cross-chain caller is the same token address and will then mint the corresponding amount.
If anyone can bypass the controls, they can access the `burn()` and `mint()` functions on the `OptimismSuperchainERC20`.
- **Risk Assessment:** Medium
    - Potential impact: High. This could risk generating liquidity for the `OptimismSuperchainERC20`, which can be converted in the `L2StandardBridge` for its legacy pair and withdrawn to L1.
    - Likelihood: Low. The contract checks seem straightforward. A protocol upgrade affecting the token implementation must go through all security checks. 
    The bridge and token address invariants are well covered in the specs and should be checked.
    There could also be a problem on the message-passing side, i.e., from the `L2toL2CrossDomainMessenger`, the `L2Inbox`or message validation from interop.
- **Mitigation:** The `sendERC20` and `relayERC20` functions are unit, fuzz, e2e and invariant tested within Solidity with full coverage. Any protocol upgrade should test for the bridge invariant. 
Interop testing campaigns should be aware of this attack.
- **Detection:** Liquidity for the most common `OptimismSuperchainERC20` representations will be tracked across the superchain. Scripts should monitor total supply invariants as described in the standard spec.
- **Recovery Path(s):** Scripts will be in place for an emergency deployment of a new implementation and an upgrade of the implementation in the `BeaconContract`.

#### Block legit tokens from converting in the `L2StandardBridge`

- **Description:** The `convert` function could block a valid token on access control. This would effectively block `OptimismSuperchainERC20` from an L1 withdrawal or legacy tokens from using interop.
- **Risk Assessment:** Low.
    - Potential Impact: Low. Funds would be safe.
    - Likelihood: Low. The potential origin of this problem would be the same as those covered in the [first issue](#grant-access-to-a-malicious-or-bugged-implementation-in-the-convert-function-on-the-l2standardbridge): wrong implementation of `_validatePair` or a bugged protocol upgrade. Any error would have to bypass all internal security controls.
- **Mitigation:** The source of this issue will be the same as described in the [first issue](#grant-access-to-a-malicious-or-bugged-implementation-in-the-convert-function-on-the-l2standardbridge), so mitigation steps will also be shared.
- **Detection:** Transactions would fail, but so would simulations. The most likely scenario is that users will report the problem. Support must be aware of this potential scenario.
- **Recovery Path(s):** A protocol upgrade can be written on the `deployments` mapping to allow access for blocked tokens and fix the issue's origin.