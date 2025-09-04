# FMA: U16a Maintenance Upgrade

|---------------------|--------------------------|
| Author              | Kelvin Fichter           |
| Created at          | YYYY-MM-DD               |
| Needs Approval From | Matt Solomon, Josep Bove |
| Other Reviewers     |                          |
| Status              | Draft                    |

## Introduction

U16a is a maintenance upgrade that replaces U16. Its primary purpose is to temporarily remove interop-specific code introduced in U16 while introducing foundational support for system-level feature toggles.

The interop withdrawal-proving code paths are removed in U16a following feedback from chain operators and security reviewers. We intend to reintroduce interop functionality in a future upgrade after working more closely with partners' security teams.

### Removal of interop withdrawal-proving code

Chains can no longer prove withdrawals when interop is enabled. ETHLockbox support remains for chains that previously upgraded to U16. Chains skipping directly from U15 => U16a cannot adopt ETHLockbox until a later upgrade.

### Introduction of system-level feature toggles

The upgrade implements system-level feature toggles via the `SystemConfig` contract. ETHLockbox is the first feature gated behind this mechanism. Feature flags are string-based and settable by chain admins.

### OPContractsManager (OPCM) updates

The OPCM supports both `U15 => U16a` and `U16 => U16a` upgrade paths. It includes explicit protections against misuse of dev feature flags in production. Upgrade tests cover both paths, and rollout will be validated on betanet first.

### Pause mechanism identifier change

Chains without ETHLockbox will use the `OptimismPortal` address as their identifier instead of the lockbox address. Incident response runbooks will be updated accordingly.

## Failure Modes

### FM1: Misconfigured Feature Toggles

#### Description

Chains may incorrectly configure the system-level feature toggles introduced in U16a. This includes enabling ETHLockbox flag without having the ETHLockbox contract deployed, failing to enable ETHLockbox when it's needed, or setting arbitrary unsupported feature flags.

#### Risk Assessment

- Impact: LOW/MEDIUM
  - Reasoning: Most misconfigurations have no functional effect or cause temporary bridge issues that can be resolved by admin intervention. The most impactful scenario is failing to enable ETHLockbox when needed, which temporarily breaks the bridge.
- Likelihood: LOW/MEDIUM
  - Reasoning: Feature toggle system is new, increasing chance of operator error. However, validator checks and clear documentation should minimize risks.

#### Mitigations

- ETHLockbox can only be enabled if lockbox address is set (requires U16 upgrade path).
- Admin can set toggle post-upgrade if initially misconfigured.
- Clear operational documentation and runbooks.

#### Detection

- Monitor for portal/lockbox balances.
- Monitor for unexpected feature flag changes.

#### Action Items

- [ ] Update standard validator to verify lockbox/feature flag configuartion
- [ ] Set up system feature flag monitor
- [ ] Set up portal/lockbox balance monitor

#### Recovery Path(s)

- Correct misconfigured toggles via admin transactions.
- No special recovery procedures beyond standard admin intervention.

### FM2: ETHLockbox Migration Gap

#### Description

Chains upgrading from U15 directly to U16a cannot adopt ETHLockbox until a later upgrade. This creates operational complexity as some chains have ETHLockbox while others don't, leading to testing asymmetry and incident response divergence in pause mechanism runbooks.

#### Risk Assessment

- Impact: LOW
  - Reasoning: Operational overhead is minimal and doesn't affect core functionality or safety. Primarily creates complexity in testing and operational procedures.
- Likelihood: HIGH
  - Reasoning: This is an intentional design decision that will definitely occur for chains following the U15 => U16a upgrade path.

#### Mitigations

- Testing covers both scenarios since major chains exist in both categories.
- Runbooks and supporting documentation will be updated to prevent operator confusion.
- Clear documentation of which chains have ETHLockbox and which don't.

#### Detection

- Track which chains follow which upgrade path.
- Monitor for operator confusion in incident response procedures.

#### Action Items

- [ ] Update incident response runbooks to handle both ETHLockbox and non-ETHLockbox chains
- [x] Ensure test coverage includes both scenarios
- [ ] Do a training session about this new state of affairs

#### Recovery Path(s)

- No recovery needed as this is expected behavior.
- Chains can adopt ETHLockbox in future upgrades.
- Operational procedures handle both pathways as designed.

### FM3: OPContractsManager Dual-Path Upgrades

#### Description

The OPContractsManager supports both `U15 => U16a` and `U16 => U16a` upgrade paths, creating complexity in the upgrade logic. Issues could arise from incorrect branching between upgrade paths or accidental enabling of dev feature flags in production environments.

#### Risk Assessment

- Impact: MEDIUM/HIGH
  - Reasoning: Incorrect upgrade path branching could leave chains stuck mid-upgrade or misconfigured. Dev feature flags in production could expose interop code paths.
- Likelihood: LOW
  - Reasoning: Dual-path upgrade logic is tested in CI and betanet. Dev feature flags have hardcoded protections preventing production use.

#### Mitigations

- Dual-path upgrade tested thoroughly in CI and betanet environments.
- `verifyOPCM` function should verify no dev flags are enabled.
- Hardcoded protections make it impossible to enable dev feature flags in production.
- Clear upgrade procedures and checklists for each path.

#### Detection

- Upgrade test suite validates both paths.
- Betanet validation before mainnet deployment.

#### Action Items

- [ ] Validate both upgrade paths on betanet
- [ ] Have VerifyOPCM check for dev feature flags in production

#### Recovery Path(s)

- Manual intervention may be required to resolve stuck upgrades.

### FM4: Removal of Interop Withdrawal-Proving Code

#### Description

U16a removes interop withdrawal-proving code paths that were introduced in U16. While no known chains or tooling currently depend on these code paths, there's a risk that some undiscovered dependencies exist.

#### Risk Assessment

- Impact: MEDIUM
  - Reasoning: If undiscovered dependencies exist, removal could break functionality for chains or tooling that relied on these paths.
- Likelihood: LOW
  - Reasoning: No known dependencies have been identified, and partner communications will confirm this before rollout.

#### Mitigations

- Partner communications planned to confirm no dependencies before rollout.
- Staged rollout allows for early detection of issues.

#### Detection

- Partner feedback during pre-rollout communications.
- Community reports of broken functionality.

#### Action Items

- [ ] Complete partner communications to confirm no dependencies
- [ ] Monitor for issues during staged rollout

#### Recovery Path(s)

- Revert to U16 if critical dependencies are discovered.
- Develop alternative solutions for affected parties.
- Delay full rollout until dependencies are resolved.
