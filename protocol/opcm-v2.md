# OPContractsManager v2

## Context and Problem Statement

OPContractsManager (OPCM) v1 is the contract that manages deployments and upgrades for OP Stack chains. Each upgrade gets its own dedicated OPCM contract. While OPCM v1 has served its purpose, accumulated learnings and recent changes to the protocol—particularly around dispute game types—have exposed significant pain points that make it difficult to work with and maintain.

The Proofs team is most heavily impacted by OPCM v1's limitations. When protocol interface changes are required (especially those affecting dispute game handling), the current OPCM design makes these changes disproportionately difficult. The core issues stem from:

1. **Hardcoded assumptions about game types**: OPCM v1 assumes exactly 2 dispute game types (Cannon and Permissioned), which is expanding to 3 with recent Proofs team changes. Any fixed number creates friction because it requires extensive changes across the codebase rather than using a dynamic, extensible interface.

2. **Function proliferation**: OPCM v1 has 5 separate functions (`deploy()`, `upgrade()`, `addGameType()`, `updatePrestate()`, `interopMigrate()`), and interface changes to core contracts often impact all of them. Most teams only need to modify 2 functions, but the Proofs team's changes affect all 5, creating significant maintenance burden.

3. **Not "always release ready"**: OPCM v1 requires special one-time "upgrade functions" to be added to contracts during upgrades, which must then be removed in subsequent upgrades. This creates ongoing maintenance overhead and bifurcates the code paths between deployments and upgrades.

4. **High knowledge burden**: Developers need deep understanding of how OPCM integrates with op-deployer (the Go deployment tool), including the glue code that connects Solidity contracts to Go tooling. This prevents developers from focusing on their specific domain without understanding the entire system.

The cumulative effect of these issues, combined with our improved understanding of the right architectural direction, makes this the appropriate time to redesign OPCM. OPCM v2 aims to simplify the interface, eliminate hardcoded assumptions, unify deployment and upgrade paths, and reduce the knowledge burden on developers while maintaining full backwards compatibility with existing functionality.

**Key constraints:**
- Scale: Must handle all OP Stack chain deployments and upgrades
- Gas efficiency: Deployments must fit within Fusaka per-transaction gas limits
- Platform: Solidity smart contracts integrated with Go tooling (op-deployer)
- Backwards compatibility: Must support all existing OPCM v1 functionality and use cases
- Security: Requires audit given the critical nature of chain deployments and upgrades

## Customer Description

The primary customers for OPContractsManager v2 are engineering teams working on the OP Stack protocol:

- **Proofs team**: Most heavily impacted by current limitations; requires frequent interface changes for dispute game handling
- **Protocol team**: Core protocol development team that uses OPCM for upgrades
- **EVM Safety team**: Ensures safety properties of protocol upgrades
- **Base team**: Operates production OP Stack chains and performs upgrades
- **DeFi Wonderland**: External development partner working on protocol improvements

Secondary customers include any developers building on or operating OP Stack chains who need to perform deployments or upgrades.

### Customer Reviewers

Required reviewers:
- **Proofs team representative**: Primary stakeholder for interface flexibility requirements
- **Protocol team lead**: For overall architectural alignment
- **EVM Safety team representative**: For security and safety validation
- **Base team representative**: For operational considerations
- **DeFi Wonderland representative**: For external developer perspective

## Requirements and Constraints

### Functional Requirements

- Support deployment of new OP Stack chains with all necessary contracts
- Support upgrades of existing chains to new contract versions
- Handle dynamic/arbitrary number of dispute game types without code changes
- Add new game types to existing chains
- Update dispute game prestates
- Support interop migration functionality
- Maintain all functionality currently provided by OPCM v1's 5 functions

### Non-Functional Requirements

- **Backwards compatibility**: Support all existing OPCM v1 operations and use cases
- **Gas efficiency**: Single deployment transaction must fit within Fusaka per-tx gas limit
- **Security**: Must be auditable and maintain security properties of v1
- **Maintainability**: Unified code paths for deploy and upgrade operations
- **Developer experience**: Minimize knowledge burden through auto-generated glue code
- **Release readiness**: Always in deployable state without post-upgrade cleanup

### Constraints

- Must integrate with existing op-deployer Go tooling
- Cannot reduce any existing functionality from OPCM v1
- Interface changes are acceptable but must serve same purposes
- Minor breaking changes acceptable only if well-justified and documented
- Must handle existing deployed chains and future chains uniformly

## Proposed Solution

OPContractsManager v2 redesigns the architecture around three core principles: **interface simplification**, **unified initialization**, and **generated integration code**.

### Architecture Overview

OPCM v2 consolidates the 5 separate functions of v1 into 2 primary functions:
- `deploy(Config memory config)`: Deploys a new OP Stack chain
- `upgrade(Config memory config)`: Upgrades an existing chain

Both functions share the same initialization path for contracts, eliminating the special upgrade functions that previously required maintenance between releases.

### Key Architectural Changes

**1. Dynamic Dispute Game Interface**

Replace hardcoded assumptions about game types with dynamic arrays:
```solidity
struct GameConfig {
    GameType gameType;
    address implementation;
    bytes32 prestate;
}

struct Config {
    // ... other config
    GameConfig[] games;  // Support arbitrary number of game types
}
```

This eliminates the need for `addGameType()` and `updatePrestate()` as separate functions—these operations become implicit in `upgrade()` when the games array differs from the current state.

**2. Unified Initialization Pattern**

Both `deploy()` and `upgrade()` use the same contract initialization path:
- Clear the `initialized` slot (for upgrades)
- Call the standard `initialize()` function
- No special upgrade functions that need removal in subsequent releases

This ensures:
- Single code path for both operations
- Always release-ready (no post-upgrade cleanup)
- Easier testing and maintenance

**3. Auto-Generated Integration Code**

Generate the Go glue code between OPCM and op-deployer from Solidity interfaces:
- Solidity function signatures/ABIs are the source of truth
- Go code for constructing function calls is automatically generated
- Developers only need to add new CLI arguments to op-deployer when introducing net-new inputs
- Reduces knowledge burden—developers don't need to understand integration layer

### Requirements Mapping

- **Deploy new chains**: `deploy()` function
- **Upgrade existing chains**: `upgrade()` function  
- **Dynamic game types**: GameConfig array in Config struct
- **Add game types**: Implicit in `upgrade()` with modified games array
- **Update prestates**: Implicit in `upgrade()` with modified games array
- **Interop migration**: Handled within `upgrade()` logic based on Config
- **Gas efficiency**: Optimized unified code path fits within Fusaka limits
- **Backwards compatibility**: Both functions support all v1 operations
- **Release readiness**: Unified initialization eliminates upgrade function maintenance

## Alternatives Considered

### Alternative 1: Continue Incremental Improvements to OPCM v1

**Approach**: Continue the pattern of making incremental improvements to OPCM v1—adding features, refactoring specific pain points, and addressing issues as they arise without a larger architectural change.

**Advantages**:
- Lower risk, familiar codebase
- No migration complexity

**Disadvantages**:
- We've already done significant incremental work and are hitting diminishing returns
- Can't address core architectural issues (function proliferation, hardcoded assumptions) without larger changes
- Would continue accumulating technical debt

**Rejection rationale**: We've reached the limit of what incremental improvements can achieve. The current pain points—especially around dynamic game types and unified initialization—require larger architectural changes. Continuing with incremental fixes would mean accepting the existing limitations indefinitely. We know the right direction now, so it's time to make the larger change.

### Alternative 2: Complete OPCM Redesign (Different Architecture)

**Approach**: Rather than improving OPCM's existing design, start from scratch with a fundamentally different architecture.

**Advantages**:
- Could potentially optimize for currently unknown future requirements

**Disadvantages**:
- OPCM's general design is sound—the issues are specific pain points, not fundamental flaws
- Would throw away working, well-understood code
- Much higher risk and longer timeline
- Harder to maintain backwards compatibility

**Rejection rationale**: The core OPCM design is good. We don't want to replace it; we want to improve specific aspects. A complete redesign would be throwing out the baby with the bathwater. The v2 approach preserves what works while fixing what doesn't.

## Risks and Uncertainties

### Security Risk: Increased Flexibility Creates Increased Attack Surface

**Risk**: While OPCM v2 isn't a massive architectural change, it is still a change to critical infrastructure. The increased flexibility (particularly around dynamic game types and re-initialization) could introduce new security vulnerabilities if not implemented defensively.

**Mitigation**: 
- Comprehensive security audit with focus on the new flexible interfaces
- Defensive coding practices throughout implementation
- State diff review for the first OPCM v2 upgrade to verify correctness
- Multiple rounds of internal security review before audit

### Functionality Coverage Risk: Missing OPCM v1 Capabilities

**Risk**: We might inadvertently miss some existing functionality that OPCM v1 provides when transitioning to the new 2-function interface.

**Mitigation**:
- Ensure all existing OPCM v1 tests pass with v2 implementation
- Comprehensive test coverage mapping v1 functionality to v2
- Review session with all customer teams to validate their use cases are covered

### Partner Adoption Risk: Re-initialization Pattern Concerns

**Risk**: Chain operators and partners may be uncomfortable with the new upgrade scheme, particularly the re-initialization approach. There's an unknown around whether this will cause discomfort or concerns, even though we believe it's safer (consistent behavior across upgrades vs. changing upgrade functions).

**Context**: We posit that re-initialization is actually safer because it stays the same every upgrade instead of changing repeatedly—less change means safer code. However, partners need to understand and be comfortable with this pattern.

**Mitigation**:
- Clear documentation explaining why re-initialization is safer than one-time upgrade functions
- Add defensive guards in the code to prevent accidental misuse of re-initialization
- Early communication with Base team and other major chain operators
- Testnet validation with partner chains before production rollout

### Open Questions

1. **Alternative designs**: Are there alternative architectural approaches we may have missed that would better solve these problems?

2. **Shipping timeline**: When should we ship this? Is U18 (upgrade 18) the target, or should it be later?

3. **Edge case coverage**: Is there anything we're missing about upgrades where our new design won't work for specific use cases or edge cases?