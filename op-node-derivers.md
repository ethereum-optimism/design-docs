# `op-node-derivers`

#### Metadata

Authors: @protolambda.
Created: May 1, 2024.

# Purpose

<!-- *This section is also sometimes called “Motivations” or “Goals”.*
 *It is fine to remove this section from the final document,
  but understanding the purpose of the doc when writing is very helpful.* -->

The core component of native cross-L2 interop is
[message dependency-validation](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/messaging.md).
This is a new core chain-validation process, adjacent to derivation,
that the rollup node implementation will have to support.

The purpose of this design doc is to support the integration of new rollup-node processes like this,
by improving the abstractions, encapsulating complexity, and decoupling chain-state from derivation-state better.

# Summary

<!-- *Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible.* -->

To support "cross-unsafe" block validation of Interop:
- Slim down and simplify the derivation, by decoupling the `EngineQueue`.
- Encapsulate the driver sub-processes, by splitting the state-loop into smaller composable "derivers".


# Problem Statement + Context

<!-- *Describe the specific problem that the document is seeking to address as well as information
needed to understand the problem and design space.
If more information is needed on the costs of the problem, this is a good place to that information.* -->

The op-node splits the "when" (driver) and the "how" (derivation).
This separation of scheduling and processing is key for fault-proving,
where there is no "when", and where the "how" needs to be synchronous.

The problem is that neither driver or derivation is extensible,
and with every feature that is forced into it, the total becomes more fragile and complicated.

## Tech-debt

The "tech-debt" topic applies to the op-node, not directly to the rollup-node spec.
However, this is an area that is under-specified,
and as de-facto reference implementation the op-node should not be encumbered with tech-debt that makes it unreadable.

Things like the unsafe-block processing and sequencing work were fitted into safe-block processing code-path:
1) This overloads the responsibilities of the derivation-pipeline, in particular the "engine-queue".
2) This overloads the responsibilities of the driver state-loop, monolithically and synchronously executing everything.

This is a remnant of the first Bedrock iteration, which synchronously processed L1 deposit events, and nothing else.

This worked reasonably well up to now, since it was mostly centered around a single new resource: the "unsafe" block.

The OP-Stack is growing a lot: new types of resources,
new customizations, and new validation rules are being introduced.
This growth is something we need to layer on more solid abstractions.

#### Driver tech-debt

The `Driver` "schedules" the work.
But currently really it only does some prioritization and is otherwise one monolithic synchronous loop.
This needs to be addressed if we want to make it extensible with new asynchronous validation processes,
rather than overloading the existing ones.

Places where scheduling can improve:
- Sequencing error handling has hot-loop edge-cases.
- Derivation steps have tight time-out needs due to locking up unrelated resources.
- Slow unsafe-block processing stalls the consolidation process for safe-head progression.
- We also interleave sequencing and derivation steps, whereas these can be parallel.
- We lock up all execution when reading the chain-state due to consistency needs.
- There is no ability to attach a new validation process to the driver,
  making engine-control sharing unnecessarily difficult.

Stylistically there is more to improve too: the `Driver` is a *struct of 29 properties*, all managed in one place.

#### Derivation tech-debt

The `DerivationPipeline` is overloaded with more than just L1 to L2 data transformation:
- The safe-block changes that happen are cached for later finality signals to work.
  (More state that derivation has to deal with.) 
- A plasma "input fetcher" may override fetching/traversal/finality as special case.
  (Special hooks, rather than substituting what a rollup does.)
- The EngineQueue derivation stage hosts a payloads queue that is not related to deriving from L1.
  (No prioritization control as caller.)
- The EngineQueue wraps an `EngineController`, rather than producing outputs.
  (Derivation-caller has no control of what/when things are processed. Context timeouts make little sense.)
- The derivation is started / reset with a "find sync start" algorithm.
  (Taking a long time to run, not replaceable, and quite fragile.)

And the above bloats the code-path that is fault-proven;
the unsafe-block and finalization parts are dead code in the fault-proof context.

### Interop

With interop, we are adding an entirely new block safety type:
["cross-unsafe"](https://github.com/ethereum-optimism/specs/blob/main/specs/interop/verifier.md#cross-unsafe-inputs).

This requires the op-node to:
- asynchronously validate "unsafe" blocks to become "cross-unsafe".
- block "safe" progression until blocks are "cross-unsafe" first (applicable to L1-derived block attributes also).
- maintain a view of external L2 chains, and reorg local L2 chain-state when remote L2 chain-state is invalidated.

### Alt-DA

To support alt-DA and other customizations,
there is a desire to extend the driver and derivation behavior with a bundled interface.

Adding or modifying functionality should not require many sparse changes.
Rather, we want a more plugin-like interface, that allows something to fit in and override core functionality,
while reusing default functionality where desired, and still being structured to be fault-provable.

Extending the stack should not require duplication of the state-transition 
modifications between the proof system and regular node.

Alt-DA also changes the meaning of "inputs";
DA commmitments alone are not sufficient to finalize data as we would with rollup data,
as the underlying data is not finalzied when the L1-registration of the commitment is.
With plasma, a challenge period has to expire first.
In other words, alt-DA changes the finality system.

# Alternatives Considered

<!-- *List out a short summary of each possible solution that was considered.
Comparing the effort of each solution* --> 

### Status-quo

To integrate interop in the op-node as-is, we would have to:
- Modify the L2 forkchoice state to include cross-unsafe.
- Modify the payload-consolidation to have a cross-unsafe pre-requisite.
- Modify the payload-force-processing to not instantly produce a safe block (that would bypass cross-unsafe).
- Modify the various forkchoice-updates of EngineController to be consistent with the cross-unsafe head.
- Modify the engine-queue/controller to allow reorgs through a different path than the current block-attributes path.
- Modify the find-sync-start loop to add a block tag of cross-unsafe.
- Modify the driver to accomodate for a special-purpose process that maintains a view of the cross-L2 safety,
  while explicitly synchronizing the engine-control access in the main state-loop.

This is high-effort (medium in size, but hard to test), and accumulates a lot more tech-debt,
since it makes everything less extensible and adds complexity without simplifying any existing code.

### Many `RepeatCond` and some shared locks

See design doc [here (private)](https://github.com/ethereum-optimism/protocol-quest/issues/202).
The gist is that it centers around conditions/effects,
and picks the Go standard-lib solution for conditions that share resources, even though not a popular choice.

From a specs perspective, this direction is promising, as it is relatively easy to match with a spec
that describes these conditions and effects without the scheduling part.
The effects change the chain-state after all, and should be better described in the rollup-node specs.

This models each driver sub-process as a `RepeatCond` (managed condition/effect loop on `sync.Cond`).
Conditionals can share locks, to avoid conflicting usage of resources.
E.g. unsafe-block-processing and sequencing both modify the tip of the chain,
independent of production of L2 attributes from L1 data.
An illustrative draft of this can be found here:
[monorepo PR 10051](https://github.com/ethereum-optimism/optimism/pull/10051);

Pros:
- The conditions/effect functions are easy to call manually, to perform sequences in non-flaky ways during testing.
- It's easy to match the implementation with the specification rules.
- Easily extensible/compositional. Adding `RepeatCond`, swapping conditions/effects, combining them, is easy.

Cons:
- The draft does not cleanly support prioritization of effects within a single resource.
- It introduces more locks and `sync.Cond`, and doesn't follow the preference of communicating state.

This is medium-effort, clears a decent amount of current tech-debt, but also introduces some new tech-debt.
The pros are not exclusive to this solution, and the cons are.

### Small state-loops

This builds on the `RepeatCond` condition/effect idea, in particular the categorization by shared resource.
There's an `unsafe`, `cross-unsafe`, `safe`, `finalized` and `currentL1` resource. 
And each proceeds 1 at a time and can be signaled a reorg.

When a condition is being checked, it implies a prestate for the effect, if the lock is not released in-between.
By hosting the conditions/effects of the same resource together, it is easy to prioritize between them,
and to keep a lock during execution of a condition/effect:
the other conditions/effects would not be applicable while the primary resource itself is taken.

The challenging thing here is to not copy-paste the issues of the current larger monolithic state-loop
into these grouped smaller state-loops.

Into this direction, we need to consider that:
- When a resource is idle, it should be readable by many.
- The loop itself should not inline key prioritization/scheduling logic, it makes it more difficult to test.

The state-loops approach is not dynamically extensible: events are manually merged,
prioritized, and categories of events are limited.

This is medium-effort, but not all that much more testable than the current approach, and retains some tech-debt.

### Derivers as block providers

Another way to look at it, from @axelKingsley, is to think of it as a blocks-first system with "providers" and a "core".
The "providers" produce blocks, and the "core" prioritizes and processes them, along with safety updates.

There are some issues we identified:
- Not all state changes are in fact blocks.
- Creating blocks involves the engine (a contentious resource).
- The "core" should not replicate the existing monolithic state-loop and synchronous execution issues.

But this does start to introduce a plugin-like abstraction.
A provider may do many things (alt-DA deriver) or just one thing (rollup safe-blocks deriver).
Ideally we find a way such that the derivation, safety-progression and scheduling logic
is modular such that we can just combine a few providers to build the driver.

We believe we do not need runtime plugins as binaries:
the Go plugin system is stagnant, and the build complexity expands.

### Pre-image Derivers

This is the most ambitious approach, but also least efficient:
it generalizes the data types completely away by centering everything around a pre-image oracle.
This idea passed by while brainstorming mid 2023, but was, and still is, too much of a generalization.

TLDR:
- `yield` to coordinator to signal pre-requisites. Similar to parking a thread.
- `yield` to coordinator before removing pre-images. Allows for back-pressure.
- fetch/serve pre-images to the coordinator
- assume safety of pre-images is tracked (by coordinator, or as yet another type of specialized pre-image)

Ultimately this type of driver/derivation backbone is impractical today:
- Encoding/decoding of objects between services which can be in the same process.
- It is too low level, you would need to design protocols on top to perform the actual driver/derivation processing.

This type of thing can also be thought of as a provable event-subscription system.
The stack only has to prove a small subset of objectively provable onchain data however,
when the need arises for this generalization, with its complexity, it can be revisited.

# Proposed Solution

<!-- *A high level overview of the proposed solution. When there are multiple alternatives
there should be an explanation of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API) is likely too low level.* -->

## Desired properties

To sum up what did not work in above alternative solutions, and what we aim to achieve with the proposed solution:

- Ability to process effects in parallel, grouped by resource, without enshrining the resource.
- Ability to merge various triggers that may affect the same group of resource, to save on resources.
- Ability to prioritize effects, if multiple apply
  (a resource may have not picked up on subsequent signals if it was busy).
- Ability to bundle conditions/effects, to not require sparse changes from rollup-node mods.
- Ability to rewire existing effects without rewriting or in-place changes, to make code more reusable.
  E.g. interop inserts an intermediate safety level.
- Ability to compose the stack: composition, not object-oriented but extensible, fits most languages and reduces code.
- Ability to couple effects: changes often cannot be atomically coupled like a DB,
  but intermediate failures should be possible to respond to, to maintain some level of consistency.
- Ability to transfer effects between threads without dead-locking or blocking multiple resources.

## Solution

### Derivers and categorized events

By categorizing events, we can schedule processing in parallel, without enshrining specific resources:
```go
type Resource string

const (
    UnsafeL2 Resource = "unsafe-l2"
    PromoteUnsafeL2 Resource = "promote-unsafe-l2" // example: to not tightly couple "unsafe" to "safe" transition
    SafeL2 Resource = "safe-l2"
    FinalizedL2 Resource = "finalized-l2"
    CurrentL1 Resource = "current-l1" // traversal point on L1 for block-attributes generation
  
    // ... standard resources can be extended by mods / new projects, e.g.:
    CrossUnsafeL2 Resource = "cross-unsafe-l2"
    
    // some functionality may gets its own category, to synchronize the work into one place, for less lock contention
    EngineL2 Resource = "engine-l2"
)

type Event struct {
    Resource Resource
    Data any
}
```

We can introduce a `Deriver` as an event processor:
```go
type Deriver interface {
    OnEvent(ctx context.Context, event *Event)
}
```
A deriver is assumed to read-access to any state it needs. Composed by the constructor of the deriver.

#### Composable derivers

Writes idiomatically happen by implementing a purpose-specific Deriver:
```go
type Engine struct {
  control EngineController
}

func (e *Engine) OnEvent(ctx context.Context, event *Event) {
    switch x := event.Data.(type) {
        case *NewPayload:
            select {
              // events can include ways to communicate the result, if necessary, across threads
              case x.Result <- e.onNewPayload(x.Envelope):
              case <-ctx.Done():
            }
        // *insert other engine events*
    }
}

func (e *Engine) onNewPayload(env *eth.ExecutionPayloadEnvelope) error {
    // ... check prestate first etc., insofar the EngineController does not already maintain it.
    return e.control.InsertUnsafePayload(ctx, x.PayloadEnvelope)
}
```

Other derivers can then re-use this, by composing:
```go
type Rollup struct {
    engine Deriver
    pipeline Deriver
    // ...
}

func (r *Rollup) OnEvent(ctx context.Context, event *Event) {
  switch x := event.Data.(type) {
  case *NewPayload:
      // 1) update some rollup state: ...
      // 2) and propagate to the engine deriver
      r.engine.OnEvent(ctx, event)
  case *L1Change:
      r.pipeline.OnEvent(ctx, event)
  default:
      r.pipeline.OnEvent(ctx, event) // can forward anything to a fallback
  }
}
```

Or, a deriver can be given access to the root-level Deriver, to surface new event data to:
```go
type Pipeline struct {
    root Deriver
}

func (p *Pipeline) OnEvent(ctx context.Context, event *Event) {
    switch x := event.Data.(type) {
    case *L1Change:
        // generate block attributes and give them to the unsafe-block processor to extend the chain
        // (or forward to next deriver if already known attributes)
        p.root.OnEvent(ctx, &Event{Resource: UnsafeL2, Data: AttributesEvent{attributes}})
    }
}
```

#### Deriver design principles

Custom stacks like various Alt-DA implementations can wrap around any existing derivers,
extend with new events, possibly re-using standard OP-Plasma events, for additional de-duplication of alt-DA code.

To encourage composability, and reduce code, events should be documented and small in scope.
I.e. create an event for a deriver to turn attributes in an unsafe block,
and follow that up with events to try to promote the block safety, suggest it for finality, and persisting it to the safe head DB.
rather than tightly coupling the four acts: (1) block processing, (2) updating of the safe-head, (3) remembering the finality data all at once, (4) persisting the safe head change.


#### Threads and parallel derivers

Stacks can define a single `OnEvent` handler, and by using `Resource`, still execute on parallel threads, if desired.
Note that things can be entirely synchronous, if the caller ignores the resource preference, and just runs one thing at a time. This can support fault-proofs, reducing the custom code (no fault-proof specific driver logic, just a synchronous loop of calling the deriver of choice).

To get the parallelism, the `Driver`, which itself can be the absolute root `Deriver`,
can create a new go routine for each new `Resource` type, when first seen.
```go
worker, ok := driver.workers[event.Resource]
if !ok {
    worker = ... // spawn new worker
    driver.workers[event.Resource] = worker // keep it around
}
worker.OnEvent(ctx, event)
```

#### Instrumentation

A `InstrumentedDeriver` can be introduced, as a shim for `OnEvent`, wrapping an underlying `Deriver`,
to provide a default set of instrumentation for all current future event handling:
- Measure the execution-time per event type.
- Track utilization per resource.
- Trace/debug log the results and timeouts.

This can help identify bottlenecks, does not have to be rewritten with each op-stack mod/extension,
and fits it all in just a few reusable/aggregate metrics charts.

### Spec changes

In the specs we enshrine:
- The abstract idea of derivers as modules that can handle events.
- The standard derivers and event-types for rollups.
- For each standard deriver: the standard conditions, and corresponding effects, to perform.

Alt-DA, interop, and other more experimental features can then introduce specs for additional derivers.

We can do this incrementally:
1. Introduce derivers as concept
2. Document existing triggers/conditions/effects, grouped by (sub)deriver. (no breaking changes!)
3. Introduce new derivers for Alt-DA and Interop etc.

Conditions/Effects to specify (grouped by resource here) loosely taken from [specs PR 96](https://github.com/ethereum-optimism/specs/pull/96):

#### Unsafe head

Triggers:
- Change of unsafe-payloads queue
- Changed unsafe L2 head
- Time to sequence (open or seal block)
- Attributes conflicting with chain
- Attributes applicable to head
- (new, maybe) Unsafe-head L1-origin not in or ahead of canonical L1
- (new, maybe) Sync-status change (if not implied by unsafe L2 head change)

Condition/effect pairs:
- Unsafe block addition
- Unsafe block sync-trigger
- Unsafe block processing
- Sequencing
- Payload attributes processing

#### Cross-unsafe head (interop)

Triggers:
- External dependencies changed
- Changed unsafe head
- Changed cross-unsafe head

Condition/effect pairs:
- Interop safety progression
- Interop safety reversal

#### Safe head

Triggers:
- Attributes older than unsafe head
- (new, maybe) Safe-head L1-origin no longer in canonical L1

Condition/effect pairs:
- Safety progression
- Safety reversal

#### Finalized head

Triggers:
- New L1 finality signal

Condition/effect pairs:
- Finality progression

#### L1 Input

Triggers:
- new L1 block
- invalidated L1 block
- derivation pipeline not idle

Condition/effect pairs:
- L1 attributes generation
- Pipeline reset


### Potential for test-vectors

Defining standards for derivers, and making events encodeable as JSON,
could be a solid basis for multi-client test-vectors.
A test-runner would simply iterate through a list of encoded events, applied to a deriver,
on top of some pre-state, and then assert some post-state properties.

### Alt-clients: Rust / Java

Typed events and event-switches are very natural in Rust and Java, and even more common than in Go.
Alternative rollup node implementations should be able to adopt this pattern.

### Migration path

This one of the more involved refactors, that we cannot execute all at once in op-node.
This is a proposal for a 3-phase migration:

#### 1) Derivation/Engine decoupling

The derivation pipeline currently is coupled too tightly to the Engine.
In particular, the `EngineQueue` should be removed,
to allow the attributes-generation and unsafe-block processing to be truly independent processes.

The derivation is somewhat statefully connected to the engine however:
- Block-attributes result in later promotion to a "safe" head
- Not all attributes are equal: those generated from span-batches result in unsafe blocks, until the last in the span is processed.
- The find-sync-start routine is coupled to the `EngineQueue` `Reset()`

We need a first step that addresses this, so the derivation process that generates the block attributes
is shareable with the new driver design.

#### 2) Incremental `OnEvent` adoption

We can incrementally change the Driver state-loop sub-processes to fit the `OnEvent` signature,
without changing the synchronous behavior of the state loop.
This way the node behaves the same, and there is no risk of new concurrency bugs.

#### 3) Swapping the driver

Once all the state-loop sub-processes are really functional `Deriver`s,
we can swap the `Driver` with one that implements the resource-based scheduling.
This can be implemented alongside the current driver, and swapped by feature-flag in the `Driver` constructor,
to introduce the concurrency-affecting part of the refactor more gracefully.

An illustrative example of this approach can be found in the draft of the `RepeatCond` in
[monorepo PR 10051](https://github.com/ethereum-optimism/optimism/pull/10051):

We can make the `Driver` itself the root deriver, which accepts external inputs through public methods,
and turns them into events, executing them with the relevant resource-specific worker go routine.

# Risks & Uncertainties

<!-- *An overview of what could go wrong. Also any open questions that need more work to resolve.* -->

## Introduction of concurrency bugs

When running derivers on different threads, there is state that may be shared between them, and needs to be safe.
Deadlocks are especially important to avoid.
Derivers should not acquire one resource before acquiring the other one when other derivers also do so.
`ctx`-timeouts might unblock this, but the stall of locking up like this is already bad.
Instead, derivers should design such that they communicate contentious work to a shared type of `Resource` worker.
I.e. don't communicate by sharing memory, but share memory by communicating.

## Breaking core functionality

The driver/derivation system is the core of the rollup-node, and it is important not to break it.
This refactor should be possible to execute without breaking integration tests, with incremental steps.
Derivation changes should be kept to a minimum (i.e. no major changes to other stages than the `EngineQueue`),
the driver is the primary thing to improve. The `OnEvent` adoption 

## New requirements

Interop and alt-DA are evolving actively, and more projects may need to change the driver/derivation.
We need to support these changes, but also ensure the current system still works.
Testing the refactor should be possible with alt-DA and interop disabled without many sparse feature-flag checks.
The new Deriver pattern should allow for this to stay encapsulated.
Drafts of the alt-DA and interop derivers may help identify issues with the new Deriver pattern before it is finalized.

# Acknowledgements

Special thanks to the OP Labs Node team for feedback on earlier
designs (@trianglesphere, @sebastianst) and input on alt-DA (@axelKingsley).

