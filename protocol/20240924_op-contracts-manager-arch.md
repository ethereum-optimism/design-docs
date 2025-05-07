# Purpose

OP Contracts Manager is a contract that will deploy the L1 contracts for standard OP Stack chains in a single transaction.
However, we also have use cases for deploying non-standard chains, such as deploying devnet and testnet contracts while developing interop.
This document will outline an architecture that:

- Enables the OP Contracts Manager to deploy these non-standard chains as an initial milestone.
- Gets us most of the way towards supporting standard chains deployments for the subsequent milestone.

# Summary

The original OP Contracts Manager (OPCM) milestones were:

- Milestone 1: Eliminating key handover complexity.
- Milestone 2: Simplifying Superchain upgrades.
- Milestone 3: Supporting interop upgrades.

This design doc proposes the architecture for a Milestone 0.5 of "OPCM is used for all interop devnet & testnet L1 deployments".
Adding this milestone will allow the interop team to build testing infrastructure that wraps OPCM, unifying our development, testnet, and production testing and deployment processes.

# Problem Statement + Context

*Some text copied from https://github.com/ethereum-optimism/design-docs/pull/52, which is related to this design doc.*

> The current L2 chain deployment approach originates from a time with Hardhat, single L1 target, and a single monolithic set of features.
> Since then the system has migrated to Foundry and extended for more features, but remains centered around a single monolithic deploy-config for all its features.
>
> The interop team needs a way to configure new multi-L2 deployments: The number of ways to compose L2s in tests grows past what a single legacy config template can support.
>
> Outside of interop, deployment also seems increasingly complex and opaque, while it does not have to be, due to the same configuration and composability troubles.

Part of the solution to this is described in [design doc 52](https://github.com/ethereum-optimism/design-docs/pull/52), and the other part involves restructuring our Solidity config processing and deploy scripts.
This design doc focuses on the L1 contract configuration and deployment part of the solution.

# Alternatives Considered

The primary alternative is for the interop team to not use OPCM for devnet and testnet deployments.
This is not ideal for various reasons:

- Duplication of effort: The interop team would need to develop and maintain their own deployment scripts, even though new ones will already be written for OPCM.
- Divergence: With duplication, it's likely the interop team's deployment scripts diverge from the production deployment scripts, leading to inconsistencies and bugs.

# Proposed Solution

There are three aspects to deploying L1 contracts:

1. **Deploy Superchain Contracts**. Superchain contracts are shared between many OP chains, so this occurs only occasionally in production, but is needed for every OP chain deployment in devnet and testnet.
2. **Deploy Shared Implementation Contracts**. This occurs once per [contracts release](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/VERSIONING.md) in production, but is needed for every OP chain deployment in devnet and testnet.
3. **Deploy OP Chain Contracts**. This occurs for every OP chain deployment in production, devnet, and testnet.

For each step we define the step inputs (configuration), sub-steps that occur, and the step outputs (artifacts).
For each we start by listing the sub-steps, then define the inputs and outputs for each sub-step, and use those to define the inputs and outputs for the step as a whole.

Sample input and output TOML files are included for each step.
The exact structure of these files is too low-level to be in-scope for this design doc, so just consider them as examples.
However, feel free to comment on specifics if you think they're important.

TOML is used because it is what the superchain-registry is standardizing on, it can be parsed by forge and go, it supports comments (unlike JSON), and it does not have the [quirks](https://noyaml.com/) of YAML.

Each step is a separate forge script that is structured as follows:

- It may be invoked two ways: by providing input TOML file as the only input, or by providing an Input configuration struct as the only input. This way go code and other forge scripts can call it without any File IO, but users can have a file-based interface.
- Similarly, outputs may be structs or TOML files for the same reasons.
- To make the output artifact self-contained, the outputs may also include the inputs. This allows a single artifact to be used to convey all information needed about a deployment.

This design allows forge scripts can be wrapped with go code to facilitate e2e testing by the interop team.
The go code can leverage the struct-based interface to ensure equivalence between the forge script execution and the go execution.

## Step 1: Deploy Superchain contracts

The sub-steps are:

1. Deploy a `ProxyAdmin`.
2. Deploy a `Proxy` owned by `ProxyAdmin`, set the `SuperchainConfig` contract as its implementation, and initialize it.
3. Deploy a `Proxy` owned by `ProxyAdmin`, set the `ProtocolVersions` contract as its implementation, and initialize it.

The inputs for each sub-step are:

1. A `proxyAdminOwner`.
2. The Guardian address and the initial pause status. (The admin of its `Proxy` is always the `ProxyAdmin` from sub-step 1, so this is not an input).
3. A `protocolVersionsOwner`, the `requiredVersion`, and the `recommendedVersion`. (The admin of its `Proxy` is always the `ProxyAdmin` from sub-step 1, so this is not an input).

The outputs for each sub-step are:

1. The `superchainProxyAdmin` contract address.
2. The `superchainConfigProxy` and `superchainConfigImplementation` contract addresses.
3. The `protocolVersionsProxy` and `protocolVersionsImplementation` contract addresses.

A sample TOML deploy config input for this step might look like:

```toml
[roles]
proxyAdminOwner = "0x1234..."
protocolVersionsOwner = "0x1234..."

[superchain_config]
guardian = "0x1234..."
# The paused variable must be part of the interface for dev/testing, but we will
# have it default to false so it can be hidden from users to simplify the interface.
paused = false

[protocol_versions]
requiredVersion = "0.1.0"
recommendedVersion = "0.1.0"
```

A sample TOML output for this step might look like:

```toml
superchainProxyAdmin = "0x1234..."
superchainConfigProxy = "0x1234..."
superchainConfigImpl = "0x1234..."
protocolVersionsProxy = "0x1234..."
protocolVersionsImpl = "0x1234..."

# Everything above this comment are the outputs, below are the inputs.
[roles]
proxyAdminOwner = "0x1234..."
protocolVersionsOwner = "0x1234..."

[superchain_config]
guardian = "0x1234..."
paused = false

[protocol_versions]
requiredVersion = "0.1.0"
recommendedVersion = "0.1.0"
```

## Step 2: Deploy Shared Implementation Contracts

We assume the latest Fault Proofs release `op-contracts/v1.5.0`.
The exact set of sub-steps and inputs may change with future releases.

The sub-steps are deploying the following contracts:

1. DisputeGameFactory
2. AnchorStateRegistry
3. DelayedWETH
4. PreimageOracle
5. MIPS
6. OptimismPortal
7. SystemConfig
8. L1CrossDomainMessenger
9. L1ERC721Bridge
10. L1StandardBridge
11. OptimismMintableERC20Factory

The inputs for each sub-step are:

1. None
1. None (takes prior step's address as an input)
1. `withdrawalDelaySeconds`
1. `minProposalSizeBytes` `challengePeriodSeconds`
1. None (takes prior step's address as an input)
1. `_proofMaturityDelaySeconds` `disputeGameFinalityDelaySeconds`
1. None
1. None
1. None
1. None
1. None

The outputs for each sub-step are the contract addresses of each deploy.

A sample TOML deploy config input for this step might look like:

```toml
[fault_proofs]
withdrawalDelaySeconds = 1
minProposalSizeBytes = 1
challengePeriodSeconds = 1
proofMaturityDelaySeconds = 1
disputeGameFinalityDelaySeconds = 1
```

A sample TOML output for this step might look like:

```toml
disputeGameFactoryImpl = "0x123..."
anchorStateRegistryImpl = "0x123..."
delayedWETHImpl = "0x123..."
preimageOracleImpl = "0x123..."
mipsImpl = "0x123..."
optimismPortalImpl = "0x123..."
systemConfigImpl = "0x123..."
l1CrossDomainMessengerImpl = "0x123..."
l1ERC721BridgeImpl = "0x123..."
l1StandardBridgeImpl = "0x123..."
optimismMintableERC20FactoryImpl = "0x123..."

# Everything above this comment are the outputs, below are the inputs.
[fault_proofs]
withdrawalDelaySeconds = 1
minProposalSizeBytes = 1
challengePeriodSeconds = 1
proofMaturityDelaySeconds = 1
disputeGameFinalityDelaySeconds = 1
```

## Step 3: Deploy OP Chain Contracts

We assume the latest Fault Proofs release `op-contracts/v1.5.0`.
The exact set of sub-steps and inputs may change with future releases.

The sub-steps are:

1. Deploy `AddressManager`
2. Deploy `ProxyAdmin` with it's owner set to `address(this)` and call `proxyAdmin.setAddressManager(addressManager)`
3. Deploy `Proxy`s for:
    1. `DisputeGameFactory`
    2. `AnchorStateRegistry`
    3. `DelayedWETH`
    4. `PreimageOracle`
    5. `MIPS`
    6. `L1ERC721Bridge`
    7. `OptimismPortal`
    8. `SystemConfig`
    9. `OptimismMintableERC20Factory`
4. Deploy and configure the legacy proxied contracts and their implementations, `L1StandardBridge` and `L1CrossDomainMessenger`. (Details left out for brevity).
5. Deploy the `FaultDisputeGame` and `PermissionedDisputeGame` contracts.
6. Set and initialize all proxy implementations
7. Transfer ownership of the `ProxyAdmin` to the input `proxyAdminOwner`.

The L2 chain ID is an input needed to generate the salt for contract deployment.
Other inputs for each sub-step are:

1. None
2. None
3. None
4. None
5. Various FDG and PDG inputs: Left out for brevity
6. Various contract inputs:
    1. Roles: `proxyAdminOwner`, `systemConfigOwner`, `batcher`, `unsafeBlockSigner`, `proposer`, `challenger`
    2. Fees: `basefeeScalar`, `blobBaseFeeScalar`
    3. Fault Proof inputs: Left out for brevity
7. None

The outputs from each step are contract addresses.

A sample TOML deploy config input for this step might look like:

```toml
l2ChainId = 10

[roles]
proxyAdminOwner = "0x123..."
systemConfigOwner = "0x123..."
batcher = "0x123..."
unsafeBlockSigner = "0x123..."
proposer = "0x123..."
challenger = "0x123..."

[fees]
basefeeScalar = 1
blobBaseFeeScalar = 1

[fault_proofs]
# left out for brevity
```

A sample TOML output for this step might look like:

```toml
systemConfigProxy = "0x123..."
l1ERC721BridgeProxy = "0x123..."
faultDisputeGame = "0x123..."
addressManager = "0x123..."
# etc.

# Everything above this comment are the outputs, below are the inputs.
l2ChainId = 10

[roles]
proxyAdminOwner = "0x123..."
systemConfigOwner = "0x123..."
batcher = "0x123..."
unsafeBlockSigner = "0x123..."
proposer = "0x123..."
challenger = "0x123..."

[fees]
basefeeScalar = 1
blobBaseFeeScalar = 1

[fault_proofs]
# Left out for brevity. Note that we may be able to mock the FP system as a way
# to reduce scope for M0.5. It still needs to be implemented in M1, but this
# may help ship something usable for interop sooner
```

The initial version of these input files will be minimal and only include the required inputs to deploy standard chains.
Future versions will include more inputs to support more complex deployments, such as custom gas tokens or alt-DA (it is currently TBD whether this will be part of Milestone 0.5).
The deploy scripts and OP Contracts Manager code will be written in such a way to default to standard chains and only require additional inputs for non-standard chains.

# Risks & Uncertainties

1. Need to make sure this architecture actually works for the interop team for the goal of deduplicating deploy scripts.
2. Need to verify which fault proof contract inputs should be exposed to users vs. which are "static" or can be inferred from other inputs.
3. It must be sufficiently extensible to handle new features such as supporting the `DataAvailabilityChallenge` contract required for Alt-DA.
4. The exact transition plan from the current legacy `DeployConfig.s.sol` to this new system is not yet defined. We'll likely want a script that can convert legacy deploy configs to the new modular format to ease the transition.
