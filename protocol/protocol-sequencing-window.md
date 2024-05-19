# Purpose

The purpose of this document is to standardize the "sequencing window" from the point of view
of the protocol. This solves a problem for [deposits of executing messages](https://github.com/ethereum-optimism/design-docs/pull/13).
This also guarantees a more standard approach to managing the sequencing window across chains, which
is important for chain operators of L3 networks.

# Summary

A standard sequencing window is defined by 12 hours or [more accurately](https://github.com/ethereum-optimism/specs/issues/199) 3600 base layer blocks.
This is not legible to the chain software and also creates issues for L3s since the sequencing window is defined in base layer
blocks rather than being defined based on time.

# Problem Statement + Context

Right now, the sequencing window is defined in terms of base layer blocks and there is no real way to know that nodes on the network are
running with an appropriate value since it is part of a config. Depending on the blocktime of the base network, the sequencing window may
want to be defined differently.

For example, an L2 with a 3600 block sequencing window on a L1 with 12s blocktime results in a sequencing window of 12 hours while an L3
with a 3600 block sequencing window on an L2 with a 2s blocktime results in a sequencing window of 2 hours. This means that the infrastructure
requirements for operating an L3 is much higher than the infrastructure requirements of running an L2.

One question to ask is whether or not we want to allow the sequencing window of chains to be different. As long as the value is legible,
I believe that it is good for the ecosystem to enable them to be different as it will create competition between chains at the infra layer
for creating the most reliable infra to result in faster finality for the chain. From the point of view of the chain, once the sequencing
window elapses, applications within the EVM can have a guarantee that there **will not be a reorg**, assuming L1 finality is shorter than the
sequencing window.

# Proposed Solution

Add the sequencing window to the `SystemConfig`. It is added to the `SystemConfig` rather than to the `SuperchainConfig` to enable the values
to be different per network.

The sequencing window can be updated via an event just like other `SystemConfig` values. Using the [following approach](https://github.com/ethereum-optimism/specs/issues/122),
there is no need to push the sequencing window as part of the L1 attributes transaction on every L2 block. Another conversation on the
approach is happening [here](https://github.com/ethereum-optimism/specs/pull/194).

The sequencing window would be stored in both the `SystemConfig` and the `L1Block` contract with a system invariant that the values
are consistent between the two locations. Only the chain operator would be able to modify the sequencing window. This allows chain
operators to modify the sequencing window, which is particularly useful for L3 networks.

## Chain Standardness

We cannot define in terms of time for standardness since time is relative to the blocktime of the basechain.
We need to be sure that our standardness definition sets a lower bound and an upper bound and assume that these values are sane for ranges of blocktimes.
Ideally we do not need to have different ranges for L2s and L3s and let the chain operator assume risk if they want to set it to a crazy low value
under fast base blocktimes.

Standard should be defined between 900 and 14400, the following table shows the lower bound and upper bound
in time given various base chain blocktimes.

| Basechain Blocktime   | Lower Bound   | Upper Bound   |
|------------|------------|------------|
| 12s | 3h | 48h |
| 16s | 4h | 64h |
| 2s | 30m  | 8h |
| 1s | 15m  | 4h |

It is in the incentive for the chain operator to run with a low sequencer window to provider a credible
commitment towards faster finality. In the lowest possible case, 4h is plenty of time to ensure that
data is posted by the `batch-submitter`.

# Alternatives Considered

## Do nothing

Hardcode and assume the sequencing window is always going to be 3600. This could work but creates rigid assumptions about the sequencing
window between chains. This isn't extensible for L3 networks, as 3600 base chain blocks where the base chain is 2 seconds is 2 hours.
This is a huge diff in quality of life and uptime requirements.

Since the sequencing window is simply a config option, it is possible that it can be changed between subsequent restarts of the software.
Without it being on chain, it is trusted that the value is set correctly. To be a [standard config chain](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/configurability.md?plain=1),
it is enforced only via the fault proof that it is set correctly. Even if some nodes were running their software with a different
value, it is likely that they would more often than not derive the same chain.

The main action item here would be to define a constant in the `op-node` for the standard sequencing window and use that value
if the value is not in the JSON deploy config file.

## Define sequencing window in time rather than blocks

If the sequencing window was defined in units of time (seconds) rather than in units of base chain blocks, then it would be easier
to handle the fact that the time between base chain blocks can vary. The main downside with this is that the system now needs some
legibility into the blocktime of the base chain, which can be hardcoded but the blocktime of L1 ethereum has changed in the past.
It may also change into the future given designs of [ePBS](https://ethresear.ch/t/why-enshrine-proposer-builder-separation-a-viable-path-to-epbs/15710)
where the blocktime is increased.

# Risks & Uncertainties

- Semantics around when to apply the changes to the sequencing window would need to be clearly defined
- We would want to provide guidelines on the range of values that we consider standard
