# OP-Supervisor: Failure Modes and Recovery Path Analysis

| | |
|--------|--------------|
| Author | Axel Kingsley |
| Created at | 2025-03-26 |
| Needs Approval From | |
| Other Reviewers |   |
| Status | Draft |

## Introduction

This document covers the supervisor as a new consensus critical component, and by its nature involves interop protocol behaviors, as well as new behavior modalities in op-node.

## Interop Protocol Terms and Behavior Explainer

To start setting context, here is a basic overview of how Interop works at the protocol level:

- **There are Two Chains, A and B** who participate in interop with one another.
- Any time a log is emitted on A, it can be referenced on B.
- And likewise, any logs on B can be referenced on A.
- The way a chain can reference logs from other chains is by creating an *Executing Message*
    - Executing Messages are emitted as Logs by the Cross-L2-Inbox contract when called.
    - Each Executing Message contains indexing information and a hash.
    - The indexing information in each Executing Message points at some *other* log, which we call an *Initiating Message*,
    potententially on another chain (ie an Executing Message on Chain A can point at an Initiating Message on Chain B)
    - When the hash matches the log at the index location, the Executing Message is *Valid*. Otherwise it is *Invalid*
- When working with some blocks produced under the interop protocol, you can be sure that all Executing Message are valid. This is because:
    - Per Protocol, blocks which contain *Invalid* Executing Messages are themselves Invalid, and should be Replaced by a Deposit-Only block.
    - Blocks which only contain *Valid* Executing Messages are, as expected, valid too.
- The Network uses this validity assurance so applications can build with Executing Messages as a secure cross-chain communication.

To track this, Nodes now consider twice as many heads as before:

- Local Unsafe: Represents the basic P2P gossiped chain with no interop validity considered.
- Cross Unsafe: Represents the farthest point within the Local Unsafe chain which optimistically appears to be valid with the Cross Unsafe data of other chains.
- Local Safe: Represents the basic L1 derived chain with no interop validity considered.
- Cross Safe: Represents the farthest point within the Local Safe chain which can be validated with Cross Safe data of other chains from this L1 source.
- Finalization: As always, the L2 data is finalized when the L1 data from which it is derived becomes Finalized. However, Finalization only meaningfully applies to the *Cross Safe* head, because the local safe chain may be discovered invalid post-finalization if data that was awaited turns out to be invalid.

While the protocol rules of interop create a very simple and effective method of keeping interoperation secure, it creates the incentive for chain operators to *not* include Invalid
Executing Messages in their blocks. If they do, then the promotion from Local Safe to Cross Safe will fail and their chain will experience a reorg in order to replace the invalid block.

So, the Supervisor was developed to effectively serve this information in a scalable, secure way:

- The Supervisor manages one “Managed Node” for Chain A, and one for Chain B
    - A Managed Node is an Op-Node set to a special behavior mode to serve a Supervisor
    - Managed Nodes receive the next L1 block from the Supervisor
    - Managed Nodes report their derivation results to the Supervisor for Indexing
    - Managed Nodes may be controlled and reset by the Supervisor to maintain sync
- (There is a different "Mode" than Managed Mode which is not yet implemented, and is described in more detail starting at FM5)
- As Nodes A and B receive unsafe blocks over P2P, they report it to the Supervisor
    - The Supervisor uses this to sync all the receipts to a log database
- The Supervisor hands down the L1 block to Managed Nodes A and B
- Each Node derives zero or more blocks from the L1 input
- As new L2 blocks are derived from the L1, the L1:L2 link is recorded to a Derivation Database
- As new data arrives in the Supervisor, safety promotion routines re-evaluate all given data
    - Unsafe Heads advance as their receipts are indexed
    - Cross Unsafe is evaluated any time new data is available
    - Safe heads advance as their derivation is reported
    - Cross Safe is evaluated any time new data is available
- If *positively invalid* messages are discovered in Safe data, the Supervisor directs the given Node to Replace the block with a Deposit-Only block

In this way, the Supervisor is an implementation of the Interop Protocol: it evaluates the validity of interop messages, and initiates the block replacement specified in the protocol.

The Supervisor *also* serves the data it computes over RPC, for other components to check the validity of interop messages *before* they are included in a block, to avoid invalid messages.

The Supervisor is as important as our Consensus Node itself — it is the primary/sole implementation of a serious aspect of the OP Spec, and if it behaves incorrectly it could mislead an entire network.

# FMs:

There are three conceptual ways for the Supervisor to fail:

- It is unresponsive
- It is incorrect
- It is destructive to the Nodes connected to it in some other way

And all Failure Modes are some subtype. Incorrect responses are much worse than no responses, as they may mislead the network. At maximum, a network which is mislead may have fraudulent interop messages played on it, resulting in arbitrary damage to that network. I say “mislead” because all invalid interop messages are invalid by the protocol, and this fact can be checked using L1 data. But because there is limited client diversity, bugs may have greater impact.

## FM1a: Supervisor Is Totally Unavailable

- Description
    - The Supervisor is entirely unavailable, as if it were disconnected.
    - No Message/Block Validity could be determined for any Node connected to this Supervisor.
    - Any Node in Managed Mode would be unable to advance their chain state at all besides local-unsafe.
    - Any Node in Standard Mode would be unable to advance cross-unsafe and cross-safe, but could still advance local-unsafe and local-safe. And because Validity can’t be checked, the Node may prefer not to trust Safe data either.
- Risk Assessment
    - Low Impact, High likelihood.
    - Software fails; ports become blocked. At some point an interruption will take down a Supervisor.
    - We have redundancy solutions being designed [here](https://github.com/ethereum-optimism/design-docs/pull/218/files?short_path=88594e4#diff-88594e47f0a70261441a7452448ef1f240c7c0f15b9132c7789b2ee2d0e07bd2), which will make Supervisor failures less impactful for Chain Operators.
    - Other Node Operators can adopt similar redundancy measures, but may follow their own infrastructure designs . Operators should treat the Supervisor as consensus critical, and manage it similar to how they would manage their L1 source.
    - The impact if a Sequencer's Supervisor goes down and cannot be replaced is an Interop Liveness failure, where the block building
    won't include interop messages. This is higher impact, as it avoidably harms UX by dropping interop messages which were presumably valid.
- Action Items
    - Monitor for a lack of Interop Messages in blocks, as an indicator that the Supervisor driving block building is down.
    Can also be used to monitor for other chains' percieved interop liveness.

## FM1b: Supervisor Is Corrupted

- Description
    - Not only is the supervisor unavailable, its database has been corrupted or destroyed.
    - In this scenario, all of the above applies, and the recovery is delayed by our ability to restore a working Supervisor.
- Risk Assessment
    - Low Impact, Low likelihood.
    - The Supervisor has robust database trimming to clear invalid states and partial writes.
    - This code, like all Supervisor code, could be tested more.
- Mitigations
    - There is a feature of the Supervisor that databases can be replicated over HTTP. If a Supervisor needs to be quickly synced it can bootstrap its databases from an existing Supervisor.
    - Supervisors themselves should be made redundant, so if one goes down, another may serve in its place. Supervisor *replacement* is not something that is well tested, and more likely you’d switch to use that backup Supervisor *and* its Managed Nodes.

## FM2a: Supervisor Incorrectly Determines Message Validity in Response to Sequencer - Unsafe Block Only

- Description
    - If the Supervisor were to claim an Executing Message is valid when it actually isn’t, Sequencers may use this determination to include an invalid message in their local-unsafe chain.
    - Assuming the Supervisor correctly evaluates the cross-promotion (a later step), it will experience a cross-unsafe head stall at this error point, because of the cross-invalid message.
    - The Sequencer does not care about cross-unsafe chain stalling, because unsafe data is subject to change, and as far as it knows, all the messages it has included are valid, so the *expectation* is that other dependent data will unstick it.
    - Eventually, the block is published to the L1 and becomes local-safe data.
    - When the Supervisor attempts to promote the local-safe data to cross-safe, it discovers the invalid message and issues an invalidation and replacement.
    - The Replacement block is applied to the block which contained the invalid message, and the chain has now reorg’d out all blocks from the invalid message to the safe head (effectively resetting the chain back to the stalled cross-unsafe head).
- Risk Assessment
    - Medium Impact, Medium Likelihood.
    - A reorg which affects the Safe Chain implicitly invalidates the Unsafe Chain as well, causing disruption to users.
    - The message validity code in the Supervisor is the most core aspect of its implementation. It has reasonable unit testing for database, accessors, apis, and all validity checking code. It has some E2E tests, and some situationally comprehensive Action tests, as well as local Kurtosis and Devnet exposure. However, this component is not battle tested.
    - *Even When* the Supervisor behaves totally correctly, this case may occur if some data cross-unsafe data is used to build a block which later becomes invalid due to a reorg on the referenced chain. In this situation, the same outcome is felt by the network.
- Mitigations
    - The Sequencer *should* detect cross-unsafe head stall and issue a reorg on the unsafe chain in order to avoid invalid L1 inclusions. Depending on the heuristic used, this could create regular unsafe reorgs with low threshold, or larger, less common ones. This also saves operators from wasted L1 fees when a batch would contain unwanted data.
    - When promoting local-unsafe to cross-unsafe, the Supervisor can additionally detect if the data it is stalled on is already cross-safe or not. If it is, it can proactively notify the Sequencer that the current chain is hopeless to be valid, creating a more eager reorg point.
    - The Batcher can decline to post beyond the current cross-unsafe head. This will avoid the publishing of bad data so the sequencer may reorg it out, saving the replacement based reorg. If it went on long enough, the Batcher would prevent any new data from being posted to L1, effectively creating a safe-head stall until the sequencer resolved the issue. This *could* be a preferred scenario for some chains.
    - We need to develop and use production-realistic networks in large scale testing to exercise failure cases and get confidence that the system behaves and recovers as expected.
- Action Items
    - Develop Monitoring for any messages where are discovered to be invalid in a block.
    Operators should be vigilant when this happens.
    - Additional Monitoring should be developed around ETH transfers per day, or other market behaviors
    (to be scoped)


## FM2b: Supervisor Doesn’t Catch Block Invalidation of Safe Data

- Description
    - FM2a has occurred, but additionally, the Supervisor doesn’t catch the invalid message when promoting from local-safe to cross-safe.
    - An output root that builds on this incorrect cross-safe head is published to L1.
    - At this point, any validators who rely on the Supervisor are following an incorrectly derived chain — blocks between the invalid message and the end of the batch should be replaced by deposit only blocks.
    - Any validators who *do not* rely on the failing Supervisor will see the correct chain, but there are currently no alternative implementations to use.
    - An output root posted from this incorrect state would be open to be fault-proven.
- Risk Assessment
    - Even Higher Impact, Low Likelihood.
- Recovery
    - Within the 12h Safe Head window, if the sequencer is repaired and rewound, it could correctly interpret the batch data to replace a block, and then would have the job of rebuilding the chain from that point. All users would need to upgrade to a version without this derivation bug, and resync the chain from the failed position. Operators who *did not* upgrade and resync would be left on a dead branch that is no longer being updated.
- Action Items
    - Improved testing is required across the board for cross-head promotion, as this is the area most likely to
    break consensus.
    - Implement an ETH rate limit which prevents more than X Eth from being transferred in a single transaction.
    This should limit the blast radius of any undiscovered bugs.
    - Implement a pause mechanism which declines interop transactions from being included in block building.
        - This pause mechanism should be triggerable via admin API
        - And also should be responsive to automated Alert/Monitoring based triggers.
        - Once paused, unpause should be manual to start with.
    - Optionally: Implement a matching Batcher pause to avoid publishing invalid blocks in batches where avoidable.

## FM3: The Supervisor Issues a Reset to the Sequencer

- Description
    - The Supervisor manages Nodes in Managed Mode, meaning they listen to the Supervisor for signals for what derivation activities to take next.
    - One such activity is to reset the node to specific heads.
    - Due to some misbehavior, the Supervisor could issue a reset to the Sequencer.
    - Due to the way the Supervisor navigates and negotiates resets to Nodes, it has the potential to reset to an arbitrary depth.
    - If this happened, the Sequencer would indeed reset, and the ability for the Sequencer to advance the chain would be broken, effectively causing an unsafe head stall until it could recover.
- Risk Assessment
    - Medium Impact (due to mitigtaions), Medium Likelihood.
    - Currently, the code used to issue resets to Managed Nodes is insufficiently tested. This is due to limitations in our ability to construct lifelike-yet-erroneous scenarios for Nodes to sync against. **As far as it has been battle-tested thus**-**far**, Managed Nodes are stable (for example, the “Denver Devnet” has run for 40 days with minimal operations, while supporting real builder traffic).
- Mitigations
    - If a reset would significantly roll back the Sequencer, a chain with a Conductor Set *should* be able to Identify that the Node is unhealthy and elect a new Active Sequencer. In this case, there would be no interruption to the chain, as the tip is continued by the new Active Sequencer.
    - We will be cleaning up the number of Node<>Supervisor messages in current operation, which will allow us to hook closer metrics to this. If A Sequencer ever gets a reset signal, it may be worthy of an alert on its own (even if the reset is due to a legitimate reason, they are rare enough to be tracked)
- Recovery
    - With respect to the Node, an arbitrary amount of re-sync may be required.
    - Recovery to the network depends entirely on the impact of the node outage.

## FM4a: Managed Nodes are at Different Heights

- Description
    - The Supervisor is managing one Managed Node per chain, and using the data reported from their derivation to calculate cross-safety.
    - While the Node for Chain A has derived to some height (L1 block 100), the Node for Chain B is still processing earlier L1 blocks (L1 block 90).
    - During this period, cross safety can’t be fully calculated between beyond L1 block 90 for *any chain which references Chain B* when the referenced data is yet unsynced (direct or indirect).
    - Any chains who do not need Chain B data of the unsynced region can process normally.
- Risk Assessment
    - Zero Impact, Certainty.
    - This is just the natural consequence of syncing from multiple data sources at once, you can’t know what you don’t yet know.
    - The Supervisor already takes this lack of data into account when responding to queries and advancing safety.

## FM4b: A Managed Node Stalls or Lags Significantly

- Description
    - The Supervisor is managing one Managed Node per chain, and using the data reported from their derivation to calculate cross-safety.
    - For some reason, a Managed Node is not able to sync the chain quickly, or at all (perhaps the node is down or is failing)
    - Without the data from this Node, the Supervisor cannot advance safety for any other chain which depends on this stalled chain.
    - The Supervisor also won’t be able to answer any validity questions about the un-synced portion of the chain (which is why safety doesn’t advance)
    - Assuming they take some dependency on un-syncing chain, other chains will eventually stall their cross-unsafe and cross-safe chains
- Risk Assessment
    - Low Impact, Low likelihood
    - The Supervisor knows the boundaries of data it can/can’t report on, and won’t answer *incorrectly* just because it doesn’t have data.
    - If the Managed Nodes are Sequencers who need interop data to build blocks, they will be unable to validate interop messages, and will therefore not include them in block building
- Mitigation
    - Supervisor supports a feature in which *multiple* Nodes of a given chain can be connected as Managed Nodes. If one Node goes down, syncing can continue with the backup.
        - This feature is mostly ready in the happy path, but there are known gaps in managing secondary Managed Nodes during block replacements and resets.
        - This feature *also* needs much more robust testing in lifelike environments. Previous development cycles were spent in the devnet trying to enable this feature, which was slow and risky. To get this feature working well, we need to leverage our new network-wide testing abilities.

## FM4c: Supervisor has Insufficient Performance

- Description
    - Like FM4b, updates are not happening on the Supervisor quickly enough and it is falling behind.
    - In this instance however, it is due to Supervisor performance, not Nodes.
    - This is a liveness threat to the Supervisor only.
    - Nodes who rely on the Supervisor may not be able to get all the queries they need answered, leading to protectively dropped interop transactions.
- Risk Assessment
    - Low Impact, Low likelihood
    - The Supervisor does not do strenuous calculations, mostly just DB lookups.
    - Nodes and their Execution Engines are likely to be the bottleneck because they have to process the Gas of the block.

## FM5: Increased Operational/Infrastructure Load Results in Fewer Nodes

- Description
    - In the current design, validating a given chain in the Supechain requires validation of *all* chains. Conceptually,
    this is unavoidable because inter-chain message validity requires inter-chain validation.
    - The way a user would achieve this today is by running one Node (`op-geth` and `op-node`) for each Chain in the Interopating Set, and would additionally run a Supervisor to manage these Managed Nodes.
    - If operators want redundancy, they are advised to create entire redundant stacks (N nodes and Supervisor)
    - Operators may not appreciate the increased burden and could decline to validate the network at all
- Risk Assessment
    - Sliding Scale Impact and Likelihood
    - Network security is not based on number of validators, but fewer people running the OP Stack means less validation, mindshare, etc.
- Mitigation
    - There is a feature which we have not yet had a chance to implement called "Standard Mode", which is a counter to "Managed Mode"
        - In Standard Mode, a Node *is not* managed by a Supervisor, and instead runs its on L1 derivation
        - Periodically, the Node reaches out to *some* trusted Supervisor to fetch the Cross-Heads and any Replacement signals
        - The Node uses this data to confirm that the chain it has derived is also cross-safe, and to handle invalid blocks
    - With Standard Mode, operators would only *need* to run a single Node, in exchange for the interop aspects of validation being a trusted activity. We expect this Mode to be valuable to offset operator burden in cases where fully trustless validation isn't critical.

## FM6: Standard Mode Nodes Trust a Malicious Supervisor

- Description
    - A Node is in Standard Mode, doing derivation on the L1 independently
    - At some moment, it reaches out to a trusted Supervisor endpoint to query for Cross Safety and Replacement information
    - The Supervisor is incorrect, or even adversarial
    - The result depends on the way in which the data is incorrect:
        - If the data ignores a block Invalidation/Replacement, the Node will be deceived into following a chain where an invalid interop message exists, allowing for invalid interop behaviors (like inappropriate minting) *on that Node* (and presumably all Nodes who also trust this Supervisor)
        - If the data claims a block should be replaced, the Node would similarly perform the replacement, leaving this Node at that state.
        - If the data simply isn't consistently applicable to what the Node independently derived, the Node has nothing to be deceived about, but also can't make forward progress, and would need to halt at this point.
- Risk Assessment
    - Low Impact, Unknown Likelihood
    - If there were a tactical reason to use a Trusted Sequencer Endpoint to fool a Node as part of a larger exploit, this would be an attractive thing to attempt.
    - Most entities who would provide a Trusted Endpoint have intrinsic incentive to keep their endpoint valid and secure. Much like a provider like Alchemy doesn't want the Node you connect to to lie.
    - At most, a dishonest Supervisor can mislead the Nodes connected to it. So long as block producers *are not* using one of these Trusted Endpoints, block production is unaffected.
- Mitigation
    - Standard Mode could allow for *multiple* Supervisor Endpoints to be specified, they could confirm that all endpoints agree, preventing dishonesty from one party from deceiving the Node.

## FM7: Supervisor Indexing Fills Disk
- Description
    - The Supervisor indexes information about every log in interoperating chains.
    - Each log takes at least 24 bytes to store, with Executing Messages taking 3x as much.
    - A large network like OPM processes millions of logs per day.
    - If not provisioned correctly, the Supervisor may run out of room to store the DB.
    - Should this occur, the Supervisor will fail in indeterminite ways and will likely crash, leading to liveness failures.
- Risk Assessment
    - Low Impact, Low Likelihood
    - We can monitor disk space and keep machines well provisioned.
    - Supervisor already holds a very minimal amount of data per log, costing ~134Mb per chain per day.
- Mitigations
    - The Supervisor may not need to hold old data if logs past a certain age can't be referenced.
    - Excessively old data could be purged from the DB and obtained through some alternative implementation,
    if the caller still needed it.

# Action Item Summary

Across all these Failure Modes, the following are explicitly identified improvements and mitigations we should make soon:
- Alternative implementations should exist to catch instances where Supervisor has a bug.
- Higher quality testing (NAT, Test-SDK, DSL) for:
    - Rigorus syncing behaviors, including node resets and multi-node syncing.
    - High volume indexing for performance.
    - Chaos-Monkey style testing for Correctness.
- Smarter behaviors for existing components:
    - Batcher 
        - Only submit up-to cross-unsafe.
        - Optionally: pause on interop failure, to avoid further publishing of invalid messages.
    - Sequencer
        - Look for cross-unsafe stalls to initiate unsafe reorg, avoiding L1 publishing of invalid message.
        - Pause interop (reject all interop Messages) when an invalid message is included in the chain.
        - Admin APIs to initiate pause during incidents.
    - Node
        - Standard Mode should be implemented to allow operators to run a single Node to validate.
            - Standard Mode should check with multiple Supervisors to avoid deception.
            - Standard Mode only needs to check with Supervisor when no Interop Messages are present.
- Operational
    - Conductor to consider Supervisor liveness for Leadership Transfers.
    - Monitoring: we have metrics and logs, and need to build and maintain further dashboards.
    We also need to build monitoring tools which go beyond metrics and can potentially challenge Supervisor
    outputs (similar to Alternative Implementations)
        - Monitoring the total amount of ETH moving
        - Monitoring the volume of Interop Messages being accepted/rejected
        - Monitoring the total number of Interop Messages per block (0 may indicate liveness issue)
        - Monitoring which confirms DB correctness against other sources (like RPC to a node)
    - Alerts: we need to develop meaningful alert criteria to respond immediately to failure early.
        - Any Invalid Message in a block is an alert
        - Supervisor crashes are an alert
    - Runbooks: at a minimum we need to further document how to operate a reorg, or how to respond when the Supervisor
    fails in various ways. Alerts should all have runbooks.

When the Supervisor fails, the worst thing it could do is report the wrong information for a given message.
This incorrect data can mislead a network into building an invalid chain which must be dropped once it is
known to be invalid. At worst, if no one catches the incorrect data, the network can be mislead into an entirely
non-canonical state indefinitely, threatening the validity of the network itself.

Additional Considerations: If we cannot reduce risk in the Supervisor with the above action items, we can further consider:
- 2 step cross chain messages plus pause button
- Only build blocks based on safe cross chain messages, eliminating the opportunity for invalid message inclusion (FM 2A)
- Use sequencer policy to only include ether transfers that are safe