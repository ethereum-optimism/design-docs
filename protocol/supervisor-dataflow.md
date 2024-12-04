# Purpose

This document enumerates a collection of architectural updates to the OP-Supervisor and OP-Node which the OP Labs Interop Engineering Team has discussed prior. 

# Summary

In order to enable more consistent reasoning and control, the OP-Supervisor component should be given the following features:
- Supervisor as L1 Source
- Supervisor as Orchestrator of Nodes
- Bidirectional RPC for "Owned Nodes"
- Reader and Sync API

# Problem Statement + Context
The OP-Supervisor is a new component of the OP-Stack which provides Cross-Chain validation by exhaustively indexing log data for the chains in a Dependency Set. The current design comes with complexity which needs to be managed across the Supervisor and Nodes. The short list of these issues are:

- Nodes get their own L1 view, leading to complexity management in the Supervisor when the L1 derived sources don't match.
- Nodes are the orchestrating authority for syncing data to the Supervisor, leading to the Supervisor having no self-determination, and leading Nodes to need to be considerate of the Supervisor at all times (ie, Nodes must initialize the Supervisor, it cannot initialize itself).
- Because Nodes are the orchestrating authority, the Supervisor is not prepared to accept connection from multiple Nodes for the same Chain. Nodes could coordinate to ensure only one writer, but the Supervisor can't self-determine when multiple writers exist.

These issues come down to a lack of clarity in the ownership model between components. Nodes are responsible for the vast majority of data and derivation, which leads to situations where the Supervisor can't enforce a consistent view over the Nodes at all. If we were to give Supervisor control over data and some activity orchestration, we can have improved consistency and reduced complexity in both the Supervisor and OP Node.

### Extra Context
Leading up to this document were two documents and a meeting:
- [Document 1 by Axel](https://oplabs.notion.site/Axel-Argues-About-Interop-Architecture-Abstractions-13ef153ee16280e4bf2dd2817c121216)
- [Document 2 by Proto](https://oplabs.notion.site/Superchain-node-architecture-thoughts-144f153ee16280a5855af9aeca2e6a87)
- [Meeting Minutes](https://oplabs.notion.site/Supervisor-Data-Flow-Minutes-14af153ee16280ba8942ecf9d056999a?pvs=73)


# Proposed Solution(s)

This document contains a collection of moderate sized refactoring tasks for the OP Node and OP Supervisor which seek to make our abstractions and patterns better suited to the goals of Interop.

## Distinguishing Node Relationships
First, we should define two different modes of association an OP Node can have with an OP Supervisor:
### External (Read Only) Nodes
External Nodes are OP Node instances which access the Supervisor to ask questions about cross safety for messages or block-heights. A home-node runner may have an External relationship to a Supervisor as an API, for example. These nodes *do not* contribute to the data a Supervisor maintains.

### Owned (Writer) Nodes
Owned Nodes are special OP Nodes which are paired to the Supervisor. There must be *at least* one Owned Node per Chain per Supervisor. Owned nodes respond to requests from the Supervisor to provide sync information (ie log data and local head data). The Supervisor may maintain multiple nodes per chain in order to provide parallelized access or redundancy.

## Specifying External Node Architecture
External Nodes have minimal bearing on the complexity of a Supervisor. That is because the External Nodes *only* make simple queries, and data held by that Node is never considered by the Supervisor. Their connections should be made from the Node to the Supervisor, and a one-way, unauthenticated RPC connection is fine.

However, the OP Node *does* need to take protective measures if it is using an External Supervisor. It is possible that the OP Node in question is deriving its chain based on data which is *not consistent* with the data the External Supervisor is using. When this occurs, the OP Node **must** halt. There is no way to attribute fault, and so the Supervisor is effectively unusable by the Node for this period. The system would resume operation when one side or the other handles a reorg which re-aligns the views.

## Specifying Owned Node Architecture
For Owned Nodes, there are several architecture decisions being made in this document:

### Bi-directional RPC
For Owned Nodes, the Supervisor should initiate the connection. The connection should be an authenticated two-way RPC which the units can communicate over. This gives us strong connectivity and instant signaling mechanisms from the Supervisor down to the OP Node.

### Supervisor as Orchestrator for Sync
In the current architecture, OP Nodes decide to send their updated heads to the Supervisor as part of Derivation. When this happens, the Supervisor reaches out and fetches receipt data. Both of these activities load data into the Supervisor's database, and this presents a problem -- how can the Supervisor interact with multiple Nodes without suffering redundant or conflicting writes?

To solve this, we propose that the Supervisor should be in control of Synchronization. The Supervisor would now take an approach like this:

- Upon startup, establish a connection to the Owned Node
- If the Supervisor database is uninitialized, ask for initialization information from the Node
    - Initialization Info is returned
- Ask the node for its current latest head
- If the Supervisor is missing data for some of the data to the Head, request it
    - The Supervisor may ask multiple Owned Nodes for data
- Repeat the check-and-request for data during runtime

This control flow also allows for the Supervisor to notice and react to discrepancies between nodes:

- During runtime, the Supervisor may request more data from OP Nodes
- It may ask multiple OP Nodes for copies of the same data
- If the data is divergent, we know one of our Owned Nodes is split, and we can halt processing until they recover

### Supervisor as L1 Source
##### This topic has been mentioned previously [in draft](https://github.com/ethereum-optimism/design-docs/pull/136).
In addition to making the Supervisor the centralized controller, we also propose the Supervisor should be the centralized source of L1 Data for Owned Nodes. This is because when OP Nodes act independently, they establish their own L1 connections, which could be divergent from others'. When this happens, there is no tie-breaking to decide which L1 source is canonical, and so the Supervisor is unable to come to meaningful conclusions with the data provided, as it comes from divergent sources.

Instead, we should use the Supervisor as the source of L1 data which Owned Nodes use for Derivation. The way we can accomplish this is:
- Establish an L1 Provider component of the Supervisor which uses the L1 fetching OP Nodes already do. Match existing data formats such that the Supervisor acts mostly as a proxy, and does not introspect the data.
- For Owned Nodes, disable L1 fetching for Derivation.
- Add an RPC API to Owned Nodes which accepts the L1 data and starts Derivation on it.
- When the Supervisor receives a new L1 payload, it calls it down to the Owned Nodes to start their Derivation against the data.

By doing this, we ensure that the data used to construct the Supervisor Database is consistent, and we allow the OP Node to still perform derivation per usual. In addition, by putting Supervisor in control of the data administration, it is able to better predict and control updates to Safe data. Any time it sends new L1 data down, it can immediately listen back for the updates from that data.

## Solution Summary
The prioritized list of changes this Design Document covers are:
- Supervisor as L1 Source (tied for 1st)
    - L1 data collected at Supervisor
    - Owned nodes don't fetch L1 inputs
    - Owned nodes respond to Derivation requests from the Supervisor
- Supervisor as Orchestrator (tied for 1st)
    - Bidirectional RPC
    - Maintaining multiple owned nodes
    - Directing sync operations to owned nodes
    - Fallbacks via multiple owned nodes
    - Diffing and halting via multiple owned nodes (stretch)
- Initialization flow simplification (once Supervisor as Orchestrator is done)
- Supervisor Snapshot Syncing (see below)
- External Node API support

And summarized to a short statement: These changes represent an architectural arrangement between OP Node and OP Supervisor which allows much higher confidence and control over data in the Supervisor, while still leaving the option open for Externally Connected Nodes to follow a Supervisor over an API. The Supervisor gains authority over the L1 data and the sync actions which flow into its database.

### Solution Footnote: Supervisor Snapshot Sync
It isn't well married to the other ideas presented in the solution, but recent architectural conversations have also proposed sync methods for Supervisors to load data without consulting Owned Nodes. For example, the Supervisor Datadir can be copied and used aggressively by other Supervisors, so supporting a "bootstrap" kind of API would allow for easier initialization. This would be effective in combination with Supervisor-driven Orchestration of Log Sync, since the Supervisor would be able to explicitly seek out the missing information from its Owned Nodes. This should be cut into its own ticket, but this design document does not cover this topic in any further detail.

## Resource Usage

### Compute Resources
These changes do not significantly increase the computational load on any components. It actually reduces L1 connections for Supernode groups, and gives more allowance to Home Node Runners to use an API based Supervisor, which is less intensive than doing it at home. The Supervisor cluster itself may take more OP Node resources for redundancy, but we needed that no matter what.

### Work Resources
This makes our plan toward the Stable Devnet more of a proper development plan. I think the current Interop Engineering Staff can execute to this plan alongside our stability and bugfixes on the way to our next milestone.

# Alternatives Considered

## Build a Monolith
Another design takes these same ideas a bit further and pulls all consensus components into one binary, featuring N Derivation Pipelines and one Safety-Index like component for cross-safety tracking. This change has value, but the extreme pivot is not something we currently have an appetite for.

## Do Nothing
We could try to continue to chase stability and consistency issues into our current system without making these architectural changes. However, the cost of this effort would rival the changes being proposed in this document. And if not addressed, we'll still have a confusing story for supporting multiple nodes per supervisor, and still have lots of wasteful complexity management with conflicting L1 views.

# Risks & Uncertainties

###### There are no risks, this team can do anything. There is only upside; WAGMI.

* Edge cases could inflate the complexities described above and make achieving this plan more difficult. We have seen in development of this system that edge cases can significantly increase our complexity. However, as this document describes a *reduction* of complexity through an increase of explicit control, we should have this well in-hand.
