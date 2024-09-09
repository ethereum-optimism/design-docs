# Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

Write state-transition tests against the Go stack, by reusing the action-test framework,
and export the tests for verification of Fault-Proof programs, L2 CL nodes, and L2 EL nodes. 

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

In the monorepo we have `op-e2e/actions`:
a set of Go integration tests, that run against stripped-down in-process nodes.
These tests run directly through the state-transition: no clocks or concurrent processes run.

The involved L1 and L2 chains are valid, and could theoretically be exported,
to reproduce the state-transition elsewhere.

This can serve the multi-client onchain testing:
the same derivation and EVM processing can be reproduced
in an external client or fault-proof program.

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

Testing of the OP-Stack can be categorized in two categories:
- `onchain`: focus on state-transition correctness. Includes derivation and EVM processing.
- `offchain`: focus on RPC compatibility, syncing stability, scheduling behavior.

Or in other words, `onchain` is primarily the Safety aspect of the classic security charter,
whereas `offchain` is primarily the Liveness part of it.

The `onchain` category is critical to the correctness of the fault-proof program.
To reproduce all the aspects of it, we have need to run derivation:
the process that turns L1 data into L2 inputs.

This is more involved than an EVM process, as it relies on structured input on L1,
including receipts, compression, batch-data channels, and blobs.

Thus, to produce correct test-vectors,
we need a process that is set up to create this type of data.

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

Similar to how the Ethereum L1 Consensus-Layer test-vectors (beaconchain only) are generated,
and more recently the Ethereum L1 Execution Layer test-vectors (EVM only),
we can piggy-back on a reference implementation, and export the artifacts.

The action tests in the Optimism monorepo can serve as the same kind of testing:
a collection of interesting state-transitions, based on the defacto reference client (the Go stack),
easy to export.

The `op-e2e` action tests are called "action" tests because a
few actors synchronously run through a sequence of actions:
the test imperatively applies the changes, there is no async scheduling, no clock.

The action tests generally look like:
- Construct test parameters, configs.
- Construct a few actors.
- Get the actors to a starting state, generally at genesis.
- Apply some actions to actors:
  - Creating a block as L1 miner, including manual tx inclusion.
  - Creating a block as L2 sequencer, including manual tx inclusion.
  - Batch-submitting L2 block(s), queuing up the tx into the L1 tx-pool, to be pulled into L1 when desired.
  - Proposing an output-root to L1.
  - Applying finality to L1.
  - Applying a reorg to L1.
  - Forwarding the L1 finality signal to a L2 node.
  - Sharing the latest L1 head signal with the L2 node.
  - Running the "sync" of a L2 client to the latest signaled head.
- Assert outcomes about the state.

With some modifications, we could export the data of test-actors during a test-run:
- Configs of L1 and L2.
- L1 blocks, receipts, pre/post-state.
- L2 blocks, receipts, pre/post-state.

These modifications are likely special cases in the L1 and L2 engine actors,
to export the produced blocks to e.g. JSON files, and any other data such as state.

Potentially, the op-program DB can be reused in the action-tests,
to capture witness data, such that individual blocks can be executed in a FP setting.

A Fault-Proof program, when given a L1 chain and L2 chain,
and potentially witness data, should be able to reproduce the state-transition,
and verify the results match against the expected L2 post-state.

To do more extensive testing, of e.g. alternative L2 consensus nodes,
we need to also export the reorg signals and safety signals,
so these offchain concepts can be acted on by the test target.

For interop, ideally the export format includes test-actor names and chain-IDs,
so we can support multi-chain tests.

### action-tests DSL

The current "actions" DSL has a few quirks, we may want to improve on:
- Some actor functions produce an action, which then still has to be executed.
  The original idea here was to randomize/fuzz actions; this may still be on the table, but will require more time.
- The test-params / genesis generation is quite limited, due to not being able to customize the allocs.
    - Different genesis parameters can be applied by running the forge scripts as part of the setup.
      See `interopgen` work for an example of producing a L1 state and multiple attached L2s, with custom configuration, within a second or two.
    - Forge scripts could help produce custom pre-states. The test could run these scripts as part of the setup phase, using the same tooling as the above.

These things would be worthwhile to improve as part of the testing effort, benefiting both the Go stack, and quality of exported test vectors.

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

The action-tests already run quite rapidly: there is no scheduling to wait for.

However, exporting the test-data will likely add some overhead:
DB features like witness-generation, and file-IO.

By making this test-data export optional, the regular action-tests can still run quickly in CI.

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

See [testing discussion call notes of 9 Sept 2024](https://www.notion.so/oplabs/Weekly-Sync-09-09-2024-implementation-discussions-for-testing-improvements-103c4ed68cc24137af92aaa115292d06).

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

## Time allocation

The action-tests are a shared responsibility, but primarily maintained by Proto,
who is occupied with Interop, and not directly involved in Holocene and Proofs work.
There may be some onboarding required to be effective.

