# Direction for OPCM test integration

## Purpose

In order to integrate the OPCM into the forge test setup (ie. `Deploy.s.sol`), we must first
simplify the deploy scripts while keeping the desired properties and removing what we don't need.

## Goals

As far as we know, these scripts are not currently being used by chain operators for deployment, and
we intend to do that with OPCM in the future anyways. Therefore, what we need to maintain about the
existing flow is:
- Artifact generation:
  - Contracts should have the same names in the resulting deploy artifacts file at
    `/deployments/{chainId}-deploy.json`
- EVM state:
  - The test values configured by each test suite must be preserved (otherwise the diff to update
    each test will explode).

Nice to haves:
- logs emitted during deployment:
  - This is not essential, but comparing before and after logs is a quick and dirty way to ensure
    that the config is roughly the same.
- Deploying contracts to the same address as before.
  - This might be achievable based on the use of
    [`create2`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/scripts/deploy/Deploy.s.sol#L1703)
    in the existing scripts. If so it would enable us to use statedumps to diff the before and after
    deployments.

## Proposed Solution

A plausible PR stack for achieving this might be:

1. Remove the `viaSafe` functionality: `Deploy.s.sol` currently starts our by deploying a new
  `SystemOwnerSafe`, and then uses `callViaSafe()` and `upgradeAndCallViaSafe()` for all calls to
    the `ProxyAdmin` While maintaining the `viaSafe()` helper methods (they are also needed in
  `DeployOwnership` for now), we should remove all usages in `Deploy.s.sol`, perform all actions
    through the deployer key, then finally transfer the `ProxyAdmin` to the `finalSystemOwner`
    (which the current scripts don't even do at the moment).
2. Deploy a separate `ProxyAdmin` contract for the Superchain. A key diff between the OPCM approach
    and `Deploy.s.sol` is that the latter shares the same `ProxyAdmin` for the Superchain and the OP
    Chain that it deploys. That needs to be fixed to prepare for the OPCM.
3. Deploy the Superchain contracts using the OPCM's `DeploySuperchain.s.sol`. With the previous PRs,
    this will hopefully be straight-forward.
4. Deploy the OP Chain Contracts using the `OPContractsManager` contract. This will be much simpler
    if we can delete the `OptimismPortal`, `L2OutputOracle` and `useFaultProofs` flag. Otherwise we
    will accomplish this using two entirely distinct flows depending on the value of
    `useFaultProofs`:
      1. `true` -> deploy using OPCM
      2. `false` -> deploy using the existing `deploy{ContractName}()` functions. It maybe possible
         to reuse some of the contracts deployed by the OPCM which do not vary pre and post fault
         proofs (ie. the `L1StandardBridge` and `L1CrossDomainMessenger`). This would allow us to
         delete the deploy functions for those contracts, and will be investigated during
         development.

## Appendix

The following content was created while investigating the current test setup flow, and
is provided here for reference.

##### Summary of the current Test Setup

The Test contracts (`UltimateTestHarness` below) inherit from `CommonTest`, and implement a
`setUp()` method which calls to `CommonTest.setUp()`. This results in an approximate call flow of:

```md
`UltimateTestHarness.setUp()`
  ├── Calls `CommonTest.enableXFEATURE()` to populate test flags in `CommonTest`
  ├── `CommonTest.setUp()`
  │    ├── `Setup.setUp()`
  │    │    ├── etches the `Deploy` contract into state
  │    │    ├── Deploy.setUp() (implemented in Deployer.s.sol)
  │    │    │   ├── Artifacts.setUp()
  │    │    │   │   ├── creates outfile to `save()` deployed addresses to
  │    │    │   │   └── optionally reads and `save()`s L1 contract addresses from a `CONTRACT_ADDRESSES_PATH`
  │    │    │   ├── etches `DeployConfig` contract into state.
  │    │    │   └── Calls `DeployConfig.read()` which sets values from the config file into the `DeployConfig`s getters
  │    │    └── Goes on to `l2Genesis.SetUp()` (out of scope here).
  │    ├── Calls feature flag setters on `DeployConfig` base (ie. `setUseFaultProofs()`) depending on the values set in the Harness contract
  │    ├── `Setup.L1()`
  │    │    ├── `Deploy.run()`
  │    │    │     └── `Deploy._run()`
  │    │    │         └── `Deploy._run(_needsSuperchain=true)`
  │    │    │               ├── `setupAdmin()` -> new `ProxyAdmin` and `AddressMessenger`
  │    │    │               ├── `setupSuperchain()` -> new `SuperchainConfig` and `ProtocolVersions`
  │    │    │               ├── if `useAltDA` then `setupOpAltDA()`
  │    │    │               └── `setupOpChain()`
  │    │    │                    ├── `deployProxies()`
  │    │    │                    │    └── Deploys OP Chain specific contracts. Does not contains forking logic based on feature flags
  │    │    │                    ├── `deployImplementations()`
  │    │    │                    │    ├── Deploys implementations. Most of the feature flag specific logic is contained here.
  │    │    │                    │    └── `OptimismPortal` or `OptimismPortal2` is deployed based on `useFaultProofs`
  │    │    │                    │        └── `OptimismPortal2` is replaced with `OptimismPortalInterop` based on `useInterop()`
  │    │    │                    ├── `initializeImplementations()`
  │    │    │                    │    ├── Portal init logic varies based on `useFaultProofs`
  │    │    │                    │    └── `SytemConfig` init logic varies based on `useCustomGasToken`
  │    │    │                    ├── `setAlphabetFaultGameImplementation()`
  │    │    │                    ├── `setFastFaultGameImplementation()`
  │    │    │                    ├── `setCannonFaultGameImplementation()`
  │    │    │                    │    └── prestate varies based on `useMultiThreadedCannon`
  │    │    │                    ├── `setPermissionedCannonFaultGameImplementation()`
  │    │    │                    │    └── prestate varies based on `useMultiThreadedCannon`
  │    │    │                    ├── `transferDisputeGameFactoryOwnership()`
  │    │    │                    └── `transferDelayedWETHOwnership()`
  │    │    └── gets deployed addresses and calls `vm.label()`
  │    └── `Setup.L2()`
  │         └──  (out of scope here)
  └── Other test harness specific logic
```

### Summary of OPSM deploy flow

```md
`OPStackManager_Deploy_Test.setUp()`
  ├── `DeployOPChain_TestBase.setUp()`
  │   ├── `new DeploySuperchain()` and etch the `DeploySuperchain` IO contracts
  │   ├── Call `DeploySuperchain.set()` to populate with test input.
  │   ├── `createDeployImplementationsContract()` (virtual function for deploying different features)
  │   ├── etch the `DeployImplementations` IO contracts.
  │   ├── `new DeployOpChain()` and etch the `DeployOpChain` IO contracts. Defer `DeployOpChain.set()` calls for now.
  │   ├── `createDeployImplementationsContract()` (virtual function for deploying different features)
  │   ├── `DeploySuperchain.run()` to deploy SuperchainConfig and ProtocolVersions
  │   ├── Call `DeployImplementations.set()` to populate impls with test input
  │   └── `DeployImplementations.run()` to deploy SuperchainConfig and ProtocolVersions
  └── Populate test input by calling `DeployOpChain.set()` in the test suite.
```

Feature flags the current iteration of OPCM can support:
- `useFaultProofs = true`
- `useInterop` (either `true` or `false`)
