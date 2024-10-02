## **SuperchainERC20 standard-only FMA (Failure Modes and Recovery Path Analysis)**

| Author | Ng, Joxes |
| --- | --- |
| Created at | 2024-10-02 |
| Needs Approval From | Mark Tyneway, Matt Solomon, and 0age
 |
| Other Reviewers |  |
| Status | Review |

## Introduction

---

This document is intended to be shared publicly for reviews and visibility purposes. It covers the Introduction of the `SuperchainERC20` standard, including the implementation and interface.

Below are references for this project:

- [Token standard specs](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/token-bridging.md).
- [[DRAFT] Implementation](https://github.com/defi-wonderland/optimism/pull/73).

It does not intend to cover related contracts such as `SuperchainERC20Bridge` or those involving migrated liquidity.

## Failure Modes and Recovery Paths

---

### Unauthorized Access to `__superchainMint` & `__superchainBurn` Functions

- **Description:** The `onlySuperchainERC20Bridge` modifier only allows `__superchainMint` and `__superchainBurn` to be callable by the `SuperchainERC20Bridge`. If the bridge address is badly defined or the modifier bypassed, an entity could mint and burn tokens.
- **Risk Assessment**: Medium.
    - Potential impact: High. All tokens based on this implementation could be potentially at risk.
    - Likelihood: Very Low. `Predeploys.SUPERCHAIN_ERC20_BRIDGE` are defined via protocol upgrades. The modifiers are sufficiently simple and battle-tested to give confidence in the implementation.
- **Mitigation**: Ensure the `SuperchainERC20Bridge` is correctly set during deployment and isn’t subject to unexpected changes.
- **Detection**: Existing off-chain scripts for token monitoring should be enough to detect any unauthorized mint or burn actions triggered by this method.
- **Recovery Path(s)**: Equivocation on `SuperchainERC20Bridge` would require a protocol upgrade or hard fork. Very unlikely to need it.

## Action Items

---

Given the small scope, there is no need for relevant actions beyond resolving all comments, and continuing code implementation and testing.

## Audit Requirements

---

No audit should be required, as it is simple and isn’t expected to have dependencies or impact on other core OP contracts.

## Additional Notes

---

The proposed implementation of the standard doesn’t prevent a token issuer from using other token standards, such as xERC20, with the `SuperchainERC20Bridge` to mint and burn tokens across an [interoperable set of chains](https://specs.optimism.io/interop/overview.html).