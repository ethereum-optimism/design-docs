# kona-node: Actor Simplification & Maintainability

|                    |                           |
| ------------------ |---------------------------|
| Author             | _@op-will_                |
| Created at         | _2025-12-10_              |
| Initial Reviewers  | _@theochap, @sebastianst_ |
| Need Approval From |                           |
| Status             | _Draft_                   |

## Summary
Note: This does not propose actually changing any functionality, so it may be unlike other design docs. It is more of a brain-dump of specific improvements, motivations, and desirable outcomes they achieve.

As far as we know, [kona-node](https://github.com/op-rs/kona) functions well, but its design is unnecessarily complex and difficult to understand. In some cases, kona's design makes unit testing very difficult or impossible. This document identifies a few low-effort ways to remove and/or consolidate existing code to increase maintainability and testability while decreasing complexity.

Here is a summary of the proposed improvements:

| # | Update                                    | Effort | Impact | Reasoning                                |
|---|-------------------------------------------|--------|--------|------------------------------------------|
| 1 | Consolidate actor logic                   | Medium | High   | Complexity, Maintainability, Testability |
| 2 | Inject and generify actor dependencies    | Low    | High   | Maintainability, Testability             |
| 3 | Define actor-specific tasks and queues    | Low    | Medium | Complexity, Metrics                      |
| 4 | Make actor <> actor dependencies explicit | Low    | Medium | Complexity                               |


## Problem Statement + Context

`kona-node` is comprised of 6 disparate tasks that are run in parallel. Since Kona refers to them as Actors, so will this document, although this is slightly misleading (see: [5](#5-actor-is-a-misnomer) below for more information). The [original architecture doc](./kona-node-arch.md#sequencer-vs-validator-mode) is very useful context and a good place to start. While I believe the general architecture is a good one that addresses the problems that it aims to solve, a few implementation details are problematic and cause outsized maintenance and complexity burdens. This document discusses them one by one.

### 1. and 2. NodeActor + Actor Struct + Actor Logic Struct Pattern
> [!NOTE]  
> Some PRs to address this problem are already in progress and/or merged. This section describes the problem being solved since that was not specified elsewhere, and it may be useful context for the remaining tasks.

`kona-node` mandates that all disparate tasks that it runs are Actors in that it prescribes the [NodeActor interface](https://github.com/op-rs/kona/blob/main/crates/node/service/src/actors/traits.rs#L19) and parallelizes its tasks ([1](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/service/node.rs#L273) | [2](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/service/util.rs#L11)) assuming that each is a `NodeActor` implementation. Until very recently, `NodeActor` also [required InboundData, OutboundData, and a Builder](https://github.com/op-rs/kona/blob/ca41a61b9ab150b1c5959cba460bf4843a22c222/crates/node/service/src/actors/traits.rs#L19) to be specified for all implementations. I imagine the thought process was:
1. Actors create the different channels that they listen on (called `InboundData`) during construction ([example](https://github.com/op-rs/kona/blob/9b713c8986359bdfc24299ccf128061d71b6a625/crates/node/service/src/actors/sequencer/actor.rs#L206)) 
1. Actors need to communicate with other Actors by writing to their channels
1. Since channels are created on instantiation, we cannot wire dependencies then, but we can pass them into the `start(...)` parallelism entrypoint as `OutboundData` ([example](https://github.com/op-rs/kona/blob/9b713c8986359bdfc24299ccf128061d71b6a625/crates/node/service/src/actors/sequencer/actor.rs#L489))
1. `start(...)` now has all the info that the task needs, so it can build the main logic struct ([example](https://github.com/op-rs/kona/blob/9b713c8986359bdfc24299ccf128061d71b6a625/crates/node/service/src/actors/sequencer/actor.rs#L490))

This became codified such that in order to run a disparate task within `kona-node`, that task had to match this pattern or the developer needed to fudge it a bit to make it work.

This causes 2 large issues:
1. This design is extremely complex, and its complexity does not add any value that could not be achieved without the complexity
2. This pattern makes dependency injection, and thus unit testing, very difficult or impossible depending on the case

There is no reason why all dependencies cannot be injected into the Actor at the time of creation. The reason that they are not is the old NodeActor structure and the pattern it encouraged. This can easily be rethought to consolidate Actor, Actor logic struct, Actor builder, etc. into a simple Actor struct.

Proposed solutions: [Consolidate Actor Logic](#1-consolidate-actor-logic) and [Generify and Inject Actor Dependencies](#2-generify-and-inject-actor-dependencies).

### 3. and 4. Actor dependencies are unclear
Using an API as a metaphor, an Actor usually creates one channel per public method rather than a single inbound channel with multiple supported payloads ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/engine/actor.rs#L78-L98)). 

This results in a few distinct problems:
1. When a developer sees an Actor sending a message to another actor, it is unclear which Actor is the receiver ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/network/actor.rs#L225))
1. The receiving Actor must have boilerplate to loop over each channel ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/engine/actor.rs#L621-L706))
1. It is difficult to identify the slow consumer problem; there is no easy way to measure how many tasks are queued for a given actor

[Proposed solution](#3-and-4-define-actor-specific-tasks-queues-and-facades).

## Proposed Solution

> [!NOTE]  
> While each problem case is different, all cases should aim to:
> - reuse existing logic and tests where possible
> - increase the testability of the referenced code if at all possible

### 1. Consolidate Actor logic
Actors and their logic should be consolidated into a single struct as the default implementation. The structure should be extensible to support more complex implementations, but those should not be necessary.

Other Actors are yet to be updated, but here is `SequencerActor`'s [before](https://github.com/op-rs/kona/blob/1520b6b13a3d8d944c0a0918c380ba10a421f31b/crates/node/service/src/actors/sequencer/actor.rs) and [after](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/sequencer/actor.rs) example. Note:
* `SequencerActorState`, `SequencerActorBuilder` (note: doesn't build `SequencerActor`), and `SequencerActor` consolidated/removed 
* `SequencerActor` is generic over its dependencies
* Unit tests are being created that mock dependencies and test `SequencerActor` logic ([example](https://github.com/op-rs/kona/pull/3172))
* Existing logic is mostly unchanged, just consolidated

Each case will be different, but the following changes are suggested:
* Do not create channels in Actor constructors; inject channels and all other necessary parameters into Actor constructors ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/service/node.rs#L204-L229))
* Since Actors already have all necessary state, remove parameters from `start(...)` (note `sequencer_actor` and `l1_watcher` [here](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/service/node.rs#L273-L313))
* Since Actors are constructed with all necessary state, `start(...)` does not need to create a separate logic struct to which the Actor delegates, and any such struct's logic and fields can be consolidated into the Actor struct. 
* If `start(...)` builds some other struct that can't / shouldn't be merged with the Actor (e.g. [RpcActor](https://github.com/op-rs/kona/blob/820ff001dfe6cd23e7d66ca6b2bbb84cd73ee0d6/crates/node/service/src/actors/rpc.rs#L132)), inject that fully configured struct into the Actor as a dependency
  * This is an indication that the Actor does something related but different and should probably not be in charge of the other struct's configuration. Using RpcActor as an example, the act of running an RPC server and configuring its endpoints are slightly different concerns and can and should be decoupled so the server can run whichever endpoints are passed to it.

After all Actors are simplified, [StartData](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/traits.rs#L24-L27) may be removed from `NodeActor`. 

#### Wins:
* Fewer lines of code
* Logic for single concept contained in single struct
* Separation of concerns
* Fewer structs = easier dependency injection = easier unit testing


### 2. Generify and Inject Actor Dependencies
Actor state includes various structs that contain logic, make RPC calls, etc. If we want to test Actor logic, we either need to _also_ test the logic of each of these structs (i.e. integration test) or need some way to mock it to just test Actor logic (i.e. unit test). To enable the latter: 
1. Where traits (read: interfaces) do not exist for dependency functions, create traits ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/sequencer/conductor.rs#L8-L23))
1. Make the Actor generic over these traits ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/sequencer/actor.rs#L57-L63))
1. Use [automock](https://docs.rs/mockall/latest/mockall/attr.automock.html) on traits to auto-generate easy-to-use mocks ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/sequencer/conductor.rs#L12))
1. Mock dependencies to unit test logic ([example](https://github.com/op-rs/kona/blob/01de8f515ce376d978894c51bd20806dacca29a5/crates/node/service/src/actors/sequencer/admin_api_impl_test.rs#L268-L321))

#### Wins:
* Actor logic that was not previously testable is now testable
* Other users of the same structs can reuse existing traits and mocks for testing

### 3. and 4. Define Actor-specific tasks, queues, and facades
The proposed solution is to have each Actor
1. define an `enum` of the tasks that it supports processing
1. queue tasks upon receipt so it has an obvious and measurable work queue
1. expose a generic clean API-like interface to others to call

Example pseudocode:
```
struct FetchedData { ... }
struct FetchDataRequest {
    identifier: u64,
    response_channel: oneshot::Sender<Result<FetchedData, SomeError>>
}

enum ActorXYZTasks {
    /// The sender will not be notified that this happened successfully
    StartProcessingID(u64),
    FetchData(FetchDataRequest),
    ...
}

struct ActorXYZ {
    cancellation_token: CancellationToken,
    task_queue: mpsc::Receiver<ActorXYZTasks>,
    ...
}

impl ActorXYZ {
    async fn start(mut self, _: Self::StartData) -> Result<(), Self::Error> {
        // Process tasks until cancelled.
        loop {
            select! {
                biased;
                _ = self.cancellation_token.cancelled() => {
                    info!("Received shutdown signal. Exiting sequencer task.");
                    return Ok(());
                },
                Some(task) = self.task_queue.recv() => {
                    self.process_task(task).await;
                }
            }
        }
    }
}

-------------------
// Optional but encouraged client to abstract underlying queue-based communication and provide a clean interface defined 
// below. This might be useful for unit tests to be able to mock a dependency rather than perform channel-based 
// communication and error handling.
// Note: this can be easily generated via AI.

trait ActorXYZClient { ... }

struct QueuedActorXYZClient {
    actor_xyz_task_queue: mpsc::Sender<ActorXYZTasks>,
}

impl ActorXYZClient for QueuedActorXYZClient {
   /// Note: This does not wait for ActorXYZ to start processing, it just confirms it received the message and returns. 
   async fn start_processing_id(id: u64) -> Result<(), SomeError> {
        self.actor_xyz_task_queue.send(ActorXYZTasks::StartProcessingID(id)).await.map_err(|_| {
            SomeError::RequestError("ActorXYZ request channel closed".to_string())
        })?;
   }
   
   async fn fetch_data(identifier: u64) -> Result<FetchedData, SomeError> {
        let (response_channel, rx) = oneshot::channel();

        self.request_tx.send(ActorXYZTasks::FetchData(FetchDataRequest{identifier, response_channel})).await.map_err(|_| {
            SomeError::RequestError("ActorXYZ request channel closed".to_string())
        })?;
        
        // NB: returns the received `Result<FetchedData, SomeError>`
        rx.await.map_err(|_| {
            SomeError::ResponseError("response channel closed".to_string())
        })?
   }
}

```

#### Clarifications:
**Cancellation** - Each Actor has a cancellation token that it monitors to signal that the process is shutting down and it should stop processing. This _can_ be treated as an inbound task but probably shouldn't since it is less of an inbound API task and more of a process-level signal. Cancellation is a common concern that all Actors need to handle, so it is a prime candidates for handling at a higher level (i.e. as a default implementation in the `NodeActor` trait). More thought is necessary, but a promising idea that @theochap had is to change the interface such that each Actor implements a function that accomplishes a single iteration of their loop rather than the loop itself. This would allow cancellation to be managed at a higher level, deduplicating that code and removing that concern from each Actor implementation. 

**Sequencing** - `SequencerActor` uses a timer that ticks every `block_time` to trigger it to do some work. This, however, is distinct from inbound messages, as this is SequencerActor's own logic that it runs rather than being triggered by some other party's request. In the case of sequencer actor, its inbound messages (handling admin API requests), do not at all conflict with block building and can likely be handled in parallel.

**Task Prioritization** - `EngineActor` has a priority queue of inbound tasks. This is because it re-queues tasks on failure in certain circumstances for later execution. This is an anti-pattern because in the event of failure, the caller should be notified to allow them to decide how to handle the failure. See [this issue](https://github.com/op-rs/kona/issues/3021) for more info. My gut feeling is that inbound priority queues should probably be avoided, but this design does not preclude them from being used if there is a legitimate need for them.

#### Wins:
* All tasks that an Actor may process are concisely defined in one place (`ActorXYZTasks`)
* An Actor may easily find all callers by searching for uses of `ActorXYZClient`
* A caller knows which Actor it is sending to because it either has a `mpsc::Sender<ActorXYZTasks>` or a `ActorXYZClient` which lists the Actor and/or tasks
* Unit testing is very easy to mock if using the `ActorXYZClient` trait and easy to build your own facade around if using `mpsc::Sender<ActorXYZTasks>`

### Resource Usage
The proposed changes are not required. Each has a relatively low level of effort and states the outcomes it achieves. If these are desirable, individual tasks should be created and prioritized.

### Single Point of Failure and Multi Client Considerations

No functionality is to change, so this does not apply.

## Failure Mode Analysis

No functionality is to change, so this does not apply.

## Impact on Developer Experience
Code will be smaller, much simpler, and easier to test and maintain.

## Alternatives Considered
Leave the existing design in place. It is not strictly required to make these changes. The effort and benefits are accurately described above.

## Risks & Uncertainties
There is a chance that one or more of the proposed patterns do not apply to some specific aspect of one of these disparate tasks. If that is the case, there is a risk of either:
1. _Making_ them apply
1. Rethinking patterns for everything to support an edge case

These risks are the cause of some of kona's complexity that this proposal aims to address. We should treat the proposed patterns as sensible defaults, using a better approach if the default does not fit. 
