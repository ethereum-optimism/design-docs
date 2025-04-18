# Interop Access List: Failure Modes and Recovery Path Analysis

| Author              | Agusduha, Skeletor-spaceman  |
| ------------------- | ---------------------------- |
| Created at          | 2025-02-05                   |
| Initial Reviewers   | Mark Tyneway                 |
| Needs Approval From | Matt Solomon, Kelvin Fichter |
| Status              | Final                        |

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
  - Implement end-to-end tests that verify validation reverts work correctly in a real environment
  - Regular bytecode verification to ensure `SLOAD` operations remain in place
- **Detection:**
  - Monitor for network upgrades that modify storage access gas costs
  - Regular bytecode verification to ensure `SLOAD` operations remain in place
  - End-to-end test suite that verifies validation reverts as expected
  - Run validation tests before accepting upgrades
- **Recovery Path(s):**
  - Deploy contract upgrade with adjusted thresholds if gas schedule changes significantly
  - Roll back to previous compiler version if optimization cannot be disabled

### FM2: Storage Slot Collision

- **Description:** The checksum calculation for message validation uses storage slots that could potentially collide with other slots from the access list, leading to false validation of messages.
- **Risk Assessment:** High
  - Potential Impact: Critical. Storage collisions could allow unauthorized messages to be validated. Additionally, this would break the ability to filter invalid executing messages at ingress, creating a spam vector where MEV searchers could attempt arbitrage by submitting invalid messages with no inclusion cost.
  - Likelihood: Extremely Low. Storage slots are calculated using a 248-bit (31 byte) hash space, providing cryptographically secure collision resistance. A collision would require finding a hash collision, which makes it infeasible to have a collision as long as the implementation is correct.
- **Mitigations**
  - Implement robust checksum calculation that minimizes collision risk
  - Add tests verifying no collisions occur with expected storage patterns
  - Document storage layout and slot calculation methodology
- **Detection:** Monitor for unexpected message validations that could indicate storage collisions.
- **Recovery Path(s):** Deploy contract upgrade with revised storage slot calculation if collisions are detected and update ALL off-chain tools and SDKs that generate these hashes on the access list.

### FM3: Unexpected Storage Slot Warming Behavior

- **Description:** The validation mechanism assumes that storage slots are only warmed through the transaction's access list and that warming is rolled back on revert. However, this behavior relies on EVM implementation details that aren't explicitly guaranteed by the protocol. Several potential issues arise:
  1. A slot could be warmed through other means than the access list
  2. The current behavior where slot warming is rolled back on revert isn't guaranteed by the protocol spec - it's an implementation detail that could change, as cached values theoretically could persist after reverts
  3. Storage warming is currently scoped per transaction, but future EVM updates could change this to be per block, which would break the security assumption that each transaction's access list independently controls its warm slots
- **Risk Assessment:** High
  - Potential Impact: Critical. If slots remain warm after reverts, can be warmed through alternative means, or warming persists across transactions in a block, it could allow unauthorized messages to be validated, bypassing the access list requirement entirely.
  - Likelihood: Low. While the current behavior is stable, it relies on implementation details rather than protocol guarantees.
- **Mitigations:**
  - Document reliance on EVM warming behavior in reverts
  - Add tests specifically verifying slot cooling behavior after reverts across multiple EVM clients
  - Document dependency on per-transaction warming scope
  - Monitor EIP proposals that could affect storage warming mechanics
- **Detection:**
  - Regular testing of slot warming behavior across multiple EVM clients, especially after network upgrades
  - Monitor for unexpected successful message validations
  - Track EVM changes that could affect storage slot warming mechanics or scope
  - Test that warm slots from previous transactions in a block don't affect current transaction
- **Recovery Path(s):** Deploy contract upgrade with alternative validation mechanism if EVM warming behavior changes

### Generic items we need to take into account:

See [fma-generic-contracts.md](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md).

- [x] Check this box to confirm that these items have been considered and updated if necessary.

## Action Items

- [x] FM1: Document gas schedule dependencies ([Access List Assumptions](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/predeploys.md#assumptions))
- [x] FM1: Implement comprehensive gas schedule testing ([CrossL2Inbox tests suite](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol))
- [x] FM2: Create storage layout documentation ([Access List Assumptions](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/predeploys.md#assumptions))
- [x] FM2: Implement slots collision tests ([CrossL2Inbox tests suite](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol))
- [x] FM3: Document EVM warming behavior dependencies and transaction-scoped assumptions ([Access List Assumptions](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/predeploys.md#assumptions))
- [x] FM3: Implement comprehensive slot warming behavior tests including multi-transaction tests ([CrossL2Inbox tests suite](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/test/L2/CrossL2Inbox.t.sol))

## Audit Requirements

The access list validation mechanism in `CrossL2Inbox` requires an audit before production deployment, with particular focus on:

- Gas introspection implementation
- Storage slot calculation and collision resistance
- Access list validation logic
- Integration with existing message passing system
