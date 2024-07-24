# Deployment chains

A deterministic incremental approach to chain deployment creates better composability and customization.
I.e. we organize it as a chain of smaller individually configurable deployments.
This enables multi-L2 Interop test setups, and aims to align with the deployment-approach changes of OPStackManager.

# Problem Statement + Context

The current L2 chain deployment approach originates from a time with Hardhat,
single L1 target, and a single monolithic set of features.

Since then the system has migrated to Foundry and extended for more features,
but remains centered around a single monolithic deploy-config for all its features.

**With Interop we need to configure a new multi-L2 deployment**:
the number of ways to compose L2s in tests grows past what a single legacy config template can support.

Outside of interop, deployment also seems increasingly complex and opaque, while it does not have to be,
due to the same configuration and composability troubles.

## DeployConfig complexity

The `DeployConfig` definition, as per the Go code, consists of 102 configurable variables.
Each one of these may affect the chain deployment at any point during deployment
due to lack of grouping and encapsulation.

Within `op-chain-ops`, we have started introducing some grouping of features:
[PR 11189](https://github.com/ethereum-optimism/optimism/pull/11189).

But this is still limited to the Go side, and the complexity still remains on the solidity deploy scripts.

## Stages of deployment

A L2 chain deployment runs through the following steps:
1. Configure `DeployConfig` (many choices).
2. Create L1 chain, if developing.
3. Deploy superchain-wide contracts to L1 (including a `ProxyAdmin` that should not be shared with step 4)
4. Deploy L2-instance-specific contracts to L1.
5. Collect deployments in one file, where there is 1 of everything (breaks interop).
6. Run the L2-genesis script, depending on L1 deployments and the deploy config.
7. Run tools to turn the L2 genesis state into a chain genesis and rollup configuration.

The configuration of step (1) forces how all the following steps go,
making it impractical to repeat step (2) per L2 chain.

The L1 deployments artifact that bundles superchain and L2-instance-specific
deployments also prevents multiple L2s from running.

The main goal is to unbundle these steps, and make them customizable and composable for better test-coverage,
including Interop parameters specifically.

## OPStackManager

Specs: https://specs.optimism.io/experimental/op-stack-manager.html

Older implementatipn draft: https://github.com/ethereum-optimism/optimism/pull/9985

The OP-Stack manager focuses on making the deployment of the L1 contracts of an L2 chain deterministic,
by deploying proxies to deterministic addresses (salt = chainID),
and coupling to superchain-provided implementation contracts.

This implies a different L1-deployment flow than we have today,
and limits it to just the L2-instance specific contracts.
A deployment phase we also want to isolate the configuration and execution of.

Note that the draft sources configuration inputs explicitly, rather than coupling to the (arguably bloated) 
deploy config.

## Deployer requirements

### Op-e2e needs

As an op-e2e test suite, I need to:
- Customize the deploy-config deeply, including L1 parameters and upgrade parameters, to cover all functionality.
- Create multiple L2s with a shared L1 (to go from isolated L2s to interoperating L2s through interop hardfork).
- Create multiple L2s with a shared L1 and shared superchain (to test interop without emulating hardfork first).

### Devnet needs

As a devnet user, I need to:
- Modify the chain template, for experimentation when building new features.
- Support different "modes" without accidental configuration mixup,
  e.g. I may to run different hardforks, or run with different DA types.

### Production needs

As a production superchain host, I need to:
- Perform a new Superchain core deployment (global contracts and implementation-contract deployments).
- Maintain the Superchain core deployment with upgrades.
- Not deploy/upgrade an L2 at the same time.

As a production L2 chain deployer, I may need to:
- Build an L2 on top of a pre-existing superchain core deployment.
- Upgrade the L2 by attaching to new superchain implementation contracts.
- Customize just the L2 part of the deployment, while retaining compatibility with the general deployment flow.

Production needs may change; there are many external users and new features. Flexibility is key.
Making the deployment more modular fits this.

# Alternatives Considered

## Few enshrined deploy-configs

Not all that different from status-quo.
With copy-paste we can add 1 more chain, and suffix it with `-interop`, to produce a 2-chain test environment.
For early testing this was drafted in [PR 10777](https://github.com/ethereum-optimism/optimism/pull/10777).

This does not yet unify the common superchain part of the deployment, or any of the artifacts,
nor supports test-specific changes to parameters.

## Presets and configs

In Eth2 there exist two classes of configuration:
- "preset" (named, enshrined, compile-time)
- "config" (customizable, run-time)

This helped reduce complexity by encapsulating and limiting compile-time configuration,
but also limited severely what could be changed.
For a state-format it may be good to contain options,
but for the amount of runtime configuration the OP-Stack has, it seems impractical.

And each variable that remains in the runtime-config still cannot be customized in `op-e2e` tests,
due to limits in the current `l1-allocs` and `l2-allocs` pre-generated files approach.

## Templating

By separating contract-code and contract-storage, the contract deployments can be reconfigured without recompiling.
However, this pushes complexity to the user of the deployments data:
it re-introduces smart-contract deployment concepts into the Go chain-ops.
E.g. storage-layouts, storage packing, the occasional `immutable` variable, all make customization difficult.
Not to mention cases where we need entirely different contracts, different hardforks, or more L2 deployments.

The insight here is that we should strongly contain smart-contract configuration to the solidity-scripting side,
and minimize post-processing in Go.

## Mega test deployment

Instead of generating many separate files per configuration,
all deployments could be applied to a single `allocs` with all contracts, and many different parameter choices.
It becomes bulky, but then allows the consumer to selectively choose what
part of the deployment matters, and what is dead world state.

This falls apart because the consumer has to track all the deployments,
and it still leaves the issues of enshrined test-configurations.

# Proposed Solution

### Requirements

The solution requirements boil down to:
- Enable full configurability.
- Make configuration composable, to have multiple L2s, and separate "superchain" from "chain".
- Generate the deployments and L2 state with solidity scripts.

This implies:
- We need to invoke chain deployment during test runtime, since it cannot be captured ahead of time.
- We need the deployment and configuration to be split, such that we can compose the steps.

#### Addressing gaps

##### How do we make this practical (fast) in tests?

We can run it once, and cache the result, if it is deterministic, and if time-specifics can be adjusted later.

##### How do we make this composable?

We group the inputs, per deployment step, to not do everything at once.

##### How do we make this incremental?

We chain the deployment steps, to not redo work.
I.e. each deployment step builds on the output of a previous step.
And we can literally "chain", by commiting to the pre-state and new input, to produce the post-state.

##### How do we chain the steps?

We define the output as the hash of the pre-state and the added input-configurtion.
This makes each step traceable and deterministic.

Note that while steps are "chained" together,
we can fork the deployment at any stage: share common steps, then expand into different settings.
So it is not a single deployment sequence, but more like a tree of possible deployments.

##### How do we keep the cache reasonably sized?

Produced increments might take up more space than desired:
for simplicity, dumping the whole state-allocs each step would be nice.

But if there are many configuration tweaks, and many steps, it may produce too many of these files.

The devnet L1 allocs file today is ~650 KB.
The devnet L2 allocs file today is 9.1 MB: the majority of this data are L2-predeploy proxy placeholders.

To reduce the size we could make the output incremental also:
an output can be JSON, and each account entry overwrites the contents of the previous account entry.
For simplicity we avoid storage-merging, and just overwrite accounts as a whole.
We do not have contracts with particularly many storage entries, nor do the medium-sized ones change with every step. 

By performing merges this way, we can also make hardforks clean:
e.g. we can produce a state for fork A, and then fork it and apply the upgrade to fork B, producing a tiny diff.

The smaller the files, the more we can afford, and the higher the limit
for configuration test-coverage is.

##### How do we not miss the cache with every redeployment?

It is important to omit the "genesis timestamp" from the cache-key, and use some fixed value as input instead:
every real-time test will need a different timestamp, so embedding it as input would break all caching.
Luckily, only the legacy L2-output-oracle L1 contract enshrines anything,
and the L2 genesis block is easily modified before chain-specs are generated.

##### How does this work with production deployments?

The caching, chaining, and commiting is not a part of production.
As it is against a live chain, not a local state-dump.

However, the isolated configuration and deployment-sub-routines do enable
a deployer to plug into the superchain contracts, while customizing just what they need,
without having to untangle a monolithic single deployment script.

Ideally this is performed through the OPStackManager.

### Result

This approach enables us to get rid of "mega-feature" env-vars (Plasma, fault-proofs, Interop)
as well as allocs hardfork-suffixes: everything is configuration,
and we can just fork of existing configuration to tweak something.

The main changes to the deploy/genesis-scripting are to:
1. Get rid of L1 and L2 "allocs": every incremental output is a small "allocs", identified by the hash of inputs.
2. Get rid of env-vars: we make all customization the same type of well-defined input.
3. Split the deploy-config: each deployment sub-routine should have its own well-contained config,
  such that we can attribute the output to the change of input.

The Go `op-e2e` side should then be able to produce the needed configuration input-groups, hash them,
and then either hit the cache of outputs, or invoke the solidity-scripts to generate the outputs. 

Backwards compatibility can be achieved by adding a function that loads a legacy deploy-config,
and applies the sequence of all deployment steps, to generate what would have previously been generated.

The OPStackManager spec and draft already strongly point the L2-instance-specific
deployment of L1 contracts to a subset of deploy-configuration.
At this time that subset is under-specified (misses configuration, no fault-proof support, etc.),
but it could utilize the L2-instance specific input-group and deployment routine, to achieve what it needs.

# Risks & Uncertainties

## Backward compatibility

The Solidity-tests still enshrine configuration through the legacy deployment-config: we do not want to break tests.
To avoid breaking tests, providing a backward-compatible global-configuration wrapper
around the new incremental deployment is important.
This way we can refactor the deployment code, without affecting setup of tests.

## Warmup time

When the cache is empty, these different L1/L2 states will have to be generated through solidity script execution.
While compilation of smart-contracts is cached, the execution might still be relatively slow. 

After trying the cold-start performance, we may want to consider preserving the cache across CI runs,
based on e.g. the checksum of the smart-contract source-code, to avoid a cold-start on every CI run.
