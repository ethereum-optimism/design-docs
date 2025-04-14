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
│   Gossip   │                    │    ┌────────────┐    ┌────────────┐
└────────────┘                    │    │            │    │            │
                                  ├──► │     ???    │──► │  L2 Chain  │
┌────────────┐    ┌────────────┐  │    │            │    │            │
│     DA     │    │            │  │    └────────────┘    └────────────┘
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
│   Gossip   │                    │    ┌────────────┐    ┌────────────┐
└────────────┘                    │    │            │    │            │
                                  ├──► │ Engine API │──► │  L2 Chain  │
┌────────────┐    ┌────────────┐  │    │            │    │            │
│     DA     │    │            │  │    └────────────┘    └────────────┘
│            ├──► │ Derivation ├──┘
│   Watcher  │    │            │
└────────────┘    └────────────┘
```






```
┌────────────┐
│L2 Sequencer│
│            ├───┐
│   Gossip   │   │   ┌────────────┐   ┌────────────┐   ┌────────────┐
└────────────┘   │   │            │   │            │   │            │
                 ├──►│ Derivation │──►│ Engine API │──►│   State    │
┌────────────┐   │   │            │   │            │   │            │
│     DA     │   │   └────────────┘   └┬───────────┘   └┬───────────┘
│            ├───┘              ▲      │                │
│   Watcher  │                  └──────┴────────────────┘
└────────────┘
```


