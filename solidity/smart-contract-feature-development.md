# Purpose

The purpose of this document is to provide guidance on how develop smart contracts without the need for feature branches.

# Summary

Feature branches result in a lot of effort spent keeping the feature branch up to date. We want to have modular code
that enables us to easily compose features together such that we can get code mainlined early and toggle it off when
running in production.

# Problem Statement + Context

We want to move away from a world with long lived feature branches.
To do so, we need to have some flexibility when it comes to composition of features.
This is easy in off-chain code as the cost is minimal for toggling off certain features but still having them in the source code.
This is not the case for Solidity. This document exists to outline how to think about composition of features for Solidity code.
When the feature is ready to be shipped, the feature specific contract should be merged into the base contract.

# Alternatives Considered

## Copy and Paste Implementation

This was the solution for the fault proofs project, which worked well until the custom gas token feature was introduced. Now the contracts
are in a state such that the custom gas token feature needs to be ported to the `OptimismPortal2` before the `OptimismPortal` and the
`OptimismPortal2` can be merged. This may be a good solution for security critical code as we do not rely on the correctness
of solidity inheritance and all of the logic lives in a single contract, making it easier to audit.

# Proposed Solution

Below includes general guidelines for how to develop solidity feature contracts that are meant to be merged into mainline at
some point in the future.

### Composition through Inheritance

Solidity allows contacts to [inherit](https://docs.soliditylang.org/en/v0.8.17/contracts.html#inheritance) other contracts.
The functionality of the contract can then be extended or overridden.
For in development features that modify core contracts already running in production,
a new contract should be created that inherits the core contract and all new functionality should be added to the new contract.
The deploy script should be able to parse some sort of config and optionally deploy the feature enabled version of the contract.

### Handling Inheritance

Inherit the contract that needs new functionality. It is very easy to add net new functions. If certain functions need to be modified,
then the original contract may need to have the `virtual` keyword added in places. The `virtual` keyword should only be added in
places where it must be added and nowhere in addition.

### Handling Semver

The parent contract should update its `version` getter to be a real function. The child contract should implement a `version` getter like so:

```solidity
function version() public override returns (string) {
  return string.concat(super.version(), "+feature-name");
}
```

The `feature-name` should be some identifier that uniquely identifies the feature. It could also be a hardfork name if the release is tied to a hard fork.

### Handling Storage

Handling storage becomes complex and there is no solution that handles all cases well. We generally want to ensure that the contracts
are forwards compatible, so if `solc` is managing storage, the base contract should have spacers when we know that a feature contract
is scheduled to be merged soon.

Another solution is to use unstructured storage, meaning not letting `solc` manage storage and instead using custom storage slots.
Assuming that digests and unique preimages are used for the storage slot keys, then this can work well for ensuring there are no
storage slot collisions. This works well if there is no need for `solc` to manage the storage layout.

If there are multiple feature contracts based on the same base contract being worked on in parallel, then there is complexity around
layout management and it is up to the teams working on features to coordinate with each other if they risk mangling storage layouts.
Using unstructured storage greatly reduces the coordination overhead here.

More attention should be paid when the contracts are going to be deployed to a production network. It is less important to have the
storage layouts 100% perfect when the contracts are going to be used on a transient network like a devnet. It is up to the developers
to be mindful of their own upgrade paths as projects may have varying requirements.

Given enough time, there will be storage layout issues. In the worst case, they can be fixed at upgrade time using the `StorageSetter`
contract, this pattern is already used in production for `Initializable`.

### Deploy Script

The deploy script should be able to parse the deploy config in a standard way where the same config option enables the feature across the
entire codebase. This means that a single config key/value pair should be able to have predeploys set in the L2 genesis, have the feature
enabled contracts deployed to L1 and also tell op-node/op-geth to use that particular feature.

### Merging Contracts

The feature specific contract should be merged back into the base contract when the scheduling allows. This means that the code is nearing
ready to be deployed to production and there is nothing immediately before that is going to be deployed and released.

# Risks & Uncertainties

- Longer compile times due to more contracts in the `contracts-bedrock` package
- Coordination around the "lock" on the contracts package adds a lot of planning overhead. We need a better strategy for handling scheduling of features.
