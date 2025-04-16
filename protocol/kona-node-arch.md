# Kona Rollup Node Architecture

# Summary

The Kona Rollup Node (aka `kona-node`) is a Rust implementation
of the OP Stack rollup node that functions as a Consensus Layer
client for OP Stack chains.

It builds upon the architectural principles of the go-based `op-node`
while introducing improvements in performance, memory safety,
and concurrency through Rust's language features. The `kona-node` will
be responsible for the same core functions as `op-node`: building,
relaying, and verifying the canonical chain of blocks, working in
conjunction with an execution layer client like `op-geth` and `op-reth`.

This design leverages existing work in the Rust ecosystem, particularly
the Kona monorepo for OP Stack types, components, and services, while
introducing a modular architecture that allows for component-level
customization and optimization. The implementation will prioritize
compatibility with the OP Stack protocol while exploring Rust-specific
optimizations for improved performance and resource utilization.

# Problem Statement + Context

The current OP Stack ecosystem relies on a single implementation of the
rollup node (`op-node`) written in Go. While functional, this creates
several challenges:

1. **Single Implementation Risk**: Relying on a single implementation
introduces systemic risk if critical bugs or vulnerabilities are discovered.
Testing the `op-node` is limited to unit tests, action tests, and integration
tests. The individuals implementing features are usually the same ones writing
tests.

2. **Proof Coverage**: "Stage 1.4" or the `kona-proof` implementation uses its own
derivation pipeline implementation. Unlike the `op-node` derivation pipeline,
Kona's derivation pipeline only has test coverage through the Optimism monorepo's
**proof** "action tests".

3. **Performance Limitations**: go-ethereum (aka "geth") was built as a singular
piece of client software. It wasn't built to be extensible for rollups. On the other
hand, reth was built as an sdk with modularity as a first-class citizen. As such,
`op-geth` and the `op-node` are limited in performance by design.

4. **Language-Specific Constraints**: Go's garbage collection and concurrency
model may introduce performance bottlenecks in specific high-throughput scenarios.
Rust's ownership model and lack of garbage collection can provide more predictable
performance characteristics, especially under load.

5. **Ecosystem Growth**: A growing number of Rust crates implement
OP Stack specifications. Some of these are used in `op-reth` as well as "op-revm".
There's an increasing security risk to rely on these implementations without having
increased coverage through an alternate rust rollup node implementation.

The OP Stack protocol itself has been designed with client diversity in mind.
Clear specifications allow for multiple implementations in different languages.
The Rollup Node specification provides a comprehensive guide for implementing a
compatible client, making a Rust implementation both feasible and valuable.

# Proposed Solution

The `kona-node` will be a modular Rust implementation of the rollup node that
adheres to the OP Stack protocol specifications while leveraging Rust's strengths
in performance, safety, and concurrency. The implementation will:

- **Maintain Protocol Compatibility**
- **Leverage Rust Ecosystem Crates**
- **Optimize for Performance**
- **Enhance Modularity**
- **Improve Testability**

## Architecture

### Building Blocks from First Principles

The Rollup Node is a core component of the OP Stack.
It is responsible for constructing the canonical safe L2 chain.
In order to produce the chain, the Rollup Node listens to two sources
of information.

1. **Data Availability Layer**: As new L1 blocks are produced, the receipts
are parsed for new L2 chain updates. L2 inputs (L2 transaction batches + deposits)
are then derived from the data availability (DA) layer.

2. **L2 Sequencer**: The L2 sequencer produces unsafe L2 blocks and sends them over
p2p gossip to other rollup nodes.

What this looks like currently is two sources feeding into derivation,
somehow producing the L2 Chain.

```
┌────────────┐
│L2 Sequencer│
│            ├────────────────────┐
│   Gossip   │                    │    ┌────────────┐
└────────────┘                    │    │            │
                                  ├──► │     ???    │──►  < L2 Chain >
┌────────────┐    ┌────────────┐  │    │            │
│     DA     │    │            │  │    └────────────┘
│            ├──► │ Derivation ├──┘
│   Watcher  │    │            │
└────────────┘    └────────────┘
```

From these sources, the rollup node imports "unsafe" blocks from the L2 sequencer
and safe blocks from the L2 derivation pipeline. Both unsafe and safe blocks
are imported into the L2 execution layer via the Engine API.

```
┌────────────┐
│L2 Sequencer│
│            ├────────────────────┐
│   Gossip   │                    │    ┌────────────┐
└────────────┘                    │    │            │
                                  ├──► │ Engine API │──►  < L2 Chain >
┌────────────┐    ┌────────────┐  │    │            │
│     DA     │    │            │  │    └────────────┘
│            ├──► │ Derivation ├──┘
│   Watcher  │    │            │
└────────────┘    └────────────┘
```

The [Holocene Hardfork][holocene] introduced steady block derivation.
This change allows a payload to be replaced with a deposits-only block
to allow the L2 chain to progress even if an invalid payload is derived.

Steady block derivation has significant architectural impacts for the
rollup node. With Holocene, the derivation component needs to be updated
if a payload is replaced with a deposits-only block. This requires the
Engine API component to send a message to the derivation component,
introducing an edge in the modular component graph.

[holocene]: https://specs.optimism.io/protocol/holocene/derivation.html

```
┌────────────┐
│L2 Sequencer│
│            ├───┐
│   Gossip   │   │   ┌────────────┐   ┌────────────┐
└────────────┘   │   │            │   │            │
                 ├──►│ Derivation │──►│ Engine API │──►  < L2 Chain >
┌────────────┐   │   │            │   │            │
│     DA     │   │   └────────────┘   └──────┬─────┘
│            ├───┘          ▲                │
│   Watcher  │              └────────────────┘
└────────────┘
```

### A Modular Architecture

The components described above are all critical to the rollup node.
Whether these components are part of the architecture is not up for discussion.
What this document addresses is how the components communicate. Namely, do
components share memory or use an actor=based architecture with message-passing.
The remainder of this section will provide reasoning to choose the latter.

In an actor-based system, each component is constructed as an "Actor". There's
a Derivation Actor, DA Watcher Actor, P2P Actor (L2 sequencer gossip),
and Engine Actor.

Instead of having some top-level object "own" components, actors are spawned
as threads, and communication between actors happens through channels with
messages. In this architecture, a top-level layer "orchestrates" the various
actor to handle wiring up communication between actors. The primary benefit
of using an actor-based system is tasks can actually be parallelized. As opposed
to kona's derivation pipeline which uses an ownership model where each stage
in the derivation pipeline owns the previous stage. Because of this ownership,
derivation pipeline work is single-threaded and blocking. Pipeline stages cannot
make progress in parallel.

If the orchestration layer is called "Rollup Node", a full modular architecture
for the Kona Rollup Node looks like the following.

```
      ┌────────────────────────────────────────────────────────────────────┐
      │                                                                    │
  ┌───┤                        Rollup Node Service                         │
  │   │                                                                    │
  │   └──────────────────────────┬────────────────┬─────────────────┬──────┘
  │                              │                │                 │
  │   ┌────────────┐             │                │                 │
  │   │L2 Sequencer│             │                │                 │
  ├─► │            ├───┐         ▼                ▼                 ▼
  │   │   Gossip   │   │   ┌────────────┐   ┌────────────┐     ┌───────────┐
  │   └────────────┘   │   │            │   │            │     │           │
  │                    ├──►│ Derivation │──►│ Engine API │     │ Rpc Actor │
  │   ┌────────────┐   │   │            │   │            │     │           │
  │   │     DA     │   │   └────────────┘   └──────┬─────┘     └───────────┘
  └─► │            ├───┘          ▲                │
      │   Watcher  │              └────────────────┘
      └────────────┘
```

Notice, the "Rpc Actor" doesn't have communication lines drawn to other actors.
This is because the JSON RPC server has rpc modules that it registers which
handle various rpc requests. In effect, each actor registers its rpc module
with the Rpc Actor at construction. The rpc modules in turn grab or perform
actions on other actors via message passing. Effectively, the Rpc Actor
messages with all other actors.

### Sequencer vs Validator Mode

Up to this point, Kona's rollup node architecture has been considered
from the perspective of a _validator_ node. That is, a rollup node
that only derives the l2 chain, but doesn't _build_ (aka "sequence")
the L2 chain. In order to extend support for sequencing the L2 chain,
the rollup node needs to accept transactions, build l2 blocks, and
broadcast these blocks as gossip over the p2p network.

Part of the architecture to consider is being able to plug in
"block building" through a simple API. The `op-node` separates
the sequencer logic from the rest of the node driver, allowing
it to toggle sequencing on and off via a simple CLI flag. Kona
can take this one step further. Using a minimal API, the
`kona-node` should allow sequencing to be toggled on and off,
but also let users easily slot in their own block building and
sequencing logic using the given API.

Consider sequencing for the `kona-node`. A new "Sequencer"
actor would be introduced, extending the architecture as follows.

```
      ┌────────────────────────────────────────────────────────────────────────────────────┐
      │                                                                                    │
  ┌───┤                                 Rollup Node Service                                │
  │   │                                                                                    │
  │   └──────────────────────────┬────────────────┬────────────────┬────────────────┬──────┘
  │                              │                │                │                │
  │   ┌────────────┐             │                │                │                │
  │   │     DA     │             │                │                │                │
  ├─► │            ├───┐         ▼                ▼                ▼                ▼
  │   │   Watcher  │   │   ┌────────────┐   ┌────────────┐   ┌───────────┐    ┌───────────┐
  │   └────────────┘   │   │            │   │            │   │           │    │           │
  │                    ├──►│ Derivation │──►│ Engine API │   │ Sequencer │    │ Rpc Actor │
  │   ┌────────────┐   │   │            │   │            │   │           │    │           │
  │   │L2 Sequencer│   │   └────────────┘   └──────┬─────┘   └─┬───┬─────┘    └───────────┘
  └─► │            ├───┘          ▲                │   ▲       │   │
      │   Gossip   │              └────────────────┘   └───────┘   │
      └────────────┘                                               │
            ▲                                                      │
            └──────────────────────────────────────────────────────┤
                                                                   │
                                                                   ▼
                                                             ┌───────────┐
                                                             │           │
                                                             │ Conductor │
                                                             │           │
                                                             └───────────┘
```

Similarly to the `op-node`, the `kona-node` should make use of the
Conductor abstraction (the `op-conductor` is the implementation).
The conductor API allows the sequencer actor to commit unsafe
payloads to the L2 chain. It also acts as an auxiliary service to
the Rollup Node, performing leader election and chain reconciliation.
Regardless, consensus and `op-conductor` guarantees are periphery
to this document. The key idea here is for `kona-node` to use the
same `op-conductor` abstraction.

# Alternatives Considered

Instead of allowing actors to "pass messages" between each other, it was
considered to route all messages through the orchestration service.

Concretely, this would look like a large-variant enum that holds all
message types. Actors would then only send and receive messages from the
"Rollup Node Service" orchestrator. This more closely resembles the `op-node`
architecture where "derivers" are registered via a "registry". Various
event types are emitted and broadcasted to rollup node components.

While this works effectively for the `op-node` it introduces significant
overhead and risk for Kona's Rollup Node. Since the `kona-node` is
parallized, mishandling or even spontaneous flakes where messages are dropped,
can result in an unrecoverable deadlock. By establishing messaging channels
directly between actors, there's less "surface area" for message passing
to be improperly configured.

# Risks & Uncertainties

Actor-based systems aren't a free lunch. Where parallelization is introduced,
so is the ability for actors to become stuck in a deadlock. Given the cycle
between the Engine Actor and Derivation Actor, this is a real risk. You can
imagine the Derivation Actor isn't reset when a holocene deposits-only block
is created. Meanwhile, the Engine Actor is waiting for safe blocks from the
derivation pipeline.

In a similar way, the `op-node`'s event system architecture also has a risk
of deadlock. With strongly typed messages and explicit well-tested message
handling and emission, this risk is mitigated. One way to extend this testing
to the Kona Rollup Node is to support node action tests like proof action
tests support running the `kona-proof`.
