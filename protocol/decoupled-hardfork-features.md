# Purpose

The purpose of this design document is to propose a mechanism for decoupling
individual features from their hardforks in the OP Stack. This will enhance
flexibility in feature activation, enabling granular control for testing,
custom chain configurations, and phased rollouts while maintaining
coordination across the superchain. The design aims to improve modularity
without introducing excessive complexity, ultimately supporting more robust
development and deployment practices of OP Stack software.

# Background

The OP Stack relies on the `RollupConfig` to specify chain parameters,
including hardfork activation timestamps (e.g., `canyonTime`, `deltaTime`).
These configurations are centralized in the [`superchain-registry`][scr]
repository, as TOML files under `superchain/configs/`. For chains that are
part of the superchain, activations are inherited from a shared timestamp
called the `superchain_time` to ensure synchronized upgrades.

Currently, when a hardfork timestamp is reached in OP Stack client software,
all features within that hardfork (e.g., EIPs, protocol changes) activate
simultaneously. Bundling features in this way simplifies coordination but
limits flexibility, such as enabling individual features for testing or
delaying specific ones without forking the config.

In the past, contributors of the OP Stack have had issues surrounding
hardfork features. For example, long before the Isthmus hardfork, the
Operator Fee Feature was [proposed][proposed] and approved. The feature
was considered to be a part of the upcoming Isthmus hardfork. Rust
implementations of OP Stack components went ahead and implemented the
feature. For months, the feature sat idle. After it was eventually merged,
there were issues in the execution client changes for the feature. Removing
implementations of the feature was considered, but the effort to remove it
was not seen as worthwhile. After a while, the feature was finally
complete. It effectively alone delayed the hardfork by a month or so.

This could have been avoided by being able to implement the feature using
a feature-specific toggle in the `RollupConfig`. The feature toggle would
be toggled on in the [`superchain-registry`][scr] once the feature is ready.
In this way, features could be implemented and left in client software
without being connected to hardfork. The key part: *no changes would
need to be made to client software to enable the feature*, only enabling
the feature inside the [`superchain-registry`].


# Problem Statement

Bundling features within hardforks limits flexibility:

- All features in a hardfork activate at once, preventing selective enablement
  for testing or debugging.
- Custom chains or testnets cannot easily remap or delay individual features
  without forking entire client implementations.
- CLI overrides are limited to hardfork timestamps, not per-feature adjustments.
- The set of features for a given hardfork must be determined prior to
  implementation, locking and committing to scope.

This rigidity hinders iteration, increases deployment risks, and complicates
superchain governance.


# Current Approach

- Hardforks are defined by named timestamps in the `RollupConfig`
  (e.g., `fjordTime: uint64`).
- Node logic checks `timestamp >= hardforkTime` to enable the all features
  in the hardfork.
- Features are implicitly tied to their hardfork; no per-feature granularity.
- CLI overrides exist for hardfork timestamps (e.g., `--override.fjord=123456`
  in `op-node`).
    - Overrides in the `op-node` exist [here][op-overrides]. In the `kona-node`,
      [here][kona-overrides].


# Proposed Solution

Decouple features from hardforks by modifying `RollupConfig`s to include a
mapping from feature names to hardfork names (e.g., a `map[Feature]Fork`
where the key is feature like "operatorFee" and value is a hardfork like "isthmus").
If a feature is absent from the map, it remains inactive.

### Key Changes

- **Superchain Registry Updates:** Add a mapping from features to their hardfork
  in the `superchain.toml` (superchain-wide config) as well as individual chain
  configs if an override is needed.
- **RollupConfig Update:** Add a field to map feature to its hardfork. For example,
  a `features` field in the `RollupConfig` is a map from feature to hardfork name.
- **Activation Check:** Add methods like `IsFeatureActive(timestamp uint64) bool`, which:
    - Look up the hardfork for the feature.
    - Retrieves the hardfork's timestamp (e.g., via existing fields like `IsthmusTime`).
    - Returns true if `timestamp >= hardforkTimestamp`.
- **Client-Level Overrides:** Extend CLI flags for direct per-feature timestamp overrides
  (e.g., `--override.feature.eip4844=isthmus` or `--override.feature.eip4844.timestamp=123456`),
  similar to current hardfork overrides.
- **Superchain Compatibility:** For chains with `superchain_time`, inherit mappings from
  a central registry, but allow per-chain overrides.

These changes allow *both* chain operators and client/protocol developers to customize
which features are active in which hardfork.

### Implementation Details

For demonstration purposes, an updated to `superchain.toml` for the `Jovian` hardfork
would look like the following.

> [!NOTE]
>
> This is pulled from `superchain-registry/superchain/configs/mainnet/superchain.toml`
> in the [`superchain-registry`][scr] repository.

```Diff
name = "Mainnet"
protocol_versions_addr = "0x8062AbC286f5e7D9428a0Ccb9AbD71e50d93b935"
superchain_config_addr = "0x95703e0982140D16f8ebA6d158FccEde42f04a4C"

[hardforks]
canyon_time = 1704992401   # Thu 11 Jan 2024 17:00:01 UTC
delta_time = 1708560000    # Thu 22 Feb 2024 00:00:00 UTC
ecotone_time = 1710374401  # Thu 14 Mar 2024 00:00:01 UTC
fjord_time = 1720627201    # Wed 10 Jul 2024 16:00:01 UTC
granite_time = 1726070401  # Wed 11 Sep 2024 16:00:01 UTC
holocene_time = 1736445601 # Thu 9 Jan 2025 18:00:01 UTC
isthmus_time = 1746806401  # Fri 9 May 2025 16:00:01 UTC
+ jovian_time = 20000000000 # TBC

+ [features]
+  minimumBaseFee = "jovian"
+  configurableCallData = "jovian"
+  operatorFeeFix = "jovian"

[l1]
chain_id = 1
public_rpc = "https://ethereum-rpc.publicnode.com"
explorer = "https://etherscan.io"
```

And in an individual chain’s config file (note the current pattern is that
superchain-wide / inherited data is just duplicated into individual chains’
config files so that it is explicit):

```Diff
name = "OP Mainnet"
public_rpc = "https://mainnet.optimism.io"
sequencer_rpc = "https://mainnet-sequencer.optimism.io"
explorer = "https://explorer.optimism.io"
superchain_level = 2
governed_by_optimism = true
superchain_time = 0
data_availability_type = "eth-da"
chain_id = 10
batch_inbox_addr = "0xFF00000000000000000000000000000000000010"
block_time = 2
seq_window_size = 3600
max_sequencer_drift = 600

[hardforks]
  canyon_time = 1704992401 # Thu 11 Jan 2024 17:00:01 UTC
  delta_time = 1708560000 # Thu 22 Feb 2024 00:00:00 UTC
  ecotone_time = 1710374401 # Thu 14 Mar 2024 00:00:01 UTC
  fjord_time = 1720627201 # Wed 10 Jul 2024 16:00:01 UTC
  granite_time = 1726070401 # Wed 11 Sep 2024 16:00:01 UTC
  holocene_time = 1736445601 # Thu 9 Jan 2025 18:00:01 UTC
  isthmus_time = 1746806401 # Fri 9 May 2025 16:00:01 UTC
+ jovian_time = 20000000000 # This is automatically generated because superchain_time = 0. Custom chains can override it.

 [features]
+ minimumBaseFee = "jovian" # This is automatically generated because superchain_time = 0.  Custom chains can override it.
+ configurableCallData = "jovian" # This is automatically generated because superchain_time = 0.  Custom chains can override it.
+ operatorFeeFix = "jovian" # This is automatically generated because superchain_time = 0.  Custom chains can override it.

[optimism]
  eip1559_elasticity = 6
  eip1559_denominator = 50
  eip1559_denominator_canyon = 250

[genesis]
  l2_time = 1686068903
  [genesis.l1]
    hash = "0x438335a20d98863a4c0c97999eb2481921ccd28553eac6f913af7c12aec04108"
    number = 17422590
  [genesis.l2]
    hash = "0xdbf6a80fef073de06add9b0d14026d6e5a86c85f6d102c36d3d8e9cf89c2afd3"
    number = 105235063
  [genesis.system_config]
    batcherAddress = "0x6887246668a3b87F54DeB3b94Ba47a6f63F32985"
    overhead = "0x00000000000000000000000000000000000000000000000000000000000000bc"
    scalar = "0x00000000000000000000000000000000000000000000000000000000000a6fe0"
    gasLimit = 30000000

[roles]
  SystemConfigOwner = "0x847B5c174615B1B7fDF770882256e2D3E95b9D92"
  ProxyAdminOwner = "0x5a0Aae59D09fccBdDb6C6CcEB07B7279367C3d2A"
  Guardian = "0x09f7150D8c019BeF34450d6920f6B3608ceFdAf2"
  Challenger = "0x9BA6e03D8B90dE867373Db8cF1A58d2F7F006b3A"
  Proposer = "0x473300df21D047806A082244b417f96b32f13A33"
  UnsafeBlockSigner = "0xAAAA45d9549EDA09E70937013520214382Ffc4A2"
  BatchSubmitter = "0x6887246668a3b87F54DeB3b94Ba47a6f63F32985"

[addresses]
  AddressManager = "0xdE1FCfB0851916CA5101820A69b13a4E276bd81F"
  L1CrossDomainMessengerProxy = "0x25ace71c97B33Cc4729CF772ae268934F7ab5fA1"
  L1ERC721BridgeProxy = "0x5a7749f83b81B301cAb5f48EB8516B986DAef23D"
  L1StandardBridgeProxy = "0x99C9fc46f92E8a1c0deC1b1747d010903E884bE1"
  OptimismMintableERC20FactoryProxy = "0x75505a97BD334E7BD3C476893285569C4136Fa0F"
  OptimismPortalProxy = "0xbEb5Fc579115071764c7423A4f12eDde41f106Ed"
  SystemConfigProxy = "0x229047fed2591dbec1eF1118d64F7aF3dB9EB290"
  ProxyAdmin = "0x543bA4AADBAb8f9025686Bd03993043599c6fB04"
  AnchorStateRegistryProxy = "0x1c68ECfbf9C8B1E6C0677965b3B9Ecf9A104305b"
  DelayedWETHProxy = "0x21429aF66058BC3e4aE4a8f2EC4531AaC433ecbC"
  DisputeGameFactoryProxy = "0xe5965Ab5962eDc7477C8520243A95517CD252fA9"
  FaultDisputeGame = "0x5738a876359b48A65d35482C93B43e2c1147B32B"
  MIPS = "0xF027F4A985560fb13324e943edf55ad6F1d15Dc1"
  PermissionedDisputeGame = "0x1Ae178eBFEECd51709432EA5f37845Da0414EdFe"
  PreimageOracle = "0x1fb8cdFc6831fc866Ed9C51aF8817Da5c287aDD3"
```

# Alternatives Considered

### **Individual Feature Timestamps**

While the feature-to-hardfork mapping adds flexibility within hardfork feature sets,
a more complete decoupling could use per-feature activation timestamps directly in
`RollupConfig`. For example, a `map[string]uint64` where key is feature name and value
is its independent timestamp. Hardforks would then serve as non-binding labels or presets
(e.g., "isthmus" predefined as activating features X, Y, Z at time T), loaded optionally
for superchain coordination.

> [!NOTE]
>
> In practice features are never actually “bound” to a specific hardfork at the clien
> level since they can be disabled or enabled via cli flags or a custom rollup config.

This approach:

- Eliminates hardfork dependency entirely for feature checks.
- Simplifies overrides (direct timestamp per feature via CLI).
- Enhances modularity for custom chains or phased rollouts.
- Risks less coordination if not enforced via governance/tools.

It simplifies the level of indirection between hardfork and individual features, as it
avoids indirect lookups while still supporting bundled activations through presets.

On the other hand, it increases the surface area for misconfigurations where timestamps
are incorrectly set for various features. The surface area is increased since there are
many more features than hardforks. If one client uses the backwards-compatible method of
implementing the feature set, and another doesn’t, one wrong timestamp could result in a
chain split.

### **Leave it Unchanged**

One option (a bit of a steel man argument) is to leave the current method unchanged.
One could argue that the problem statement is a result of the hardfork upgrade process
- not the actual implementation method. Instead of handling this at the implementation
level, fix the upgrade process so that a hardfork’s feature set is properly established
prior to implementation.

# Risks

- **Increased Complexity:** Adding maps/methods complicates config management and client
  logic, risking bugs in activation checks.
- **Coordination Challenges:** Decoupling might lead to inconsistent feature enablement
  across superchain, undermining unified upgrades.
- **Backward Incompatibility:** If existing hardfork features are migrated to individual
  feature toggles, incomplete mappings could disable features unexpectedly.
- **Performance:** Map lookups in hot paths could add minor latency if not optimized.
  - Note, in practice, there are a number of ways to solve this problem at comp-time
    rather than taking a performance hit at runtime. This could be considered a low impact,
    medium likelihood risk.
- **Governance:** Requires updated processes in `superchain-registry` for defining/validating
  feature mappings or timestamps.



[scr]: https://github.com/ethereum-optimism/superchain-registry
[proposed]: https://github.com/ethereum-optimism/design-docs/pull/81

[kona-overrides]: https://github.com/op-rs/kona/blob/main/bin/node/src/flags/overrides.rs
[op-overrides]: https://github.com/ethereum-optimism/optimism/blob/develop/op-service/flags/flags.go#L16-L24
