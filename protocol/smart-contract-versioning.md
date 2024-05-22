# Purpose

This design doc proposes a new smart contracts versioning scheme to improve clarity and more closely align with typical semantic versioning practices.

# Summary

The proposed versioning scheme for contracts will follow a true semver format, with major, minor, and patch versions, as well as beta, release candidate, and feature tags.
The monorepo releases will also follow semver, with the version bump determined by the highest version bump of any individual contract modified in the release.

# Problem Statement + Context

There are two facets to contract versioning: the versioning of monorepo releases, and the versioning of individual contracts.

- **Monorepo Releases**:  The current versioning scheme of monorepo contract releases does not follow a true semver. For example, the Fault Proofs release introduced breaking changes by changing the ABI, and we are releasing this as a minor version bump from `op-contracts/v1.3.0` to `op-contracts/v1.4.0`. This was not an intentional decision. We likely slid into this path because node releases cannot bump the major version (for go-specific reasons), and our current [contracts release process](https://www.notion.so/oplabs/Minimal-Contracts-Release-Process-ef8e91b1b13944e7a7f632a824895764?pvs=4) does not specify how to determine the version bump for an upcoming monorepo release.
- **Individual Contracts**: The current versioning scheme of individual contracts is also not a true semver, but in a different way. We do bump major and minor versions appropriately, but we don't distinguish when a contract is in beta, release candidate, or feature status. This makes it difficult to determine the stability and compatibility of a given contract version.

These have resulted in various issues, such as chains deploying in-development versions of contracts.

Unlike typical software development, OP Stack chains cannot rely on "compatible semvers" to compose the chain of various contracts.
Instead, a given monorepo contracts release must be defined by an exact set of contract versions, not a range of semver-compatible versions.

# Alternatives Considered

1. Keep the current versioning scheme: This alternative would maintain the status quo but would not address the issues of clarity, consistency, and alignment with semver practices.

2. Use semver for contracts but not for monorepo releases: This alternative would improve the versioning of individual contracts but would not provide a clear mapping between monorepo releases and contract versions.

# Proposed Solution

The proposed solution is to more closely adopt a true semver format for both individual contracts and monorepo releases.

We preserve the existing rules for when to bump major, minor, and patch versions in individual contracts:

- `patch` releases are to be used only for changes that do NOT modify contract bytecode (such as updating comments).
- `minor` releases are to be used for changes that modify bytecode OR changes that expand the contract ABI provided that these changes do NOT break the existing interface.
- `major` releases are to be used for changes that break the existing contract interface OR changes that modify the security model of a contract.

Versioning for individual contracts:

- A contract is only `X.Y.Z` on `develop` if it's been governance approved. If it's `X.Y.Z` before that, it must be on a branch. More on this below.
- For contracts undergoing development, a `-beta.n` identifier must be appended to the version number.
- For contracts in a release candidate state, an `-rc.n` identifier must be appended to the version number.
- For contracts with feature-specific changes, a `+feature` identifier must be appended to the version number. See [ethereum-optimism/design-docs#8](https://github.com/ethereum-optimism/design-docs/pull/8) to learn more.
- When making changes to a contract, always bump to the lowest possible version based on the specific change you are making. We do not want to e.g. optimistically bump to a major version, because protocol sequencing may change unexpectedly. Use these examples to know how to bump the version:
  - Example 1: A contract is currently on `1.2.3`. We don't yet know when the next release of this contract will be. However, you are simply fixing typos in comments so you bump the version to `1.2.4-beta.1`.
    - The next PR that makes in that same contract but makes no functional changes, so it bumps the version to `1.2.4-beta.2`.
    - The last PR introduces a breaking change, which bumps the version from `1.2.4-beta.2` to `2.0.0-beta.1`.
  - Example 2: A contract is currently on `2.4.7`. We know the next release of this contract will be a breaking change. Regardless, as you start development by fixing typos in comments, bump the version to `2.4.8-beta.1`. This is because we may end up putting out a release before the breaking change is added.
    - Once you start working on the breaking change, bump the version to `3.0.0-beta.1`.
    - Continue to bump the beta version as you make changes. When the contract is ready for release, bump the version to `3.0.0-rc.1`.
- New contracts start at `0.y.z` and only become `1.0.0` when they are ready for production.

Versioning for monorepo releases:

- Monorepo releases continue to follow the `op-contracts/vX.Y.Z` naming convention.
- The version used for the next release is determined by the highest version bump of any individual contract in the release.
  - Example 1: The monorepo is at `op-contracts/v1.5.0`. Various tech debt is cleaned up in contracts, but no features are added, so all contracts only bump the patch version. The next monorepo release will be `op-contracts/v1.5.1`.
  - Example 2: The monorepo is at `op-contracts/v1.5.1`. Legacy `ALL_CAPS()` getter methods are removed from a contract, causing that contract to bump the major version. The next monorepo release will be `op-contracts/v2.0.0`.
- A monorepo contracts release must map to an exact set of contract semvers, and this mapping must be defined in the contract release notes which are the source of truth. See [`op-contracts/v1.4.0-rc.4`](https://github.com/ethereum-optimism/optimism/releases/tag/op-contracts%2Fv1.4.0-rc.4) for an example of what release notes should look like.

The above gets us most of the way there, and there is one problem not yet addressed: When a release is proposed to governance, we propose a commit hash, and often those proposed contracts are already deployed to mainnet.
For example, the [Fault Proofs governance proposal](https://gov.optimism.io/t/upgrade-proposal-fault-proofs/8161) provides specific addresses that will be used.
This is ideal and we should continue to do that, but we need to fit this in to the above scheme.

To accommodate this, once ready for governance approval the release flow is:

- On the `develop` branch, bump the version of all contracts to their respective `X.Y.Z-rc.n`. The `X.Y.Z` here refers to the contract-specific versions so differs per-contract, whereas the `.rc.n` is the same across all contracts.
- Branch off of `develop` and create a branch name `proposal/op-contracts/vX.Y.Z`. Here, `X.Y.Z` is the new version of the monorepo release.
- On this branch, create a new commit that removes the `-rc.n` from all contracts. This is the commit hash that will be used to deploy the contracts and proposed to governance.
- Once governance approves the proposal, merge the proposal branch into `develop` and bump the version of all contracts to their respective `X.Y.Z`. When there are conflicts, resolve them in favor of the `develop` branch. In other words:
  - If a contract has not had any changes since the proposal, the version should be bumped to `X.Y.Z`.
  - If a contract has had changes since the proposal, the version should be stay as whatever is the latest version on `develop`.

Lastly, we will start maintaining a CHANGELOG for contract releases.
The next expected release version will have a tracking issue that documents the new versions of each contract that will be included in the release, along with the links to the PRs that made the changes.
Every contracts PR should have an accompanying changelog entry in the tracking issue once it is merged.
See [ethereum-optimism/optimism#10592](https://github.com/ethereum-optimism/optimism/issues/10592) for an example of what this tracking issue should look like.

If this design doc is approved, a PR to the [style guide](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/STYLE_GUIDE.md#versioning) will be opened to reflect these changes.

# Risks & Uncertainties

We need to decide how to enforce this new versioning scheme:

- Manually: We must be diligent about manually enforcing the new scheme on PRs, as the existing scheme has been in place for a long time and it will be easy to forget to follow the new scheme.
- Automatically: We can default to having a semgrep check fail if a PR introduces an `X.Y.Z` version without a `-beta.n` or `-rc.n` identifier. This check should be bypassable with a special comment to allow for exceptions when the contract becomes part of an approved release. This is the recommended approach.
