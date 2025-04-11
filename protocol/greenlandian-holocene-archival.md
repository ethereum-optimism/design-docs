# Project Greenlandian

# Purpose

Make the OP-Stack less complicated by archiving pre-Holocene blocks.

# Summary

Holocene brought important OP-Stack protocol simplifications.
The Holocene upgrade simplifications are currently only forward-looking:
new blocks are batch-submitted and confirmed with leaner more stable rules.
The history of pre-Holocene data still requires major derivation-pipeline complexity in the stack.

Holocene is the "warm and stable" geological epoch we live in today.
"Greenlandian" is the first Age of the Holocene,
named after ice-layers that effectively archived the transition out of the ice age with high resolution.

This design doc proposes a small project to archive the pre-Holocene block data,
so it no longer has to be processed by the op-node, allowing major tech-debt cleanup,
and unblocking greenfield ( ;) ) implementations (Rust!).

No breaking protocol changes are required, this work is platforms-focused.

# Problem Statement + Context

The L2 needs the historical chain inputs to re-process blocks for archive-node purposes.

L2 blocks may be retrieved from multiple different sources:
- reproduce from L1 data
- import as ready L2 block data

L1 blobs have a limited data-availability time duration.
To challenge an invalid claim of L2 outputs on L1, the state-transition needs to be proven.
For this to be proven by anyone, the input data needs to be available, to reproduce the computation.

Actors with an interest in the system are assumed to have access to an older already widely agreed-upon state,
but need access to the last few weeks of input data permissionlessly to not rely on the L2 sequencer,
which can otherwise make false output claims against their interest.

Given this limited availability period, there already exists a widely-used archiving strategy today.
An archive that is verified against L1 block commitments which are agreed upon.

Archiving the blobs and re-running derivation is increasingly less and less sustainable:
- Blobs may contain different L2s. It is more data than any single chain may need.
  Limiting archival just to 1 inbox contract helps,
  but this is limited to a single-chain inbox design, incompatible with shared inbox designs.
- Derivation also relies on L1 receipt data, which thus also needs to be archived.
  Many L1 nodes prune older receipts. Receipt availability today is slowly disappearing,
  with work such as EIP-4444 moving to archive it.
- Legacy derivation adds significant complexity to new implementations,
  and is significant maintenance burden, hurting implementation progress with legacy design choices.
  Blob archives are not useful to new implementations that aim to avoid the pre-Holocene derivation complexity.

## Trust assumption

The L2 block archival solution trust-model is equivalent with the L1 blob archival trust model,
with one new minor theoretical assumption:
**The canonical L2 chain can be established by the node,
with a delay from the actual tip of the chain as long as blobs are available for.**

Without *any* blob archival, this is `MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS (4096 L1 beacon epochs)`,
or approximately 18 days. With some blob archival, this can be increased.

In practice L2s have a bridge with widely accepted finality of around a week,
determining a canonical chain as user, at 2-3x the time during node bootstrapping, should be of relatively ease.

## `op-node` cleanup

Once we archive pre-Holocene blocks, we can remove and cleanup the op-node implementation as following:
- Remove legacy parts of the `rollup/engine` package that maintains fragile forkchoice state workarounds such as
  "backup unsafe L2 blocks" and "pending safe blocks".
- Remove derivation-stage multiplexing from the `rollup/derive` package.
- Remove the legacy `ChannelBank` and `BatchQueue` logic.

Cleanup of the `engine` package allows us to better reason about the execution-engine failure cases
and maintain a simplified forkchoice state for better extensibility.

Cleanup of the `derive` package allows us to cut some of the most complicated derivation code,
simplifying maintenance and the fault-proof that is built on top of this.

# Proposed Solution

If we are deprecating pre-Holocene derivation,
we need a way to reliably archive and distribute the post-derivation work: the L2 blocks themselves.

Archival of execution-layer blocks is already standardized on L1,
something the L2 stack should already be compatible with.

Using the L1 Ethereum ERA file format we are able to archive the chain in a universal cross-client ethereum format.

In op-geth, we should be able to run two geth commands: `export-history`, `import-history`.

```bash
op-geth export-history --datadir=... --datadir.ancient=... <dir> <first> <last>

# help:
#   The export-history command will export blocks and their corresponding receipts
#   into Era archives. Eras are typically packaged in steps of 8192 blocks.
```

```bash
op-geth import-history --datadir=... --datadir.ancient=... \
    --db.engine=pebble --state.scheme=path \
    --op-network=... \
    <dir>

# help:
#   The import-history command will import blocks and their corresponding receipts
#   from Era archives.
```

This functionality was introduced in Geth L1 here: https://github.com/ethereum/go-ethereum/pull/26621
The general era file-format is documented here:
https://github.com/status-im/nimbus-eth2/blob/stable/docs/e2store.md#era-files
The Era1 file format is documented in Geth here:
https://github.com/ethereum/go-ethereum/blob/4cdd7c86314e5f5a98fa68e928deaa99c026f40d/internal/era/builder.go#L34-L73
And Nethermind officially started supporting Era files in v1.31.0:
https://github.com/NethermindEth/nethermind/releases/tag/1.31.0

Note that once we archive Era files,
we can also start running EIP-4444 nodes that have partial history,
i.e. that do not retain all blocks since genesis.
This makes full-nodes more viable for users with limited disk-space.

## Resource Usage

This removes derivation complexity,
and history import now does not need derivation processing,
or L1 blob and receipt archives.
Instead, a L2 block (receipts included) archive is used to sync,
which can be read from disk, removing RPC bottlenecks to spin up an archive node.

## Single Point of Failure and Multi Client Considerations

Era files are a standard format and archival strategy from L1,
and can be checksummed and compared to ensure consistency across nodes.

# Alternatives Considered

Syncing L2 blocks exclusively via Execution-Layer P2P sync is possible,
but relies on node connectivity, and reduces the network to rely on some node still providing the history.
Archive files provide better trust guarantees, efficient data replication and distribution,
and faster syncing without P2P network bottlenecks.

# Risks & Uncertainties

The Era file format is expected to be compatible with L2 out of the box,
but tools/clients might need minor adjustments,
if the export or import code-path does not instantiate the OP-Stack EVM correctly from the chain-config.

OP-Mainnet era-file import is a special case:
for archival nodes it will need to be imported into an existing database instead of freshly initialized genesis,
due to the Bedrock upgrade state migration.

For full-nodes, it may be possible to import, and then still snap-sync to a recent state,
while skipping the block-header/block-body/receipt P2P downloader stages.

No changes are expected, but protocol-support may be needed to troubleshoot any op-geth quirks.
