# Purpose

To outline the steps required to remove the pre-fault proofs contracts from the code base while maintaining support
for chains that rely on the legacy contracts.

# Summary

With the fault proof system approved by Optimism Governance and being shipped to mainnet, we want to remove legacy
code to reduce maintenance costs. However, some chains still build on top of the legacy `L2OutputOracle` (L2OO).
Therefore, we want to make sure these chains have a path forward.

# Problem Statement + Context

The fault proof system was developed in parallel with the legacy `L2OutputOracle` based proposals to ensure that chains
with and without fault proofs could both be supported. Most of the fault proof system is in new contracts such as
`DisputeGameFactory` and `FaultDisputeGame` but there are a few places where multiple implementations were required to
support the with and without fault proof cases:

* `OptimismPortal` is the pre-fp portal and `OptimismPortal2` is the post-fp version.
* `op-proposer` contains support for publishing both via `L2OutputOracle` and the `DisputeGameFactory` via a CLI flag.
* `make devnet-allocs` and `Deploy.s.sol` include a feature toggle to enable or disable fault proofs.
* `L2OutputOracle` is no longer required.

To minimize maintenance costs we now want to consolidate all those cases to just retain the fault proofs version.

# Alternatives Considered

## Maintain the Legacy Code

We could maintain support for the pre-FP `OptimismPortal` and `L2OutputOracle`.

* New features that integrated with the portal would not be available unless we duplicated the development effort.
* Incurs maintenance cost to continue maintaining this code.

# Proposed Solution

Given the long term desire for all chains to utilize fault proofs, deleting the legacy code and utilizing the
permissioned game for chains that currently need L2OO-like functionality is recommended. This removes the legacy code
and associated development complexities from the monorepo codebase, and ensures chains still have a permissioned game
option.

Once the old contracts and associated `Deploy.s.sol` changes are made, we can then rename `OptimismPortal2` to simply
`OptimismPortal`. This is to avoid confusion to users and developers who onboard to the codebase, as seeing an
`OptimismPortal2` without a corresponding `OptimismPortal` is unintuitive.

Deletion of legacy support from other, non-contracts parts of the codebase, such as the proposer, needs to be more
gradual because the latest releases still support the legacy functionality. For these, we should communicate to
stakeholders that we will remove the legacy functionality in the proposer in `x` months, where `x` is based on feedback
from teams that need to migrate.

# Risks & Uncertainties

There may be more chains that rely on the legacy codebase than expected, resulting in support requests. We do not
anticipate too much complexity, but will field these requests as they come in. For chains that specifically do need the
old `OptimismPortal` and L2OO functionality, they can vendor the legacy contracts into their code base (i.e. effectively
copy/paste the legacy code). Because this code is static and no longer maintained, this should not be too complex.
