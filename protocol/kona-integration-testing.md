# Purpose

The purpose of this document is to outline an integration testing strategy for the kona node (github repo: `https://github.com/op-rs/kona`).  
More specifically, we are looking to test the interactions between the internal components of the node (l2 gossip, da watcher, rpc, networking, derivation) and the external actors (other l2 nodes, da layer, sequencers, batchers, etc.).

The testing strategies will be compared using the following metrics (in no specific order):

- Iteration speed (how quickly can we get feedback when writing new code).
- Testing speed (how quickly can we run the tests).
- Development speed/maintainability.
- Reusability by other teams.
- Customizability (how easy it is to adapt the testing strategy to new environments/scenarios).
- Coverage/granularity (how many components are covered by a single test / how granular can we make the testing strategy).
- Credibility/Scope (how much confidence can we derive from the tests).

## Summary

We are looking into several different integration testing strategies, mainly (in increasing order of granularity, maintenance, iteration speed, decreasing order of maintenability and reusability):

- using the `devnet-sdk` with Kurtosis (`https://github.com/ethpandaops/optimism-package`) or in-process to write realistic e2e integration tests with external actors (op-node, op-reth, op-geth, full-scale l1). This is not very granular and the feedback loop is slow.
- Using a custom rust integration test suite that either mocks external actors or deploy them in separate processes. The most granular approach, but a lot of work to implement and maintain.

## Problem statement + context

Currently, we are using the following strategies to debug/test `kona-node`:

- Using very granular, component specific unit tests. This usually tests a single component in isolation. This works well when adding new internal logic to a given component, but is less relevant when the feature involves external components/actors (e.g p2p gossip, or RPC).
- Manually test the features that are not covered by unit tests by either manually running a `kona-node` instance with `op-reth` or using a custom `kurtosis` network configuration. This is very slow, requires a lot of manual work, is not granular and does not prevent further bugs from being introduced.

This document is intended to outline a new testing strategy for the `kona-node` that will allow to automatically test the interactions with external actors and between the node's internal components. This is to:

- Improve the confidence in the e2e performances of the node.
- Make it easier to debug new features *and write new tests*.
- *Prevent future bugs from being introduced*.
- Improve the maintainability of the node.
- Improve the node's resilience to failures.

## Proposed solutions

We have identified two avenues for writing integration tests for the `kona-node`:

### 1. Using `devnet-sdk` + `kurtosis` to write e2e rust integration tests in kona-node

We are currently manually testing the `kona-node` using kurtosis deployments from the `optimism-package` repo. We could automate this testing process using the `devnet-sdk` go bindings to define and run tests involving the kona node.

#### 1.1. Iteration speed

The `devnet-sdk` is already implemented in go and can be used to define/run tests involving a custom kurtosis network. There is some overhead when writing tests with this approach because this involves using a different language (go) and a different framework (kurtosis). Requires building a new docker image.

#### 1.2. Testing speed

This can be pretty useful to integrate in kona's CI pipeline, but it may not be the best solution for local development because of the long time it takes to run the tests (requires building a new docker image and deploy a full kurtosis network every time ~ several minutes for setup per test).

#### 1.3. Development speed/Maintenability

Quite easy to write new tests with the DSL from the `devnet-sdk`. Very low maintenance as we reuse an existing library.

#### 1.4. Reusability by other teams

The `devnet-sdk` is designed to be as flexible as possible. The tests written for the `kona-node` can easily be repurposed by other teams.

#### 1.5. Customizability

Great flexibility in terms of network configuration.

#### 1.6. Granularity

Not very granular: we can only test the interface with the external actors but not the internal components of the node that are not exposed through RPC.

#### 1.7. Credibility/Scope

Good for emulating real-world scenarios. Good for full-scale integration testing. Harder to debug since the tests are e2e.

#### 1.8. Alternatives to `devnet-sdk` + `kurtosis`

- We could use the `in-process` mode of the `devnet-sdk` to define and run tests in a go process and bypass kurtosis. This can improve the testing speed because we don't need to contenarize anything. We would have to write a small go program that spins up a `kona-node` as a subprocess.
- We could use the `op-e2e` library to define and run tests, although this doesn't seem as well maintained as the `devnet-sdk` and was not recommended by the `op-node` protocol team.

### 2. Using a custom rust integration test suite specifically for the `kona-node`

We could also write a custom rust integration test suite specifically for the `kona-node`.

#### 2.1. Rough sketch of the design of the test suite

- This test suite should be more granular than the `devnet-sdk` and aims to specifically test the `kona-node` and its internal components - interfaces/bindings with other nodes/actors are not prioritized.
- This custom test suite could be used to set the node in a specific state and manually test the response of its components to a given input.
- We could leverage the actor model of the `kona-node` to define hooks that isolate the node's internal components. Given a combination of input data to an actor's channel, capture the data emitted to the actor's output channel.
- Interactions with external actors are a bit more involved.
  - We could mock the external actors (L1 nodes, L2 nodes, ...). This would require a lot of work to implement and maintain.
  - We could run the binaries of the external actors in separate processes and communicate with them through RPC. Less work to implement and maintain.

#### 2.2. Iteration speed

Little to none overhead to write new tests with this approach. The overhead to write new tests will mostly depend on the complexity of the test suite. Does not require building a new docker image or the full-node binary.

#### 2.3. Testing speed

Quite fast to run the tests since everything is written in rust and can be done inside `nextest`. In the case where we're running the node/DA in a separate process, we have some overhead to wait for the binaries to start and communicate with them.

#### 2.4. Development speed/Maintenability

Will require writing/maintaining a new DSL/framework. The complexity of the testing suite greatly depends on the granularity we aim for. Properly mocking external actors (especially DA layer) is not trivial and will likely require a lot of work. Running the binaries of the external actors in separate processes would likely be easier to implement and maintain.

#### 2.5. Reusability by other teams

Not meant to be reused by other teams - writing a generic test suite for the `kona-node` would mostly amount to re-implementing the `devnet-sdk` in rust.

#### 2.6. Customizability

Less flexibility for the network configuration (would require a lot of work to implement). Most of the flexibility will come from the extended control we have over the input data to the actors.

#### 2.7. Granularity

Very granular: we can test private interfaces from the `kona-node` that are not exposed through RPC.

#### 2.8. Credibility/Scope

Good for iterating over the internal components of the node and can help with debugging new features (in a more granular way than the `devnet-sdk`). Good to test the resilience of the node to invalid input data/malicious behaviour.

## Proposed course of action

The proposed course of action is to:

- Start by integrating kona-node the `devnet-sdk` (with `kurtosis` or `in-process`) to write e2e rust integration tests in the `kona-node`. This will likely be fast to write and will allow us to bootstrap the e2e testing suite in CI. We can start by using `kurtosis` and later migrate to `in-process` if we find it to be too slow.
- Once the e2e testing suite is in place, we can isolate the failing tests/components and start writing granular rust tests that reproduce these  failures. This will allow us to bootstrap a minimal testing suite in rust.
- Iterate on the testing suite and add more tests to improve code coverage.
