# Forge Scripts Deploy in Go

# Purpose

The purpose of this doc is to improve the integration between the `forge script` based chain deployment/genesis flow
and the Go tooling and testing.

# Summary

By running the chain-deployment/genesis solidity scripts in an instrumented Go EVM we can reduce the roundtrip time,
and greatly improve integration with inputs (configs, dependencies) and outputs (state, artifacts).

This unlocks runtime-customization of deployments in Go tests,
to increase coverage and create new multi-L2 chain deployments for Interop tests.

# Problem Statement + Context

From [Design doc 52](https://github.com/ethereum-optimism/design-docs/pull/52):

> The current L2 chain deployment approach originates from a time with Hardhat,
> single L1 target, and a single monolithic set of features.
> 
> Since then the system has migrated to Foundry and extended for more features,
> but remains centered around a single monolithic deploy-config for all its features.
> 
> **With Interop we need to configure a new multi-L2 deployment**:
> the number of ways to compose L2s in tests grows past what a single legacy config template can support.
> 
> Outside of interop, deployment also seems increasingly complex and opaque, while it does not have to be,
> due to the same configuration and composability troubles.

See the design-doc for further in-depth context on the problem.

The integration between (1) Go testing/tooling and (2) Forge deployment/genesis needs to improve
to reduce the complexity of deploying and testing.

Specifically, Go tools need reliable and organized inputs for deployment/genesis,
and interface with the forge scripts artifacts or scripts themselves
in a way that is configurable but not error-prone.

# Proposed Solution

In [deployment-chains design doc 52](https://github.com/ethereum-optimism/design-docs/pull/52) and
[OPContractsManager design doc 60](https://github.com/ethereum-optimism/design-docs/pull/60) the importance
of Modular configs and incremental deploy steps is outlined.

To utilize those modular configs and deploy steps, we need a way to either 
run and cache the individual Forge script functions (as outlined in 52),
or integrate it closer such that no caching complexity is required.

This document proposes the latter: run the Forge scripts within Go,
such that we do not need multiple steps of Forge script sub-process to build a state for testing / genesis tooling.

## Running Forge scripts in Go

Forge scripts are really just solidity smart-contracts, with the addition of Forge cheat-codes.
Running a contract in Go is relatively trivial: e.g. Cannon tests run the Cannon contracts in an instrumented Go EVM.

To support the Forge cheatcodes, we need to emulate the cheatcode behavior, as the Geth EVM does not natively have support for it.
Cheatcodes are essentially responses to `STATICCALL`s to a special system address, acting like a precompile,
with functions that interact with the EVM environment.

We do not have to support all Forge cheatcodes; just the subset used by the OP-Stack scripts would be sufficient.

The most important cheat-codes are:
- `vm.chainId`: change chain ID
- `vm.load`, `vm.store`: storage getter/setter
- `vm.etch`: write contract code
- `vm.deal`: set balance
- `vm.prank`: change sender
- `vm.getNonce`/`vm.setNonce`: account nonce getter/setter
- `vm.broadcast`,`vm.startBroadcast`, `vm.stopBroadcast`: capture calls as transaction candidates. (Can be no-op, if we do not want to go through Go for production deployments)
- `vm.getCode`/`vm.getDeployedCode`: artifact inspection
- `vm.env{Bool/Uint/Int/etc.}`: env var getters
- `vm.keyExists{...}/parse{...}/writeJson/etc.`: encoding utils
- `vm.addr`: priv key to addr
- `vm.label`: name an address
- `vm.dumpState`: export EVM state
- `vm.loadAllocs`: import EVM state

Note that with many of these, Go instrumentation can really improve integration:
- `dumpState`/`loadAllocs`: no need to encode/decode state or read/write to disk -> faster Go tests
- `getCode`/`getDeployedCode`: attach directly to artifacts, and *track which artifacts were used for a deployment*
- `broadcast`: we can extend the deploy-tool functionality with transactions. Out of scope for now, but can be very useful.

## Forge-artifacts as Go FS

A Go FS is a simple filesystem abstraction: it provides read-only access to some source of files.
A local directory can be wrapped into such FS, but also tarballs, or even data embedded in the Go binary, can be represented as Go FS.
By using this FS abstraction, we can make the access to artifacts very simple, and "mount" the relevant FS into it.

For Go tests, this would be the local FS.

For Go genesis tooling, this might be a bundled FS, or one from a versioned release tarball.

Using an FS helps simplify the way we interact with forge-artifacts.

## Semver the scripts

We have an existing `Semver` pattern for production contracts, but don't apply the same to deploy scripts, yet.
If we introduce this, then the version of the scripts (part of the artifacts) can be inspected
by the Go genesis / test tooling, and usage of the script can then be adapted.
Or at the very least, a warning can be thrown when the Go tool does not support the script.

This improves on the current situation, since the Go tool cannot tell anything about the compatibility
of the deployment output of the forge scripts with the genesis-generation it does.

## Fast input/output

In addition to the Go forge cheatcodes, we could also substitute known contracts in the Forge script setup.
In particular, deploy configs are registered at fixed global addresses.

By mocking these contracts, we can couple config-reads directly to the actual config,
rather than having to load a JSON into a long list of EVM MPT storage leafs,
only then to read it construct it back into a memory JSON string many times over right after.

Instrumentation of the config inputs can also provide a clear trace of when and how configuration affects a deployment.

Similar to config inputs, we can capture outputs more efficiently:
upon `vm.dumpState` we do not have to encode the state; we can simply copy it in-process.
This is great for the execution speed of the Go tests: we do not have to use intermediate state JSON files on disk.

## Usage by op-chain-ops

The op-e2e tests and `op-node genesis` tool both rely on `op-chain-ops/genesis` to prepare a chain state,
given some deploy-configuration.

The `op-chain-ops` package is then responsible of applying the configuration, to generate the correct state.

With this solution it means that it:
- Opens the forge-artifacts FS
- Takes any configuration, and sets up the necessary ENV vars for cheat-code usage.
- Instantiates an instrumented EVM, with the initial script entry-point loaded into it.
  - Including cheat-codes hooked up to the forge-artifacts for ENV data
  - Includes cheat-codes hooked up to state loading / writing ability.
  - Includes any mocked config contracts, where we basically map the bytes4 calldata back to a config attribute.
- Calls the entry-point script, with an ABI argument, to run the particular deploy function of interest.
  - E.g. `deploySuperchain`, or smaller deploy functions like `deployProxyAdmin`
- Capture any output state
- Capture any deployed contract addresses (calls to `Artifacts.s.sol` interface),
  or alternatively simplify the script to just use simple `vm.label` functionality.
- Capture any `vm.label` for later debugging.

## Resource Usage

While this solution does not include caching like
[deployment-chains design doc 52](https://github.com/ethereum-optimism/design-docs/pull/52),
it does improve the performance of genesis generation a lot by bringing the forge execution closer, into the Go process.
In this new solution there is less cost in disk-IO, as there is no writing of cache files,
and configuration and state data can all stay in-process and skip JSON encoding/decoding.

In the past we have generated L2 genesis state through manual artifact inspection and Go based state surgery,
which was moved away from due to complexity of the surgery code, but was sufficiently fast for Go scripts.

This keeps the test setup simple (we don't duplicate any special deploy function work into Go)
and fast (avoid lots of IO / encoding / sub-process overhead).

## Close integration, but no contract logic leaks

By loading the forge scripts, the deployment logic all stays native to the smart-contracts,
to avoid manual surgery steps in Go.

When adding a deployment config variable, the only Go change needed is to add it to the Go config definition,
and write any tests to exercise that deployment, as should be the default for every protocol feature.

The deployment implementation details stay encapsulated in the Forge scripting,
which is unified with the Forge testing, and thus unifying the production and test code paths.

## Potential future extension: deployment integration

The `vm.broadcast` cheat-code is an opportunity for future deployment improvements:
rather than running through Forge script when preparing the production deployment transactions,
we could run through the Go genesis tool.

This would allow us to script more advanced post-processing of the transactions:
- cross-validation against the superchain-registry
- simulation of the transaction with custom tracing

And all bundled in Go, so the end-user does not need to ensure a specific Forge version,
does not need to as many manual `forge script` invocations,
and all environment settings (that might otherwise be set with forge script flags)
can be controlled by defining the exact CLI interface.

This is out-of-scope for now, but may help unify the deploy process that production chains use,
and the deploy process that devnets / op-e2e use.

# Alternatives Considered

See proposed solution [in design doc 52](https://github.com/ethereum-optimism/design-docs/pull/52),
where `forge script` is used as is, and performance concerns are mitigated with caching.

An [experimental draft of the caching](https://github.com/ethereum-optimism/optimism/pull/11297) was implemented,
but arguably the caching introduced too much complexity and fragility.

Other solutions / ideas are discussed in design-doc 52 as well, but were not viable.

# Risks & Uncertainties

## Geth Go EVM

The Go EVM instrumentation might be difficult, as the geth EVM is not as widely used in tooling.
However, the tracing functionality is excellent, and Go is quite flexible.
If we need to we can make very minor tweaks to `op-geth`,
to expose any inaccessible EVM internals needed to implement the forge cheatcodes.

## Eng time

This is a mini-project: the scope of implementing the cheat-codes is not that large (a few days at most),
but the integration into tooling and op-e2e may be more involved (can be a week, maybe two).

In the past the `L2Genesis.s.sol` and `allocs` work, that moved us away from the manual and error-prone op-chain-ops surgery,
was completed successfully in the form of an interop side-project to remove tech-debt. This project scope looks quite similar.

This functionality does block Interop op-e2e testing: without it,
we are not able to customize deployments sufficiently and cleanly (avoiding many more `allocs` special cases),
to get multi-L2 deployments into the op-e2e.

## Devrel feedback

Historically devrel has not been included sufficiently in the deployment-flow design.
Known pain-points like inconsistency between the Go and forge genesis generation,
unclear allocs, and monolithic deploy-config are being addressed, but more feedback may still improve the design.
