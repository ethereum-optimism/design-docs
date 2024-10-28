<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Reference Notes](#reference-notes)
- [Generic items we need to take into account for any smart contract upgrade:](#generic-items-we-need-to-take-into-account-for-any-smart-contract-upgrade)
  - [Proxy Initialization Failure](#proxy-initialization-failure)
  - [New Implementation Overwrites (wrong) Slots](#new-implementation-overwrites-wrong-slots)
  - [Proxy Initialized to Wrong Values](#proxy-initialized-to-wrong-values)
  - [Can’t Upgrade Implementation](#cant-upgrade-implementation)
  - [Able to Reinitialize Implementation](#able-to-reinitialize-implementation)
  - [Backwards-incompatible ABIs](#backwards-incompatible-abis)
  - [Invalid `DisputeGameFactory.setImplementation` execution](#invalid-disputegamefactorysetimplementation-execution)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### Reference Notes

Mitigations and Recovery paths for referenced generic items below should be implemented
on a per-FMA basis. That is, since upgrades vary in how the below failure modes apply,
FMAs that reference this document should outline the specific mitigations and recovery
paths for the failure modes outlined below.

### Generic items we need to take into account for any smart contract upgrade:

#### Proxy Initialization Failure

- **Description:**
  Unable to initialize the proxy due to a logic error.

- **Risk Assessment:**
  High severity since it allows arbitrary initialization.
  Low likelihood since the initialization is a minimal operation.

- **Mitigations:**
  Extensive unit and end-to-end testing for proxy initializations.
  Proxy initializations are verified through simulations prior to execution.

- **Detection:**
  Call `initialize(..args)` or by simulating the effort.

- **Recovery Path(s):**
  Call `initialize`, or upgrade the implementation to perform the initialization.


#### New Implementation Overwrites (wrong) Slots

- **Description:**
  An upgrade to a smart contract that inadvertently overwrites the wrong storage values in proxy.

- **Risk Assessment:**
  High severity due to arbitrary slot bounds.
  Low likelihood since this is well looked out for.

- **Mitigations:**
  Storage layouts are validated to only extend and not rewrite the slots of the previous version.

- **Detection:**
  If this error were missed, it could be detected by looking at the storage values in the proxy or recognizing an inconsistency in variables stored in overwritten slots.
  Alternatively, we could detect it from monitoring or user reports that something seems off/incorrect.

- **Recovery Path(s):**
  Deploy new implementation with a corrected storage layout and write correct values to storage.
  Requires an upgrade.


#### Proxy Initialized to Wrong Values

- **Description:**
    The proxy is initialized with the wrong values.

- **Risk Assessment:**
  High severity since arbitrary values can be incorrectly used.
  Low likelihood since proxy initialization has a small surface area and is well verified/tested.

- **Mitigations:**
  Validate initialized values through extensive unit testing.
  - See: Instances of `test_initialize_succeeds()` such as the one [here](https://github.com/ethereum-optimism/optimism/blob/e6ef3a900c42c8722e72c2e2314027f85d12ced5/packages/contracts-bedrock/test/L1/L1CrossDomainMessenger.t.sol#L37-L44).
  - See: Deploy script checks incorporated in [ChainAssertions](https://github.com/ethereum-optimism/optimism/commit/e6ef3a900c42c8722e72c2e2314027f85d12ced5#diff-0f78978618d5f98971fee611fc5476e58643902051d969e0616d5d91a05cff9c) when `_isProxy` = `true`.
  Simulations prior to performing the upgrade, verifying the initialized values.

- **Detection:**
  A simulation can show an unintended initialized proxy value.
  Monitoring or user reports.

- **Recovery Path(s):**
  Repair the upgrade path (if needed).
  Upgrade the smart contract.


#### Can’t Upgrade Implementation

- **Description:**
  Contracts are unable to be upgraded.

- **Risk Assessment:**
  High severity since it effectively bricks an arbitrary contract in the system.
  Low likelihood since contract upgrades are verified and follow industry-wide, standardized patterns.

- **Mitigations:**
  Validate through extensive unit testing.
  Perform simulations that demonstrate upgradability.

- **Detection:**
  Simulate a contract upgrade to check for failure.

- **Recovery Path(s):**
  Upgrade contracts and/or proxies.
  Update `op-chain-ops` if needed.


#### Able to Reinitialize Implementation

- **Description:**
  A third party is able to reinitialize the contract to arbitrary values.

- **Risk Assessment:**
  High severity since an arbitrary actor can perform an action with arbitrary values.
  Low likelihood since authentication is a critical, tested and validated property of the system.

- **Mitigations:**
  Validate through extensive unit testing.
  - See: [Initializable.t.sol](https://github.com/ethereum-optimism/optimism/blob/e6ef3a900c42c8722e72c2e2314027f85d12ced5/packages/contracts-bedrock/test/vendor/Initializable.t.sol).
  Fuzz testing with random values.
  Invariant testing that authentication properties hold.

- **Detection:**
  Simulate an arbitrary reinitialization.
  Alternatively, we could detect it from monitoring or user reports that something seems off/incorrect.

- **Recovery Path(s):**
  Perform an upgrade that prevents the reinitialization.


#### Backwards-incompatible ABIs

- **Description:**
  Functionality expected to hold across an upgrade or update breaks compatibility with prior systems.

- **Risk Assessment:**
  Low severity since liveliness is not impacted.
  Low likelihood since ABI changes are validated.

- **Mitigations:**
  Validate ABI diffs.

- **Detection:**
  Tooling breaking or partners/integrators telling us something broke.

- **Recovery Path(s):**
  Update integration paths for backwards-compatibility.
  Some integrations (e.g. immutable contracts) may not be fixable.
  Perform an upgrade to fix breaking change.


#### Invalid `DisputeGameFactory.setImplementation` execution

This item is detailed in [./fma-generic-hardfork.md](./fma-generic-hardfork.md#invalid-disputegamefactorysetimplementation-execution)
