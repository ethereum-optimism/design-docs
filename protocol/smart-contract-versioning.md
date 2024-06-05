# Purpose

This design doc proposes a new smart contracts versioning scheme to improve clarity and more closely align with typical semantic versioning practices.

# Summary

The proposed versioning scheme for contracts will follow a true semver format, with major, minor, and patch versions, as well as beta, release candidate, and feature tags.
The monorepo releases will also follow semver, with the version bump determined by the highest version bump of any individual contract modified in the release.

# Problem Statement + Context

There are two aspects to contract versioning: the versioning of monorepo releases, and the versioning of individual contracts.

- **Monorepo Releases**:  The current versioning scheme of monorepo contract releases does not follow a true semver. For example, the Fault Proofs release introduced breaking changes (e.g. by changing the ABI of some contracts), and we are releasing this as a minor version bump from `op-contracts/v1.3.0` to `op-contracts/v1.4.0`. There was not a strong rationale for this—we likely default to a minor bump for consistency with how node releases bump their versions (they cannot bump the major version for go-specific reasons), and our current [contracts release process](https://www.notion.so/oplabs/Minimal-Contracts-Release-Process-ef8e91b1b13944e7a7f632a824895764?pvs=4) does not specify how to determine the version bump for an upcoming monorepo release.
- **Individual Contracts**: The current versioning scheme of individual contracts is also not a true semver, but in a different way. We do bump major and minor versions appropriately, but we don't distinguish when a contract is in beta, release candidate, or feature status. This makes it difficult to determine the stability and compatibility of a given contract version.

These have resulted in various issues, such as users deploying in-development versions of contracts.

Unlike typical software development, OP Stack chains should not rely on "compatible semvers" to compose the chain of various contract at different versions.
Instead, a given monorepo contracts release must be defined by an exact set of contract versions, not a range of semver-compatible versions.

# Alternatives Considered

1. **Keep the current versioning schemes**. Although we are now familiar with the existing versioning schemes, they are not ideal for the reasons given above.

2. **Keep with the current contract versioning scheme but improve the monorepo release versioning scheme**: Because chains should only be deploying their contracts from a tag, we arguably do not need to be as granular with the versioning of individual contracts. However, the existing scheme is confusing because it is nonstandard, and since we are already using a form of semver on the contracts, it is preferable to improve that than to continue with the nonstandard approach. It is also simpler to have a similar versioning scheme for both contracts and monorepo releases instead of two different approaches.

# Proposed Solution

The proposed solution is to more closely adopt a true semver format for both individual contracts and monorepo releases.
There are five parts to the solution:

- [Semver Rules](#semver-rules): Preserves the existing rules for when to bump major, minor, and patch versions in individual contracts.
- [Individual Contract Versioning](#individual-contract-versioning): Introduces a new versioning scheme for individual contracts that includes beta, release candidate, and feature tags.
- [Monorepo Contracts Release Versioning](#monorepo-contracts-release-versioning): Introduces a new versioning scheme for monorepo releases that follows semver.
- [Release Process](#release-process): Describes the process for deploy contracts and proposing a release to governance.
- [Changelog](#changelog): Describes how we will maintain a CHANGELOG for contract releases.

## Semver Rules

We preserve the [existing rules](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/STYLE_GUIDE.md#versioning) for when to bump major, minor, and patch versions in individual contracts:

> - `patch` releases are to be used only for changes that do NOT modify contract bytecode (such as updating comments).
> - `minor` releases are to be used for changes that modify bytecode OR changes that expand the contract ABI provided that these changes do NOT break the existing interface.
> - `major` releases are to be used for changes that break the existing contract interface OR changes that modify the security model of a contract.

Note that bumping the patch version does change the bytecode, so we carve out an exception for that and will update the style guide to reflect this.

## Individual Contract Versioning

Versioning for individual contracts works as follows:

- A contract is only `X.Y.Z` on `develop` if it has been governance approved. If it's `X.Y.Z` before that, it must be on a branch. More on this below.
- For contracts undergoing development, a `-beta.n` identifier must be appended to the version number.
- For contracts in a release candidate state, an `-rc.n` identifier must be appended to the version number.
- For contracts with feature-specific changes, a `+feature-name` identifier must be appended to the version number. See the [Smart Contract Feature Development](https://github.com/ethereum-optimism/design-docs/blob/main/smart-contract-feature-development.md) design doc to learn more.
- When making changes to a contract, always bump to the lowest possible version based on the specific change you are making. We do not want to e.g. optimistically bump to a major version, because protocol sequencing may change unexpectedly. Use these examples to know how to bump the version:
  - Example 1: A contract is currently on `1.2.3`.
    - We don't yet know when the next release of this contract will be. However, you are simply fixing typos in comments so you bump the version to `1.2.4-beta.1`.
    - The next PR made to that same contract clarifies some comments, so it bumps the version to `1.2.4-beta.2`.
    - The last PR introduces a breaking change, which bumps the version from `1.2.4-beta.2` to `2.0.0-beta.1`. A `1.2.4-rc.1` and `1.2.4` version both never exist.
  - Example 2: A contract is currently on `2.4.7`.
    - We know the next release of this contract will be a breaking change. Regardless, as you start development by fixing typos in comments, bump the version to `2.4.8-beta.1`. This is because we may end up putting out a release before the breaking change is added.
    - Once you start working on the breaking change, bump the version to `3.0.0-beta.1`.
    - Continue to bump the beta version as you make changes. When the contract is ready for release, bump the version to `3.0.0-rc.1`.
- New contracts start at `0.y.z` and only become `1.0.0` when they are ready for production.

Below is sample workflow explaining how this works in practice once the release process begins:

- A PR into develop which replaces the `-beta.n` suffix with the `-rc.n` suffix for all contracts in your release. Once this PR is merged, it becomes the last commit to develop for this release until it's governance approved.
  - At this point, the lock on develop is released, and others may modify your `-rc.n` contracts.
- Next, create a branch from that commit on `develop` called `proposal/op-contracts/vX.Y.Z`. Using the `proposal/` prefix signals that this branch is for a governance proposal, and intentionally does not convey whether or not the proposal has passed.
- Open a PR into the `proposal/op-contracts/vX.Y.Z` branch to remove all the `-rc.n` suffixes from contracts. Once merged, this commit is where contracts are deployed, and this commit is tagged as `op-contracts/vX.Y.Z-rc.1` for the governance proposal.
- Subsequent changes only bump the `rc.n` identifier at the monorepo release level. One exception: a contract change results in a higher version level than prior bumps, e.g. if `rc.1` only bumped the minor version, but now we must introduce a breaking change for `rc.2`, we need to bump the major version. Ideally once we're at an `rc.n` this should not occur. This also results in a new branch being created.
- Once the governance proposal passes, create the `op-contracts/vX.Y.Z` tag and release from the latest `op-contracts/vX.Y.Z-rc.n` release, then merge the branch into develop.

## Monorepo Contracts Release Versioning

Versioning for monorepo releases works as follows:

- Monorepo releases continue to follow the `op-contracts/vX.Y.Z` naming convention.
- The version used for the next release is determined by the highest version bump of any individual contract in the release.
  - Example 1: The monorepo is at `op-contracts/v1.5.0`. Clarifying comments are made in contracts, so all contracts only bump the patch version. The next monorepo release will be `op-contracts/v1.5.1`.
  - Example 2: The monorepo is at `op-contracts/v1.5.1`. Various tech debt and code is cleaned up in contracts, but no features are added, so at most, contracts bumped the minor version. The next monorepo release will be `op-contracts/v1.6.0`.
  - Example 3: The monorepo is at `op-contracts/v1.5.1`. Legacy `ALL_CAPS()` getter methods are removed from a contract, causing that contract to bump the major version. The next monorepo release will be `op-contracts/v2.0.0`.
- Feature specific monorepo releases (such as a beta release of the custom gas token feature) are supported, and should follow the guidelines in the [Smart Contract Feature Development](https://github.com/ethereum-optimism/design-docs/blob/main/smart-contract-feature-development.md) design doc. Bump the overall monorepo semver as required by the above rules, and append the `-beta,n` modifier to the version number. For example, if the last release before the custom gas token feature was `op-contracts/v1.5.1`, because the custom gas token introduces breaking changes, its beta release will be `op-contracts/v2.0.0-beta.1`.
  - A subsequent release of the custom gas token feature that fixes bugs and introduces an additional breaking change would be `op-contracts/v2.0.0-beta.2`.
  - This means `+feature-name` naming is not used for monorepo releases, only for individual contracts as described below.
- A monorepo contracts release must map to an exact set of contract semvers, and this mapping must be defined in the contract release notes which are the source of truth. See [`op-contracts/v1.4.0-rc.4`](https://github.com/ethereum-optimism/optimism/releases/tag/op-contracts%2Fv1.4.0-rc.4) for an example of what release notes should look like.

## Release Process

The above gets us most of the way there, and there is one problem not yet addressed: When a release is proposed to governance, we propose a commit hash, and often those proposed contracts are already deployed to mainnet.
For example, the [Fault Proofs governance proposal](https://gov.optimism.io/t/upgrade-proposal-fault-proofs/8161) provides specific addresses that will be used.
This is ideal and we should continue to do that, but we need to fit this in to the above scheme.

To accommodate this, once ready for governance approval the release flow is:

- On the `develop` branch, bump the version of all contracts to their respective `X.Y.Z-rc.n`. The `X.Y.Z` here refers to the contract-specific versions, so it differs per-contract. The `-rc.n` begins as `-rc.1` for all contracts. Any `-beta.n` and `+feature-name` identifiers are removed at this point.
- Branch off of `develop` and create a branch named `proposal/op-contracts/vX.Y.Z`. Here, `X.Y.Z` is the new version of the monorepo release.
- On this branch, create a new commit that removes the `-rc.n` from all contracts. This is the commit hash that will be used to deploy the contracts and proposed to governance.
- Once governance approves the proposal, merge the proposal branch into `develop` and bump the version of all contracts to their respective `X.Y.Z`. When there are conflicts, resolve them in favor of the `develop` branch. In other words:
  - If a contract has not had any changes since the proposal, the version should be bumped to `X.Y.Z`.
  - If a contract has had changes since the proposal, the version should be stay as whatever is the latest version on `develop`.
- If subsequent monorepo `-rc.n` versions are added to the branch, only update the `-rc.n` for the contracts that have changed since the last `-rc.n` version. This is to avoid unnecessary version bumps for contracts that have not changed.
- Fixes to an `-rc.n` version should be made into the `proposal/op-contracts/vX.Y.Z` branch and then merged into `develop` afterwards. In other words, the `proposal/op-contracts/vX.Y.Z` branch behaves like a feature branch only for the duration of the governance proposal.

## Changelog

Lastly, we will start maintaining a CHANGELOG for contract releases:

- Each upcoming release will have a tracking issue that documents the new versions of each contract that will be included in the release, along with links to the PRs that made the changes.
- Every contracts PR must have an accompanying changelog entry in a tracking issue once it is merged.
- Tracking issue titles should be named based on the expected Upgrade number they will go to governance with, e.g. "op-contracts changelog: Upgrade 9".
  - See [ethereum-optimism/optimism#10592](https://github.com/ethereum-optimism/optimism/issues/10592) for an example of what this tracking issue should look like. (Though the title does not currently match this proposal).
  - We don't name it with a version number because it may be hard to predict the final version number of a release until all PRs are merged.
  - Using upgrade numbers also acts as a forcing function to ensure we think about the upgrade sequencing and governance process early in the development process.

If this design doc is approved, a PR to the [style guide](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/STYLE_GUIDE.md#versioning) will be opened to reflect these changes.

# Risks & Uncertainties

We need to decide how to enforce this new versioning scheme. There are two options:

- **Manually**: We must be diligent about manually enforcing the new scheme on PRs, as the existing scheme has been in place for a long time and it will be easy to forget to follow the new scheme.
- **Automatically**: We can default to having a semgrep check fail if a PR to `develop` changes a Solidity file's `X.Y.Z` version without a `-beta.n` or `-rc.n` identifier. This check should be bypassable with a Solidity comment to allow for exceptions when the contract becomes part of an approved release. This is the recommended approach.
