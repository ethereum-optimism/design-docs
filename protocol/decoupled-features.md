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

# Proposed Solution: Client Side Decoupling

Decouple features from hardforks at the client level. Make no changes to the
superchain registry. Instead, allow clients to implement a feature mapping
or hardcoded activation methods for individual features within a hardfork.

CLI flags can be used to toggle features on or off.

This allows features to be developed independently from a hardfork,
allowing the finalized set of features to not impede on protocol development.

### Key Changes

- **RollupConfig Update:** The rollup config can be changed in the node to hold
  features but it **must** not be deserialized from the superchain registry or by
  cli flags. The rollup config can only hold a hardcoded mapping of features to
  hardforks that when deserialized, always results in the same hardcoded mapping.
  In rust for example, the default field value would always be used when deserialized.
- **Activation Check:** Add methods like `IsFeatureActive(timestamp uint64) bool`, which:
    - Look up the hardfork for the feature.
    - Retrieves the hardfork's timestamp (e.g., via existing fields like `IsthmusTime`).
    - Returns true if `timestamp >= hardforkTimestamp`.
  This method _can_ use the mapping in the rollup config as specified above, or use
  hardcoded fallbacks like @ajsutton suggested. Since this is a client side implementation
  detail, it doesn't need to be consistent.

Notice, there are no changes to the superchain registry (SCR). Only the protocol
developers can programmatically change the activated features for testing/development
purposes. Rollup configs read from the superchain registry or through the CLI will
not be able to affect feature activation.

### Implementation Methods

Two implementation options are provided here.

#### Option A: Fallback Activation Methods

These are methods like @ajsutton originally suggested that defer back
to the hardfork timestamp for whether or not a feature is activated.

It's worth noting that this is less flexible for testing since the features
are not configurable through the `RollupConfig` programmatically, but there's
no risk of misconfiguration. This approach is typically used on L1 though
because the features that are actually in a hardfork doesn't change often.

For example, an activation check for the operator fee that is part of the
Isthmus hardfork could look like the following.

```
func IsOperatorFee() {
  return IsIsthmus()
}
```

> [!NOTE]
>
> Notice that with this approach, the method could just be modified to
> return false if the feature needs to be disabled for some reason.


> [!NOTE]
>
> For brevity, activation booleans should read `isFeature` or `isFork`
> rather than `isFeatureEnabled` or `isForkEnabled`.

This approach could add additional work on the backend if individual
feature activation methods are removed in favor of using a hardfork
activation check. This would be done after the features are all developed
and once the set of features for the hardfork is specified.

#### Option B: Embedded RollupConfig Mapping

Potentially a more middle-grounds approach to the above option and
making changes to the superchain registry is to have a feature mapping
present in the `RollupConfig` on the client side.

Importantly, when deserializing a `RollupConfig` from the SCR or CLI,
this mapping would *always* be set to the default, *hardcoded* feature
to hardfork mapping.

Activation methods for features would still be used, but now, instead
of defaulting to a hardcoded hardfork like in Option A, the mapping
would be used to check which hardfork the feature is a part of.

By embedding a mapping in the `RollupConfig`, test and development
environments may individually enable or disable features. This differs
from Option A since all features for a given hardfork are bundled and
would be active unless the method implementation is changed to "return
false". That means the test cannot programmatically toggle individual
features on or off.


# Alternatives Considered

### **Superchain Registry Features**

A more complete writeup on this alternative exists in #297.

This alternative method is to add a map from feature name to hardfork name
in the superchain registry to each `RollupConfig`. This allows significantly
more flexibility in customizing which features are activated when. On the
flipside, it makes a commitment to decoupling features from hardforks for
the foreseeable future. In effect, it also makes it easier to misconfigure
features and hardforks.

While this allows chain operators to customize which features are active,
it makes changes to the superchain registry which is meant for customizable
chain configuration values. A valid argument can be made that once features
are committed to a given hardfork, they are not considered customizable and
should not be a part of the superchain-registry. Further, this method locks
in clients into using the feature mapping for client/protocol side
development.


### **Individual Feature Timestamps**

While the feature-to-hardfork mapping adds flexibility within hardfork feature sets,
a more complete decoupling could use per-feature activation timestamps directly in
`RollupConfig`. For example, a `map[string]uint64` where key is feature name and value
is its independent timestamp. Hardforks would then serve as non-binding labels or presets
(e.g., "isthmus" predefined as activating features X, Y, Z at time T), loaded optionally
for superchain coordination.

> [!NOTE]
>
> In practice features are never actually “bound” to a specific hardfork at the client
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
- **Backward Incompatibility:** If existing hardfork features are migrated to individual
  feature toggles, incomplete mappings could disable features unexpectedly.
- **Performance:** Map lookups in hot paths could add minor latency if not optimized.
  - Note, in practice, there are a number of ways to solve this problem at comp-time
    rather than taking a performance hit at runtime. This could be considered a low impact,
    medium likelihood risk.


[scr]: https://github.com/ethereum-optimism/superchain-registry
[proposed]: https://github.com/ethereum-optimism/design-docs/pull/81

[kona-overrides]: https://github.com/op-rs/kona/blob/main/bin/node/src/flags/overrides.rs
[op-overrides]: https://github.com/ethereum-optimism/optimism/blob/develop/op-service/flags/flags.go#L16-L24
