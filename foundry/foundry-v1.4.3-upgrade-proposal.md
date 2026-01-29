# Foundry Version Upgrade Proposal

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Upgrade Info](#upgrade-info)
- [Changelog Links](#changelog-links)
- [Benefits to the OP Stack](#benefits-to-the-op-stack)
  - [1. Performance Improvements](#1-performance-improvements)
  - [2. Code Quality and Security Tools](#2-code-quality-and-security-tools)
  - [3. Advanced Testing Capabilities](#3-advanced-testing-capabilities)
  - [4. EIP-712 Improvements (v1.3.0)](#4-eip-712-improvements-v130)
  - [5. Solar Formatter (v1.4.0)](#5-solar-formatter-v140)
  - [6. Osaka Hardfork Support (v1.4.0)](#6-osaka-hardfork-support-v140)
- [Notable Features](#notable-features)
  - [v1.3.0 Features Requiring Attention](#v130-features-requiring-attention)
  - [v1.4.0 Features Requiring Attention](#v140-features-requiring-attention)
- [Notable Bug Fixes](#notable-bug-fixes)
  - [Security-Relevant Fixes](#security-relevant-fixes)
  - [Test Reliability Fixes](#test-reliability-fixes)
- [Additional Notes](#additional-notes)
  - [Implementation Checklist](#implementation-checklist)
  - [Risk Assessment](#risk-assessment)
  - [Compatibility](#compatibility)
  - [Community Adoption](#community-adoption)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |                                                    |
| ------------------- | -------------------------------------------------- |
| Author              | Michael Amadi                                      |
| Created at          | 2026-01-23                                         |
| Needs Approval From | Matt Solomon, Kelvin Fichter                       |
| Status              | In Review                                          |

## Upgrade Info

- **Current Version:** v1.2.3
- **Proposed Version:** v1.4.3

**Release Timeline:**
- Current version (v1.2.3) released: 2025-06-08
- Proposed version (v1.4.3) released: 2025-10-22

## Changelog Links

The following versions were released between v1.2.3 and v1.4.3:

**Major Releases:**
- [v1.3.0](https://github.com/foundry-rs/foundry/releases/tag/v1.3.0) (2025-07-31)
- [v1.4.0](https://github.com/foundry-rs/foundry/releases/tag/v1.4.0) (2025-10-08)

**Patch Releases:**
- [v1.3.1](https://github.com/foundry-rs/foundry/releases/tag/v1.3.1) (2025-08-12)
- [v1.3.2](https://github.com/foundry-rs/foundry/releases/tag/v1.3.2) (2025-08-21)
- [v1.3.3](https://github.com/foundry-rs/foundry/releases/tag/v1.3.3) (2025-08-29)
- [v1.3.4](https://github.com/foundry-rs/foundry/releases/tag/v1.3.4) (2025-09-03)
- [v1.3.5](https://github.com/foundry-rs/foundry/releases/tag/v1.3.5) (2025-09-08)
- [v1.3.6](https://github.com/foundry-rs/foundry/releases/tag/v1.3.6) (2025-09-16)
- [v1.4.1](https://github.com/foundry-rs/foundry/releases/tag/v1.4.1) (2025-10-14)
- [v1.4.2](https://github.com/foundry-rs/foundry/releases/tag/v1.4.2) (2025-10-18)
- [v1.4.3](https://github.com/foundry-rs/foundry/releases/tag/v1.4.3) (2025-10-22)

## Benefits to the OP Stack

### 1. Performance Improvements

**Test Execution Speed:**
- Up to **21.50% faster** forge test execution (v1.3.0)
- Up to **11.41% faster** fuzzed test execution (v1.4.0)
- Up to **10.52% faster** forge coverage (v1.4.0)

These improvements directly reduce CI/CD pipeline times for the OP Stack monorepo and reduce developer feedback loops during local testing.

**Forked Test Optimization:**
- Improved Reth client support with `eth_getAccountInfo` API reduces RPC calls from 3 to 1 per account
- Over 5 seconds improvement in forked test execution times
- Particularly beneficial for OP Stack's extensive fork testing against mainnet

### 2. Code Quality and Security Tools

**Forge Lint (v1.3.0):**
- Built-in Solidity linter that runs on `forge build` by default
- Helps catch style issues, unchecked calls, unused imports, and other code quality issues
- Configurable rules with inline comment directives
- Improves code consistency across the OP Stack contracts codebase

### 3. Advanced Testing Capabilities

**Coverage-Guided Fuzzing (v1.3.0):**
- Enables more thorough invariant testing by targeting code paths systematically
- Useful for complex OP Stack contracts like dispute game contracts and cross-domain messaging
- Saves and reuses corpus for deterministic test runs

**Table Tests (v1.3.0):**
- Structured way to test multiple input/output combinations
- Reduces test code duplication
- Makes test intent clearer and easier to maintain

### 4. EIP-712 Improvements (v1.3.0)

- New cheatcodes and utilities for working with EIP-712 signatures
- Relevant for OP Stack's cross-chain signature validation and authorization patterns

### 5. Solar Formatter (v1.4.0)

- Complete formatter rewrite using Solar parser
- More robust and feature-complete formatting
- Better handling of edge cases in complex Solidity code

### 6. Osaka Hardfork Support (v1.4.0)

- Forward compatibility with upcoming Ethereum hardforks
- Ensures OP Stack can test against future network upgrades

## Notable Features

### v1.3.0 Features Requiring Attention

1. **`forge lint` runs by default on `forge build`**
   - **Impact:** May cause builds to fail if linting errors are present
   - **Mitigation:** Can be disabled with `lint_on_build = false` in `foundry.toml`
   - **Recommendation:** After upgrading, review any linting errors and either fix them or configure lint rules in `foundry.toml` as needed

2. **Coverage-Guided Fuzzing Mode**
   - **Impact:** Changes fuzzing behavior when `corpus_dir` is configured
   - **Consideration:** Existing invariant tests may behave differently
   - **Recommendation:** Review invariant test configurations and results after upgrade

3. **Updated `revm` dependency (v27)**
   - **Impact:** Core EVM execution engine update
   - **Consideration:** Potential behavioral changes in edge cases
   - **Recommendation:** Full test suite execution required

### v1.4.0 Features Requiring Attention

1. **Solar-based Formatter**
   - **Impact:** Complete formatter rewrite may format code differently
   - **Consideration:** Large formatting diffs in first PR after upgrade
   - **Recommendation:** Run `forge fmt` separately in a dedicated PR to isolate formatting changes

2. **Multi-chain Deployment Enhancements**
   - **Impact:** Improved handling of multiple chain configurations
   - **Benefit:** Directly supports OP Stack's multi-chain deployment needs (superchain-ops)

3. **Deprecated Etherscan v1 API**
   - **Status:** Foundry v1.4.0 deprecated the Etherscan v1 API for contract verification
   - **Impact:** No migration required - OP Stack's `op-deployer` already uses Etherscan v2 API ([source](https://github.com/ethereum-optimism/optimism/blob/213b3981e7507f4103fa7bc2191f977a16bf75ab/op-deployer/pkg/deployer/verify/etherscan.go#L55))
   - **Action:** None needed, included for awareness only

## Notable Bug Fixes

### Security-Relevant Fixes

**v1.3.0:**
- **Fixed `vm.cool` storage cleaning issue** (#10546) - Previously marked storage as cold instead of cleaning it, which could lead to incorrect gas accounting in tests
- **Fixed prank applied on contract creation** (#10532) - Ensures pranks work correctly during contract deployment
- **Fixed EIP-7702 multiple auth handling** (#10623) - Correct handling of authorization in EIP-7702 transactions

**v1.4.0:**
- **Multiple EIP-7702 authorization fixes** - Improved handling of delegated code execution
- **Fork state synchronization fixes** - Ensures correct state when testing against forked networks

**v1.4.3:**
- **Fixed coverage exclusions** (#12202, #12216) - Correctly excludes abstract contracts and virtual functions without implementation from coverage

### Test Reliability Fixes

**v1.3.0:**
- **Fixed `expectEmit` with count 0** (#10534) - No longer reverts when event with count 0 is not emitted
- **Fixed `vm.chainId` regression in isolation mode** (#10589)
- **Fixed persistent storage updates** (#10576)

**v1.4.0:**
- **Improved test backtrace handling** - Better debugging experience for failing tests
- **Fixed coverage item counting** (#11565) - More accurate coverage reporting

**v1.4.3:**
- **Fixed backtrace verbosity** (#12211) - Backtraces only shown at `-vvvvv` level, reducing noise

## Additional Notes

### Implementation Checklist

Per the [Foundry Versioning Policy](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/book/src/policies/foundry-upgrades.md), the following locations must be updated:

- [ ] `mise.toml`: Update `forge`, `cast`, and `anvil` version entries to `1.4.3`
- [ ] `op-deployer/pkg/deployer/forge/version.json`: Update forge version and checksums for all platforms
- [ ] `superchain-ops` repository: Update Foundry version configuration

### Risk Assessment

**Low Risk:**
- v1.4.3 is a stable release that has been in use by the broader Foundry community for 3+ months
- Primarily contains performance improvements and bug fixes
- Changes are well-documented and reversible

**Potential Issues:**
- Linting may surface existing code quality issues that need addressing
- Formatter changes may create large diffs (cosmetic only)
- Small behavioral changes in edge cases due to revm updates

**Mitigation:**
- Thorough testing of the full OP Stack test suite
- Separate PRs for formatting changes
- Review of lint findings before merge

### Compatibility

- Compatible with current Solidity version used in OP Stack (0.8.30 activated as default in v1.3.0)
- Prague hardfork activated as default (v1.3.0)
- Osaka hardfork ready (v1.4.0)

### Community Adoption

v1.4.3 is part of the v1.4.x series which has been widely adopted in the Solidity development community. The 3+ month period since release has allowed for community vetting and bug discovery.
