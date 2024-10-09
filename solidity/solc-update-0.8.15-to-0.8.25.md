# Solidity Version Upgrade Proposal

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Upgrade Info](#upgrade-info)
- [Benefits to the OP Stack](#benefits-to-the-op-stack)
    - [DevX](#devx)
    - [Gas Optimization](#gas-optimization)
    - [Features](#features)
- [Notable Features](#notable-features)
- [Notable Bug Fixes](#notable-bug-fixes)
- [Changelog Links](#changelog-links)
- [Action Items](#action-items)
- [Additional Notes](#additional-notes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |                         |
| ------------------- | ----------------------- |
| Author              | Michael Amadi           |
| Created at          | 2024-10-02              |
| Needs Approval From | Kelvin Fichter          |
| Other Reviewers     | Matt Solomon, Maurelian |
| Status              | In Review               |

## Upgrade Info

- **Current Version:** 0.8.15, 0.8.19, 0.8.25.
- **Proposed Version:** 0.8.25.

## Benefits to the OP Stack

#### DevX

- Allow defining custom operators for user-defined value types via using {f as +} for T global syntax (>=0.8.19): This greatly improves type safety.
- Add support for NatSpec documentation in enum and struct definitions. (>= 0.8.20): Improves documentation and developer experience
- Include NatSpec from events that are emitted by a contract but defined outside of it in userdoc and devdoc output (>= 0.8.20)
- Allow qualified access to events from other contracts. (>= 0.8.21)

#### Gas Optimization

- Unchecked loop increments (>= 0.8.22)
- Use `mcopy` instead of `mload`/`mstore` loop when copying byte arrays (>= 0.8.25)
- Preventing Dead Code in Runtime Bytecode (>=0.8.19): This also facilitates slightly larger contract sizes
- Disabling CBOR metadata (>=0.8.18): Allows omitting the CBOR metadata section from the bytecode. Also facilitates larger contract sizes.

#### Features

- Relax restrictions on initialization of immutable variables. Reads and writes may now happen at any point at construction time outside of functions and modifiers. Explicit initialization is no longer mandatory (>= 0.8.21)
- Support for the EVM Version "Cancun" (>= 0.8.24): Introduces global `block.blobbasefee`, `blobhash(uint)`, `blobbasefee()`, `blobhash()`, `mcopy()`, `tload()` and `tstore()`.

## Notable Features

- Deprecate support for "homestead", "tangerineWhistle", "spuriousDragon" and "byzantium" EVM versions: This might need special attention if any contracts are required to be compiled with any of the deprecated EVM versions. This won't be possible with the v0.8.25.

## Notable Bug Fixes

- [Head Overflow Bug in Calldata Tuple ABI-Reencoding](https://soliditylang.org/blog/2022/08/08/calldata-tuple-reencoding-head-overflow-bug/) (Bug fixed in v0.8.16)
- [Storage write removal before conditional termination](https://soliditylang.org/blog/2022/09/08/storage-write-removal-before-conditional-termination/) (Bug fixed in v0.8.17)
- [FullInliner Non-Expression-Split Argument Evaluation Order Bug](https://soliditylang.org/blog/2023/07/19/full-inliner-non-expression-split-argument-evaluation-order-bug/) (Bug fixed in v0.8.21)

## Changelog Links

- [Solidity 0.8.16 Release Announcement](https://soliditylang.org/blog/2022/08/08/solidity-0.8.16-release-announcement/)
- [Solidity 0.8.17 Release Announcement](https://soliditylang.org/blog/2022/09/08/solidity-0.8.17-release-announcement/)
- [Solidity 0.8.18 Release Announcement](https://soliditylang.org/blog/2023/02/01/solidity-0.8.18-release-announcement/)
- [Solidity 0.8.19 Release Announcement](https://soliditylang.org/blog/2023/02/22/solidity-0.8.19-release-announcement/)
- [Solidity 0.8.20 Release Announcement](https://soliditylang.org/blog/2023/05/10/solidity-0.8.20-release-announcement/)
- [Solidity 0.8.21 Release Announcement](https://soliditylang.org/blog/2023/07/19/solidity-0.8.21-release-announcement/)
- [Solidity 0.8.22 Release Announcement](https://soliditylang.org/blog/2023/10/25/solidity-0.8.22-release-announcement/)
- [Solidity 0.8.23 Release Announcement](https://soliditylang.org/blog/2023/11/08/solidity-0.8.23-release-announcement/)
- [Solidity 0.8.24 Release Announcement](https://soliditylang.org/blog/2024/01/26/solidity-0.8.24-release-announcement/)
- [Solidity 0.8.25 Release Announcement](https://soliditylang.org/blog/2024/03/14/solidity-0.8.25-release-announcement/)

## Action Items

- [ ] Contracts compile successfully with upgraded solidity version and without stack too deep error(s)
- [ ] Contracts compile successfully with upgraded solidity version and without any contract intended to be deployed exceeding the contract code size limit (24576 bytes)

## Additional Notes

- **Stack too deep errors during compilation**: Current OP Stack contracts only compile with the legacy pipeline with the optimizer on. Compiling with the optimizer off or using the via-ir pipeline result in stack too deep errors. When upgrading to a new solidity version, we have to make sure that successful compilation using the current build settings is possible.
- **Contracts exceeding code size when compiled**: There are a few OP Stack contracts that are close to the contract code size limit. We have to ensure that the chosen compiler version to upgrade to compiles all OP Stack contracts successfully and without any exceeding the contract code size limit.
