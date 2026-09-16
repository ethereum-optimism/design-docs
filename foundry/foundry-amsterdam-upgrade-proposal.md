# Foundry Version Upgrade Proposal

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Upgrade Info](#upgrade-info)
- [Changelog Links](#changelog-links)
- [Benefits to the OP Stack](#benefits-to-the-op-stack)
- [Notable Features](#notable-features)
- [Notable Bug Fixes](#notable-bug-fixes)
- [Additional Notes](#additional-notes)
  - [Risks and alternatives](#risks-and-alternatives)
  - [Rollout and rollback](#rollout-and-rollback)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

|                     |                  |
| ------------------- | ---------------- |
| Author              | Maurelian        |
| Created at          | 2026-09-11       |
| Needs Approval From | MDS; TODO: second Security team reviewer |
| Status              | READY FOR REVIEW |

## Upgrade Info

Propose **Foundry v1.8.3 for Forge, Cast, Anvil, and Chisel in both optimism and superchain-ops**.

v1.8.3 was released Tuesday, September 15, 2026. The [release notes](https://github.com/foundry-rs/foundry/releases/tag/v1.8.3) explain that v1.8.2 was left unpublished because of RUSTSEC-2026-0285; v1.8.3 ships patched binaries.

Pins verified on September 11, 2026, from local checkouts and read-only GitHub access:

| Surface                                                                                                                                                                                                                                                                               | Current                                          | Proposed                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ----------------------------- |
| [optimism Forge/Cast/Anvil](https://github.com/ethereum-optimism/optimism/blob/f01cf91d8c1b372b98d7bf4b556c65846d7131a0/mise.toml)                                                                                                                                                    | v1.2.3                                           | v1.8.3                        |
| [op-deployer Forge](https://github.com/ethereum-optimism/optimism/blob/f01cf91d8c1b372b98d7bf4b556c65846d7131a0/op-deployer/pkg/deployer/forge/version.json)                                                                                                                          | v1.2.3                                           | v1.8.3                        |
| [superchain-ops Forge/Cast/Anvil](https://github.com/ethereum-optimism/superchain-ops/blob/b960159ea70a9479d579532efb3776dccf73cdc1/mise.toml)                                                                                                                                        | v1.1.0                                           | v1.8.3                        |
| Chisel in both repos                                                                                                                                                                                                                                                                  | No explicit pin in the inspected mise.toml files | Add v1.8.3 pin and tool alias |
| forge-std: [optimism](https://github.com/ethereum-optimism/optimism/tree/f01cf91d8c1b372b98d7bf4b556c65846d7131a0/packages/contracts-bedrock/lib/forge-std) / [superchain-ops](https://github.com/ethereum-optimism/superchain-ops/tree/b960159ea70a9479d579532efb3776dccf73cdc1/lib) | `6853b9ec` / `3b20d60d`                          | `88c8a288` in both            |

Source links pin the inspected repository revisions. The Glamsterdam branch at `34a70ad9093ad7e9029cb0b855f521582e422671` uses tested Forge pin `nightly-4306ec60f947ac20cf1c069366a1ff870b79fe82` with the proposed forge-std revision. This is preliminary evidence; validation must be repeated on released v1.8.3. Builds, tests, and simulations were not run while drafting.

## Changelog Links

Review includes the additional v1.1.0-to-v1.2.3 interval for superchain-ops.

| Interval                                          | Release notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| superchain-ops baseline through optimism baseline | [v1.1.0](https://github.com/foundry-rs/foundry/releases/tag/v1.1.0), [v1.2.0](https://github.com/foundry-rs/foundry/releases/tag/v1.2.0), [v1.2.1](https://github.com/foundry-rs/foundry/releases/tag/v1.2.1), [v1.2.2](https://github.com/foundry-rs/foundry/releases/tag/v1.2.2), [v1.2.3](https://github.com/foundry-rs/foundry/releases/tag/v1.2.3)                                                                                                                                           |
| v1.3 series                                       | [v1.3.0](https://github.com/foundry-rs/foundry/releases/tag/v1.3.0), [v1.3.1](https://github.com/foundry-rs/foundry/releases/tag/v1.3.1), [v1.3.2](https://github.com/foundry-rs/foundry/releases/tag/v1.3.2), [v1.3.3](https://github.com/foundry-rs/foundry/releases/tag/v1.3.3), [v1.3.4](https://github.com/foundry-rs/foundry/releases/tag/v1.3.4), [v1.3.5](https://github.com/foundry-rs/foundry/releases/tag/v1.3.5), [v1.3.6](https://github.com/foundry-rs/foundry/releases/tag/v1.3.6) |
| v1.4 series                                       | [v1.4.0](https://github.com/foundry-rs/foundry/releases/tag/v1.4.0), [v1.4.1](https://github.com/foundry-rs/foundry/releases/tag/v1.4.1), [v1.4.2](https://github.com/foundry-rs/foundry/releases/tag/v1.4.2), [v1.4.3](https://github.com/foundry-rs/foundry/releases/tag/v1.4.3), [v1.4.4](https://github.com/foundry-rs/foundry/releases/tag/v1.4.4)                                                                                                                                           |
| v1.5 and v1.6 candidate                           | [v1.5.0](https://github.com/foundry-rs/foundry/releases/tag/v1.5.0), [v1.5.1](https://github.com/foundry-rs/foundry/releases/tag/v1.5.1), [v1.6.0-rc1](https://github.com/foundry-rs/foundry/releases/tag/v1.6.0-rc1) (release candidate, not a stable recommendation)                                                                                                                                                                                                                            |
| v1.7 series                                       | [v1.7.0](https://github.com/foundry-rs/foundry/releases/tag/v1.7.0), [v1.7.1](https://github.com/foundry-rs/foundry/releases/tag/v1.7.1)                                                                                                                                                                                                                                                                                                                                                          |
| v1.8 series                                       | [v1.8.0](https://github.com/foundry-rs/foundry/releases/tag/v1.8.0), [v1.8.1](https://github.com/foundry-rs/foundry/releases/tag/v1.8.1), [v1.8.3](https://github.com/foundry-rs/foundry/releases/tag/v1.8.3)                                                                                                                                                                                                                                                                                     |

## Benefits to the OP Stack

Forge v1.2.3 cannot run the Amsterdam EVM tests needed for Glamsterdam. The [glamsterdam test](https://github.com/ethereum-optimism/optimism/compare/jm%2Fglamsterdam-test?expand=1) branch's `just test-amsterdam` recipe needs the newer execution engine and state-gas result. These checks help establish whether L1 deployment and upgrade paths remain within the applicable gas limits.

The `superchain-ops` repo also imports monorepo contracts and uses Forge to simulate governance operations, so its simulations must be validated alongside optimism's tests and op-deployer.

## Notable Features

- **Gas accounting.** [v1.8.3](https://github.com/foundry-rs/foundry/releases/tag/v1.8.3) adds Amsterdam-specific fixes needed for monorepo testing, including the nested CREATE gas-accounting fix linked below.
- **Changed defaults.** [v1.8.0](https://github.com/foundry-rs/foundry/releases/tag/v1.8.0) enables isolate mode and dynamic test linking by default. Check gas snapshots, CREATE-derived addresses, and script-output parsing against the current toolchain.
- **Formatting.** [v1.4.0](https://github.com/foundry-rs/foundry/releases/tag/v1.4.0) replaced the formatter with a Solar-based implementation. The experimental branch skips some formatting checks under nightly Forge. Review formatting-only changes separately, measure the diff, and restore all checks using the approved version before adoption.

## Notable Bug Fixes

- [bug(forge): vm.skip in setUp reported as FAIL instead of SKIP when setUp does substantial work (regression)](https://github.com/foundry-rs/foundry/issues/16197)
- [bug(forge): nested CREATE runs out of gas under Amsterdam with EIP-7825 cap enabled](https://github.com/foundry-rs/foundry/issues/16602)

## Additional Notes

### Risks and alternatives

The main risk is adopting a new release without the normal observation period. Given the imminent need for Amsterdam EVM testing and the required fixes in v1.8.3, this proposal requests an exception to the [Foundry policy](https://github.com/ethereum-optimism/optimism/blob/f01cf91d8c1b372b98d7bf4b556c65846d7131a0/packages/contracts-bedrock/book/src/policies/foundry-upgrades.md)'s three-month waiting period. Adoption requires an explicit decision on that exception and unanimous approval from at least two Security team members. Waiting the full period would delay adoption.

### Rollout and rollback

After approval and passing compatibility checks, merge PRs updating Forge, Cast, Anvil, and Chisel to v1.8.3 in both repos, including new Chisel pins and op-deployer's Forge version and checksums. These PRs will introduce the new binaries as well as update formatting and
any other test changes required to support updated Foundry behavior. At this point rollback should be fairly straightforward with a reverting PR, but will become more difficult over time.

A follow-up PR will then be needed to run the L1 test suite against the Amsterdam EVM. This PR will be larger and make more significant changes that will be more likely to create conflicts in an attempted future revert.
