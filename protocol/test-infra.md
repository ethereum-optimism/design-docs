# Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

The OP-Stack is growing in many ways, and testing needs to grow with it.

This design doc aims to express what the challenges are,
and what our bigger vision is to improve testing to support the growth.

This is a shared document between Protocol and Platforms teams.

*Note: this doc is actively changing still, this is just a draft opened by Proto.*

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

In summary, the stack is changing in following ways:
- Growth of chains: more test deployments / monitoring
- Growth of features: more edge-cases to validate
- Growth of history: more syncing and regression tests
- Growth of clients: more spec conformance checks
- Growth of activity: more benchmarks and uptime checks
- Growth as platform: more need for testing to be extensible
- Interoperability: new cross-L2 test requirements

Testing is both an Infra and a Software problem.
Arguably this is the Platforms-Protocol team split, but things can be fuzzy between them.

With both platforms and protocol teams we can align on what improvements we need,
and implement them collaboratively.

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

As outlined above, there are different areas where the stack is expanding,
and where testing needs to expand with it.

What we currently have may be sufficient for a while,
but the pressure of a "test ceiling" does also stall development.
E.g. more `op-e2e` bloat -> more `op-e2e` flakes -> less confidence in `op-e2e` -> fewer features / changes.

*Some pressure* may be good: complexity has to stop somewhere.
But ideally this comes as a design-choice, not a pain/delay in development.

The sections below review what we have today, what challenges we have,
and what ideas we have, for each of the growth domains.

### Growth of chains

With a greater number of chains, we will need more test deployments,
and more monitoring.

Today:
- Superchain-registry [validation checks](https://github.com/ethereum-optimism/superchain-registry/tree/main/validation)
- Test nodes on each network, with alerts
- [pessimism](https://github.com/base-org/pessimism) monitoring (deprecated)
- [monitorism](https://github.com/ethereum-optimism/monitorism)

Challenges:
- Test nodes can be unstable: lack of peers can result in missed blocks.
- Test nodes can elevate costs: not everything has to run on the same tier hardware as sequencers.
  There is a economies-of-scale thing to investigate.
- Monitorism may need to be deployed to more chains.


### Growth of features

"Mo' features, mo' problems." Or really, more edge-cases to validate.
We need the test-infrastructure to efficiently express how things can go wrong,
and what we expect to happen in each case.

Today:
- [`op-e2e` system tests](https://github.com/ethereum-optimism/optimism/tree/develop/op-e2e/system)
- [`op-e2e` action tests](https://github.com/ethereum-optimism/optimism/tree/develop/op-e2e/actions)
- [smart contracts tests](https://github.com/ethereum-optimism/optimism/tree/develop/packages/contracts-bedrock/test)
- [op-chain-ops upgrade checks](https://github.com/ethereum-optimism/optimism/tree/develop/op-chain-ops/cmd)
- [superchain-ops](https://github.com/ethereum-optimism/superchain-ops/) checks
- Interop tests (see [interop section](#interoperability))

Challenges:
- System tests are prone to flakes: running N systems,
  in a resource-constrained environment, with concurrent work / timers / etc., tends to miss assertions due to processing stalls.
- `op-e2e` tests are not portable to alternative client implementations.
  We need ways of generalizing test assertions, and decouple the nodes setup from the test itself.
- `op-e2e` system tests are quite monolithic. Adding/removing nodes is high friction.
  Some tests run services that do not influence the test.
- `op-chain-ops` and `superchain-ops` are not automated against scheduled CI runs.
  Upgrade checks should run more regularly than just one-off in devnet/testnet.
- Limited resource sharing: a lot of the feature-tests spawn a copy of the system,
  rather than being able to run against an already running system.

In general the main idea here is that we need a
way to express tests with system requirements, decoupled from system setup:
we can unify upgrade-checks / tests, and run them against shared chain resources,
to not overwhelm CI resources.


### Growth of history

With more chain history, there is more syncing to do, and more regression tests to maintain.

Today:
- Sync tests (internal nodes scheduled to perform a resync).
- Anecdotal syncs (internal/external sync feedback).
- Scheduled Fault-Proof test-runs against testnet.

Challenges:
- Sync tests lack ownership, difficult to set up, and results do not feed into follow-up work well.
- Anecdotal syncs lack context: when something stalls or errors it is difficult to determine why.
- Feature changes that affect sync do not feed into sync-testruns easily.

Some more visible automation, and a log of results
(perhaps with grafana dashboard links, filtered to the affected node), would be very useful here.
Perhaps a discord bot, that can own these sync test-runs, and post the progress/results, could solve the challenges.
There is some success with posting the fault-proof test-runs to Slack, but starting special runs is still too difficult.

### Growth of clients

With more client implementations, there are more (duplicate) spec conformance checks to run.

Today:
- `op-e2e` action tests [exported for Kona](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/proofs/helpers/kona.go)
- `op-e2e` external-geth shim (removed in [12216](https://github.com/ethereum-optimism/optimism/pull/12216))
- [optimism kurtosis](https://github.com/ethpandaops/optimism-package/) to spin up clients in a network

Challenges:
- The Go tests generally don't export to alternative implementations like Rust
- Kurtosis is an "infra" solution to what is also a "software" problem:
  after spinning up the network, we still need to run tests against it.

This might get more approachable with more separation between node setup and testing,
as proposed in the feature-growth section.
Ideally we then run common tests against different combinations of each of the client implementations.

### Growth of activity

With more vertical scaling comes more concern around performance and stability.
We need more benchmarks and uptime checks to understand where we can improve.

Today:
- [op-ufm (user facing monitoring)](https://github.com/ethereum-optimism/infra/tree/main/op-ufm)
- [`replayor`](https://github.com/danyalprout/replayor)
- Internal infra dashboards / alerts
- One-off chain activity analysis

Challenges
- `op-ufm` should run against more chains, and be more visible to engineers.
  E.g. if transaction inclusion speed is not good, we need to do something about it.
- `replayor` needs automation, ideally on top of a shadow-fork of a real network,
  such that we can analyze performance without spending real ETH or harming a production network in any way.
- We need to monitor node performance better.
  `pprof` / Go-resource dashboards should improve and be more center to our work.

Automating flight-recording in `op-service` would be great.
See https://go.dev/blog/execution-traces-2024 for information about flight-recording.
E.g. on RPC call or on particular pre-programmed conditions,
a buffer of performance data from the last N seconds can be dumped and uploaded to some service.
When slow blocks occur, or engineers are interested in a performance snapshot of a real network, this could be great.
Also see [op-geth flight-recording design-doc (internal)](https://github.com/ethereum-optimism/design-docs-private/blob/main/op-geth-flight-recording.md).

There also exists https://pyroscope.io/ for nice continuous monitoring,
although it is built on top of older Golang profiling hooks, and seems to provide less fine-grained information.

### Growth as platform

The more the stack becomes a "platform" that others fork and build new things on top on,
the more thought we should give to the public interface and extensibility of our testing.

Today: no testing platform.

Challenges:
External OP Stacks forks end up having to fork the testing infra as well, and wire in their customizations.
A lot of tests may work by default, but some features may not.
E.g. 4844 blob tests may need to be disabled for an alt-DA network test suite.

There may be ways we can better categorize tests, make test setup more composable,
and improve parametrization of contract versioning and chain configurations.

### Interoperability

Interoperability is a net-new domain, and requires some deeper work than adjustments to handle growth.
Specifically, this requires cross-chain testing.
This testing adds multi-L2 deployments, multi-L2 test setup, and multi-L2 tests to the testing scope.

Today:
- interop-devnet [docker-compose](https://github.com/ethereum-optimism/optimism/tree/develop/interop-devnet)
- interop op-e2e system variant: [SuperSystem](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/interop/supersystem.go) tests
- interop op-e2e action tests variant: [InteropSetup](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/actions/interop/interop.go) tests

Challenges:
- Setting up multiple L2s that all attach to the same L1,
  and managing unique per-chain keys, deployments, configs, resources can be difficult.
  [`op-deployer`](https://github.com/ethereum-optimism/optimism/tree/develop/op-deployer),
  [`interopgen`](https://github.com/ethereum-optimism/optimism/tree/develop/op-chain-ops/interopgen)
  are a good start, but we things can be improved.
  Ideally by unifying more of the setup between kurtosis and op-e2e tests, so there is less setup code to maintain.
- Existing `op-e2e` single-chain and interop variants are diverging.
  We need to unify the test framework more, so that there is no inconsistency between testing the two kinds.
- While already difficult in an existing test, with interop it becomes even more challenging:
  we have 5+ services running per chain, and then N chains, and we need to understand the interactions between them.

We need to make it more seamless to interact with multiple chains from within the same test.
Part of thi is the setup challenge,
part of it is how we express these multi-L2 things without making things incredibly verbose.

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

## Towards a solution

Based on the challenges outlined above, I believe we need:
- Automation to spin up test chains
- Flexibility with node types
- Separation of setup and test
- More expressive language to define tests in
- Improved insights

### Automation to spin up test chains

The automation story is largely a platforms/infra project.
With `op-deployer` and Kurtosis as the main solution to setup configs and nodes quickly.

### Flexibility with node types

Flexibility requires some standardization:
each execution-engine talks the same RPC,
but rollup-nodes, peripherals, alternative proof systems, etc. still have unique API interfaces.

Defining some interfaces (especially RPC namespaces) as "standard", can go a long way for portable testing.

### Separation of setup and test

One pattern we are trying to work towards in `op-e2e` is to hide
the test-setup and node-access with Go interfaces as much as possible.
The more it is separated by interface, the more flexible the implementation of the setup is,
and the more reusable the test.

One example of this is the Go testing that also runs against local docker-compose devnet,
which was unfortunately removed in [PR 12216](https://github.com/ethereum-optimism/optimism/pull/12216).

### More expressive language to define tests in

A test should be simple, easy to maintain by anyone.

At the same time, a test should be able to navigate into an edge-case,
to cover the more subtle state-transition problems.

A lot of the time, the test is complicated due to the setup,
and not having common idioms to access nodes and state that assertions are made over.

Besides separating from the setup, we need to review how we express the tests themselves.
Part of this is the language itself, part of it is the DSL (domain specific language) we build on top.

#### Golang

Golang can be great:
- A lot of the existing tests and test-utilities can be ported over quickly.
- There is a Kurtosis Go SDK we can integrate with
- Golang is the simplest common denominator for test-writing (assuming no introduction of Python just for testing).

While there are many existing test utilities and abstractions, the Go test stack can still improve.
Notably it can be challenging to fork a test into two independent tests, or parametrize in general.

With these existing test utils, some of it already looks like a DSL.
E.g. the action-tests have a specific way of making one thing happen at a time, in a bigger orchestration of actors.

This DSL-like functionality needs to improve in the system-tests,
where awaiting events and more asynchronous work can still feel verbose and fragile.

#### Rust

There may be an argument for testing in Rust.
But with all existing infra tools written in Go,
this may be more challenging to maintain by the current platforms team.

One option might be to automate Kurtosis,
and then define a Rust test-suite runner with the [Rust kurtosis SDK](https://crates.io/crates/kurtosis-sdk)
to interface with the kurtosis deployment.
For those test-cases where we prefer to write it in Rust instead of Go or Solidity.

There is no DSL yet, this would be quite new ground for a lot of the non-Rust engineering.

#### Solidity

However, both Go and Rust fall short on contract interactions: creating bindings for every interaction can be tiresome.
And solidity rust macros might still be too much context switching for a test writer.

A spike, to explore what testing in a solidity-first environment could look like, could be worth doing.
The [`whatif.sol`](https://gist.github.com/protolambda/1c149b54ec54b57610eca6661f687170) is an early draft of what this could look like.

Solidity test scripts, expressing invariants and such,
can also be a great common language between the OP Stack specs, and the tests.
Similar to Python in the [Ethereum L1 Consensus specs](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md).
Defining invariants in solidity, as part of the spec, which we then pull into tests,
could make protocol development more test-driven.

For DSL, we do have Forge patterns:
switching to a fork, and announcing the next call as a broadcast,
are common patterns in foundry-tests that could work well, and might only need minimal changes.
Custom DSL / cheatcodes can be supported if we run the tests in the Go script environment,
where we can plug in our own cheatcodes.
These cheatcodes can potentially just be a thin wrapper around the Go test framework.

### Improved insights

After everything is set up and tests are running, we still need to improve our insights.
How do things fail? How does a live network perform after triggering a non-fatal edge-case?

Generally this means improving monitoring, instrumentation, etc.

Some ideas to improve:
- Light-weight RPC proxies everywhere. We can capture JSON-RPC exchanges between all the services,
  and flag weird timing, and review RPC logs after problems.
  In a way we can untangle the communication between 20+ services, just like logging,
  by tracing and labeling what happens.
- Make flight-recording a standard practice for every service.
  Being able to look at what a node is/was doing is very helpful.
  For tests that take more resources to re-run, having more data after test-failure is important.
- Making op-node stream its events, so we can assert things based on what is happening inside of the node,
  similar to [assertoor](https://github.com/ethpandaops/assertoor/) in L1.
- For more long-running tests, post progress and aggregate results in a channel on the R&D discord.
  And maybe allow channel participants to interact with the test; e.g. node restarts, API queries, pause the testing, restart the test, etc.
- Differential tests: the better we can diff the results against other results
  (previous runs or of alternative clients), the more confidence we can get in compatibility between different implementations.

## Concrete solution

*This document is a work in progress, the above ideas are still being iterated on.*

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
