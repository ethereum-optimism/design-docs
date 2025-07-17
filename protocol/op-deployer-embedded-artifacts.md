# OP Deployer Embedded Artifacts

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

We propose embedding the contract artifacts for a given contracts version inside the op-deployer binary itself. This
will reduce the complexity of our deployments, and prevent OP Deployer and the contract artifacts from becoming out
of sync.

## Problem Statement + Context

Today, OP Deployer lets users provide an "artifacts locator" which tells the tool where to find the contract
artifacts to use for deployment. The locator can be a `file://` url, an `https://` url, or a `tag://` URL. File and
HTTPs locators are only used in dev for special cases like unit tests, and therefore are not a concern here. Tagged
locators, however, are used for production deployments and are a huge area of confusion for users for the following
reasons:

- The artifacts that a tagged locator points to are created from a different branch than OP Deployer itself. This
  can lead to a mismatch between OP Deployer's deployment code and the artifacts it is trying to deploy.
- There is a 1:1 mapping between each OP Deployer version and the tagged versions it supports. `v0.2.0` of OP
  Deployer, for example, only supports smart contracts from `op-contracts/v2.0.0`. However, the version of the smart
  contracts can sometimes change by minor or patch versions. This makes it difficult to determine which version of
  the contracts each version of OP Deployer supports, and can lead to unexpected behavior or broken deployments.

In addition, maintaining the tagged locators themselves is non-trivial amount of work:

- New releases require contract artifacts to be built, pushed to GCP, and added to `standard.go`. This means that
  every new contract release requires a corresponding code change to OP Deployer.
- Because the artifacts each locator points to are on GCP, they need to be validated client-side before being used.
  This adds complexity, overhead, and potential for security vulnerabilities.
- It's much less clear which commit a given set of artifacts belong to. Artifacts are built using a
  content-addressable hash of their source code. A commit that doesn't change the source code won't change the hash,
  so the commit for a given contract release may not match the commit in the uploaded artifacts. While not a huge
  issue, this makes determining the provenance of a given set of artifacts quite difficult.

## Proposed Solution

Rather than continuing to support tagged locators, we propose embedding the contract artifacts for a given release
directly inside the OP Deployer binary itself using `go:embed`. HTTPs and file locators will continue to work as
before. The CLI will select the tagged locator by default if it exists, so users will not generally have to specify
a locator at all when invoking OP Deployer.

This change would increase the size of the OP Deployer binary by ~60MB after removing test code from the artifacts.
It is our belief that this is an acceptable tradeoff, particularly since users would have to download the artifacts
anyway.

For local development, we can make the embedding optional. This will avoid increasing Go compile times when we're not
cutting an official OP Deployer release.

As part of this solution, we will start developing OP Deployer on the same proposal branches as the contracts 
themselves. This way, OP Deployer is released side-by-side with the contracts, and we can ensure that they always 
stay in sync.

## Impact on Developer Experience

Tools like Kurtosis will continue to be able to use HTTPs and file locators. If users want to use a tagged locator,
they will need to change the version of OP Deployer they use rather than change the locator itself. This will likely
impact Netchef the most.

## Alternatives Considered

- We considered continuing the status quo, but just developing OP Deployer on the same proposal branches as the
  contracts. This would help reduce the likelihood of the contracts and OP Deployer getting out of sync, but would 
  not resolve the other problems described above.