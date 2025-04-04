# Interop Access List: Failure Modes and Recovery Path Analysis

| Author              | Agusduha, Skeletor-spaceman  |
| ------------------- | ---------------------------- |
| Created at          | 2025-02-05                   |
| Initial Reviewers   | [Initial Reviewer Names]     |
| Needs Approval From | Matt Solomon, Kelvin Fichter |
| Status              | Draft                        |

## Introduction

This document covers the failure modes and recovery paths for the access list validation mechanism in cross-chain message processing. The access list enhancement aims to prevent denial of service (DoS) attacks and enable cheap pre-validation of cross-chain messages.

The following components are included:

- **Contracts**:
  - Updates to `CrossL2Inbox`: Implements gas introspection to validate `ExecutingMessage` transactions
  - Storage slot warming mechanism for message validation
  - Checksum calculation and verification logic

Below are references for this project:

- Design PR: [Access List design](https://github.com/ethereum-optimism/design-docs/pull/214)
- Specs PR: [Access List spec](https://github.com/ethereum-optimism/specs/pull/612)

## Failure Modes and Recovery Paths

### FM1: Gas Introspection Breaks Due to EVM Changes or Compiler Optimizations

- **Description:** The validation mechanism relies on gas introspection to determine if storage slots are warm. This can break in two ways:
  1. If the EVM's gas schedule changes, particularly around storage access costs, it could break the validation logic
  2. Future changes in the Solidity compiler, via-IR pipeline, or optimizer settings could eliminate or modify the `SLOAD` operation used for gas introspection, causing the contract to incorrectly determine that a cold storage slot is warm
- **Risk Assessment:** High
  - Potential Impact: Critical. Changes to gas costs or optimized-away operations could cause all cross-chain message validation to fail or allow invalid messages to pass.
  - Likelihood: Medium. While gas schedule changes are rare, they do occur during network upgrades. Additionally, compiler optimizations regularly evolve and could unexpectedly affect low-level gas introspection.
- **Mitigations:**
  - Set `WARM_READ_THRESHOLD` conservatively to account for potential fluctuations
  - Lock compiler version and optimization settings
  - Document compiler dependencies, optimization constraints, and gas schedule dependencies
  - Add tests that verify validation works across different gas cost scenarios
  - Regular bytecode verification to ensure `SLOAD` operations remain in place
- **Detection:**
  - Monitor for network upgrades that modify storage access gas costs
  - Regular bytecode verification to ensure `SLOAD` operations remain in place
  - Test suite that verifies gas costs match expectations
  - Run validation tests before accepting upgrades
- **Recovery Path(s):**
  - Deploy contract upgrade with adjusted thresholds if gas schedule changes significantly
  - Roll back to previous compiler version if optimization cannot be disabled

### FM2: Storage Slot Collision

- **Description:** The checksum calculation for message validation uses storage slots that could potentially collide with other slots from the access list, leading to false validation of messages.
- **Risk Assessment:** High
  - Potential Impact: Critical. Storage collisions could allow unauthorized messages to be validated.
  - Likelihood: Low. Storage slots are carefully calculated using Identifier and message hash parameter. The way both are encoded in the contract's logic needs to be flawed for this case to happen.
- **Mitigations**
  - Implement robust checksum calculation that minimizes collision risk
  - Add tests verifying no collisions occur with expected storage patterns
  - Document storage layout and slot calculation methodology
- **Detection:** Monitor for unexpected message validations that could indicate storage collisions.
- **Recovery Path(s):** Deploy contract upgrade with revised storage slot calculation if collisions are detected and update ALL off-chain tools and SDKs that generate these hashes on the access list.

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [ ] FM1: Implement comprehensive gas schedule testing
- [ ] FM1: Document gas schedule dependencies
- [ ] FM2: Create storage layout documentation
- [ ] FM2: Implement slots collision tests

## Audit Requirements

The access list validation mechanism in `CrossL2Inbox` requires an audit before production deployment, with particular focus on:

- Gas introspection implementation
- Storage slot calculation and collision resistance
- Access list validation logic
- Integration with existing message passing system
