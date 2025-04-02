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
- Specs PR: [Access List Validation](https://github.com/ethereum-optimism/specs/pull/612)

## Failure Modes and Recovery Paths

### FM1: Gas Schedule Changes Break Validation

- **Description:** The validation mechanism relies on gas introspection to determine if storage slots are warm. If the EVM's gas schedule changes, particularly around storage access costs, it could break the validation logic.
- **Risk Assessment:** High
  - Potential Impact: Critical. Changes to gas costs could cause all cross-chain message validation to fail or allow invalid messages to pass.
  - Likelihood: Low. While gas schedule changes are rare, they do occur during network upgrades.
- **Mitigations:**
  - Set `WARM_READ_THRESHOLD` conservatively to account for potential fluctuations
  - Add tests that verify validation works across different gas cost scenarios
  - Document gas schedule dependencies clearly for future upgrades
- **Detection:** Monitor for network upgrades that modify storage access gas costs. Run validation tests before accepting upgrades.
- **Recovery Path(s):** Deploy contract upgrade with adjusted thresholds if gas schedule changes significantly.

### FM2: Storage Slot Collision

- **Description:** The checksum calculation for message validation uses storage slots that could potentially collide with other slots from the access list, leading to false validation of messages.
- **Risk Assessment:** High
  - Potential Impact: Critical. Storage collisions could allow unauthorized messages to be validated.
  - Likelihood: Low. Storage slots are carefully calculated using Identifier and message hash parameter. The way both are encoded in the contract’s logic needs to be flawed for this case to happen.
- **Mitigations**
  - Implement robust checksum calculation that minimizes collision risk
  - Add tests verifying no collisions occur with expected storage patterns
  - Document storage layout and slot calculation methodology
- **Detection:** Monitor for unexpected message validations that could indicate storage collisions.
- **Recovery Path(s):** Deploy contract upgrade with revised storage slot calculation if collisions are detected and update ALL off-chain tools and SDKs that generate these hashes on the access list.

### FM3: Access List Manipulation

- **Description:** Malicious actors could attempt to manipulate transaction access lists to force validation of unauthorized messages.
- **Risk Assessment:** High
  - Potential Impact: High. Could allow unauthorized message execution.
  - Likelihood: Medium. Access lists are user-controlled transaction parameters.
- **Mitigations:**
  - Ensure checksum calculation is cryptographically sound
  - Implement proper validation of access list entries
  - Add checks for maximum access list size
- **Detection:** Monitor for unusual patterns in access list usage and message validation attempts.
- **Recovery Path(s):** Pause message processing if manipulation is detected. Deploy fixes for validation logic.

### FM4: Relayer Implementation Errors

- **Description:** Relayers may implement access list construction incorrectly, leading to failed message validation even for valid messages. One possible cause for this could be the checksum calculation not matching between clients and on-chain logic.
- **Risk Assessment:** Medium
  - Potential Impact: Medium. Could cause temporary disruption of cross-chain messaging.
  - Likelihood: Medium. Relayers need to adopt new patterns for access list construction.
- **Mitigations:**
  - Provide clear documentation and examples for relayer implementations
  - Create reference implementations and test suites
  - Add monitoring for failed validation attempts
- **Detection:** Track validation failure rates and patterns across relayers.
- **Recovery Path(s):** Work with relayer implementations to fix access list construction.

### FM5: Compiler Optimizations Break Gas Introspection

- **Description:** Future changes in the Solidity compiler, via-IR pipeline, or optimizer settings could eliminate or modify the `SLOAD` operation used for gas introspection. This could cause the contract to incorrectly determine that a cold storage slot is warm, as the gas cost check would be optimized away.
- **Risk Assessment:** High
  - Potential Impact: Critical. Could allow validation of unauthorized messages by bypassing the access list check entirely.
  - Likelihood: Medium. Compiler optimizations regularly evolve and could unexpectedly affect low-level gas introspection.
- **Mitigations:**
  - Lock compiler version and optimization settings
  - Document compiler dependencies and optimization constraints
- **Detection:**
  - Regular bytecode verification to ensure `SLOAD` operations remain in place
  - Test suite that verifies gas costs match expectations
- **Recovery Path(s):**
  - Deploy contract upgrade with revised gas introspection implementation that cannot be optimized away
  - Roll back to previous compiler version if optimization cannot be disabled

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [ ] FM1: Implement comprehensive gas schedule testing
- [ ] FM1: Document gas schedule dependencies
- [ ] FM2: Create storage layout documentation
- [ ] FM2: Implement slots collision tests
- [ ] FM3: Add access list validation tests
- [ ] FM3: Document security considerations for access list handling
- [ ] FM4: Create relayer implementation guide
- [ ] FM4: Develop relayer test suite
- [ ] FM5: Document compiler version and optimization constraints

## Audit Requirements

The access list validation mechanism in `CrossL2Inbox` requires an audit before production deployment, with particular focus on:

- Gas introspection implementation
- Storage slot calculation and collision resistance
- Access list validation logic
- Integration with existing message passing system
