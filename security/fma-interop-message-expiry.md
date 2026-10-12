# FMA: Interop Message Expiry and SuperchainETHBridge Refunds

|  |  |
| --- | --- |
| Author | Karl Floersch |
| Created at | 2026-10-12 |
| Needs Approval From | TBD (security) |
| Status | Draft |

## Introduction

Interop executing messages can't be forced in through deposits, so a destination sequencer can keep a message from ever being relayed. When that message is a `SuperchainETHBridge` send, the ETH burned on the source chain is stranded. Message expiry lets the source chain learn, through censorship-resistant paths only, that a message can never be delivered, and lets the bridge refund the ETH. It ships in Lagoon (U22). This FMA covers the contract changes that implement it and the node-side window rule they depend on.

**Key architectural aspects:**

- **Export on the destination.** A new `UndeliveredMessageExporter` predeploy (`0x4200000000000000000000000000000000000030`) hashes a message with its own chain as the destination, refuses one its chain relayed, and sends the fact as a withdrawal to the source chain's `L1CrossDomainMessenger`. It isn't an executing message, so it can be forced in as a deposit.
- **Relay on L1.** `L1CrossDomainMessenger.relayUndeliveredMessage` deposits `expireMessage` into its own chain's `L2ToL2CrossDomainMessenger`, only when this chain runs interop, the chain isn't paused, the caller is a real `L1CrossDomainMessenger` (its portal's `SystemConfig` names it), the caller's portal is authorized by this chain's `ETHLockbox`, and the caller is relaying a withdrawal from the `UndeliveredMessageExporter`.
- **Expire on the source.** `L2ToL2CrossDomainMessenger` records each message's send timestamp and marks a message expired when L1 reports it unrelayed more than `expiryPeriod()` after the send. From then on the protocol rejects any relay of it, so expiry and delivery exclude each other. The period lives in proxy storage, set by `initialize(uint256)` (rejects 0 and anything above 365 days); production uses 8 days (`Constants.L2_TO_L2_MESSAGE_EXPIRY_PERIOD`): the 7-day protocol window plus a day of margin.
- **Refund.** `SuperchainETHBridge.refundETH(destination, nonce, from, to, amount)` rebuilds the send's message hash and refunds the sender once, by forced send, only if the messenger marked the message expired. `SuperchainETHBridge` goes to 2.0.0.
- **Window cap.** op-core, kona and op-interop-filter reject a message expiry window override above 7 days (#23276).
- **Fresh install.** On OP Mainnet, Ink and Unichain (mainnet and Sepolia), none of the interop predeploys (`0x…22`–`0x…25`, `0x…30`) has an implementation before Lagoon (implementation slots read 2026-10-12), so there is no earlier messenger or bridge state for the new storage to meet.

**Out of scope:** refunds for anything other than ETH. `SuperchainTokenBridge` / `SuperchainERC20` sends that expire are still stranded; other applications can read `expiredMessages(messageHash)` and build their own undo.

**References:**

- [Spec: ethereum-optimism/specs#960](https://github.com/ethereum-optimism/specs/pull/960)
- [optimism#23286: expire undelivered interop messages](https://github.com/ethereum-optimism/optimism/pull/23286)
- [optimism#23284: refund expired SuperchainETHBridge sends](https://github.com/ethereum-optimism/optimism/pull/23284)
- [optimism#23276: reject message expiry window overrides above 7 days](https://github.com/ethereum-optimism/optimism/pull/23276)
- [optimism#23277: expiry acceptance tests](https://github.com/ethereum-optimism/optimism/pull/23277)
- [optimism#23337: op-interop-mon expiry and refund checks](https://github.com/ethereum-optimism/optimism/pull/23337)
- [FMA: Message Passing Contracts](./fma-message-passing-contracts.md), [FMA: SuperchainETHBridge](./fma-superchainethbridge.md), [FMA: L2ContractsManager](./fma-l2cm.md)

For generic smart contract failure modes, see the [Generic Contract FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-contracts.md). For hardfork-related failure modes, see the [Generic Hardfork FMA](https://github.com/ethereum-optimism/design-docs/blob/main/security/fma-generic-hardfork.md). How they apply here:

- **Storage layout (generic contracts).** The messenger gains send timestamps, `expiredMessages` and the expiry period; the bridge gains `refunded`. Lagoon installs both fresh on production chains, so the risk at Lagoon is limited to devnets that ran earlier interop code. Any later upgrade of either contract must preserve this layout.
- **Initialization (generic contracts).** `L2ContractsManager` calls `initialize(uint256)` inside the Lagoon NUT deposit, so it can't be front-run; upgrades always pass 8 days.
- **Activation (generic hardfork).** The bundle's deploy gas grows (messenger 2.359M, bridge 1.175M, exporter 650k). An under-provisioned deploy fails at activation; the bundle-applies-on-every-production-genesis check (optimism#23097) covers it.

- [ ] Check this box to confirm that generic items have been considered and updated if necessary.

## Failure Modes

### FM1: A message is both relayed on its destination and refunded on its source

#### Description

The safety property is that a message is either relayed on its destination or expired (and, for a bridge send, refunded) on its source, never both. A violation mints ETH on the destination and refunds the same ETH on the source. It becomes possible if any of these hold:

- a node, op-interop-filter or fault-proof absolute prestate accepts a message expiry window above 7 days, so a message can still be relayed after its source has expired it;
- a resend-capable messenger implementation (the 1.3.x line with `resendMessage`) is live on a chain with expiry, for example after a rollback through `ProxyAdmin.upgrade`, which has no version check; anyone could then resend a refunded message and relay it;
- a chain in the cluster runs an exporter that isn't the standard code;
- chain IDs collide across cluster members and the chains in their dependency sets;
- an `L1CrossDomainMessenger` that trusts the L2ToL2CrossDomainMessenger (`0x…23`) as a sender is deployed.

#### Risk Assessment

Critical severity / Low likelihood

#### Mitigations

- Relay and expiry exclude each other in `L2ToL2CrossDomainMessenger`: an expired message can't be relayed.
- The 8-day expiry period exceeds the 7-day window by a day of margin.
- op-core, kona and op-interop-filter reject window overrides above 7 days (#23276).
- Exports come only from the `UndeliveredMessageExporter` predeploy, which refuses a message its chain relayed; L1 accepts them only from a real, lockbox-authorized `L1CrossDomainMessenger` relaying that predeploy's withdrawal.
- Refunds go once per message (`refunded`), to the original sender.

#### Detection

- op-interop-mon (#23337), deployed and alerting for Sepolia and Mainnet: it checks from chain data that no destination relays a message its source marked expired, that every `MessageExpired` matches an unrelayed, exported message, and that every bridge burn is matched by at most one mint. It pages on `expiry_violations_total`.
- With `--supernode-endpoints`, a relay after expiry also counts in `cross_safety_violations_total` once it reaches cross-safe.

#### Recovery Path

1. op-interop-mon alerts.
2. Trigger the guardian pause on L1 (resolved through each chain's `ETHLockbox`, optimism#22760). It stops `relayUndeliveredMessage`, so no new expiries (and no new refunds), and it stops withdrawals and lockbox unlocks. It does not pause anything on L2: `SuperchainETHBridge`, the messenger and `CrossL2Inbox` keep working. Expiry deposits already queued on L1 still land. The op-interop-filter failsafe in the next step runs on each chain's own sequencer, so on chains OP Labs doesn't operate it needs that operator.
3. Trigger the op-interop-filter failsafe (manual override), which makes sequencers reject all interop executing transactions, so no new relays.
4. Fix through an `L2ContractsManager` upgrade or a hard fork. Complex: needs governance.

#### Action Items

- [ ] Deploy op-interop-mon with the expiry checks (#23337) for Sepolia and Mainnet, alerting on `expiry_violations_total`, before Sepolia activation.
- [ ] Every node, op-interop-filter and the Lagoon absolute prestate enforce the 7-day window cap (#23276) before any destination's exporter goes live.
- [ ] Never ship a resend-capable messenger implementation to a chain with expiry; the Lagoon NUT bundle is re-snapshotted from develop with messenger, bridge and exporter at their expiry versions (#23287).
- [ ] Add the pause-plus-failsafe response to the interop runbook.
- [ ] Evaluate a cluster-wide L2 pause mirrored from the L1 guardian pause, so OP Labs can stop bridge sends, mints, refunds and relays on every interop chain without each operator's failsafe (under evaluation for U22).

### FM2: A still-deliverable message is marked expired

#### Description

A message that could still be relayed is marked expired, so the protocol rejects its relay. A bridge sender gets a refund instead, but an application without refund handling loses the message. Causes:

- a production chain runs with a short `expiryPeriod`, for example a devnet override carried to a real chain;
- the recorded send timestamp is wrong;
- a forged report reaches `L1CrossDomainMessenger.relayUndeliveredMessage`.

#### Risk Assessment

High severity / Low likelihood

#### Mitigations

- `initialize(uint256)` rejects 0 and anything above 365 days; upgrades always initialize the messenger with 8 days.
- op-deployer refuses a shorter period for `standard` and `standard-overrides` intents and on mainnet, Sepolia, Holesky and Hoodi.
- `relayUndeliveredMessage` checks that this chain runs interop, that the caller is a real `L1CrossDomainMessenger` named by its portal's `SystemConfig`, that its portal is authorized by this chain's `ETHLockbox`, and that it is relaying a withdrawal from the `UndeliveredMessageExporter`.

#### Detection

- op-interop-mon (#23337) checks each `MessageExpired`: the send timestamp is set, a `SentMessage` in the source block rebuilds to the hash, `undeliveredAt` is past `sentAt + expiryPeriod`, the destination has no relay of it, and the destination's block at `undeliveredAt` contains the export.

#### Recovery Path

- The relay can't be recovered for an expired message. A bridge sender is refunded through `refundETH`. Fixing a wrong period needs an upgrade.

#### Action Items

- [ ] After activation, check `expiryPeriod()` is 8 days on every production interop chain.

### FM3: An undelivered message can't be expired

#### Description

The export or the L1 relay rejects a legitimate report, so the message never expires and bridged ETH stays stranded, as it is today without expiry. Causes include a source chain missing from the destination chain's `ETHLockbox` authorized portals, or the L1 relay running out of gas.

#### Risk Assessment

Medium severity / Low likelihood

#### Mitigations

- If `relayUndeliveredMessage` runs out of gas, the call lands in the caller's failed messages and can be replayed.
- Acceptance tests (#23277) and the end-to-end expire-then-refund test (#23284) exercise the full path.

#### Detection

- User reports of stuck sends; op-interop-mon's `eth_in_flight` shows ETH that stays in flight.

#### Recovery Path

- Replay the failed L1 message, or fix the cluster's lockbox authorization. Simple to moderate.

#### Action Items

- [ ] None beyond the acceptance tests.

## Audit Requirements

This change requires an audit: it is in scope for the U22 (Lagoon) Spearbit contracts audit.
