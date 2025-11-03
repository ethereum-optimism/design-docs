# Custom Gas Token: Design Doc

| Author | *AgusDuha, Joxes, Skeletor* |
| --- | --- |
| Created at | *2025-07-24* |
| Initial Reviewers | *Mark Tyneway, TBD* |
| Need Approval From | *TBD* |
| Status | *In Review* |

## Purpose

Enable OP Stack chains to use an asset other than ETH as their native fee currency by introducing a minimal, flexible, and standardizable Custom Gas Token (CGT) architecture. 

## Summary

The proposed Custom Gas Token upgrade lets any OP Stack chain introduce its native asset as the gas currency with almost no core-code intrusion: a single `isCustomGasToken()` flag turns off ETH transfer flows in all bridging methods, while two new pre-deploys, `NativeAssetLiquidity`, a contract with pre-minted assets, and `LiquidityController`, an owner-governed mint/burn router that manages the supply, are designed to couple with any release mechanism, from ERC-20 converters to third-party bridging models or emission schedules. No bridge or token is enshrined in the protocol; any bridge/adapter lives entirely at the app layer and is authorized only as a minter on `LiquidityController`, coupled after genesis. Overall, the design keeps the OP Stack lean, token-agnostic, and future-proof while unlocking custom economics and flexibility to make the native asset expressive in its manner.

## Problem Statement + Context

The [prior CGT design](https://github.com/ethereum-optimism/specs/blob/5887eb2f74821b4125c2f47523d6d54379fd46d8/specs/experimental/custom-gas-token.md) anchored the gas token to an L1 ERC‑20, hard‑coding bridge paths, and cluttering core contracts with token metadata. These restrictions make it hard to handle the following scenarios properly:

- Integrates with a token that doesn’t live on L1.
- Launch chains whose native assets do not yet exist or have an ERC20 representation elsewhere.
- Experiment with custom supply management or economics.
- Implement any novel or non-conventional bridging mechanisms.

Additionally, this design is interested in feasibly becoming a standard version for future compatibility, such as interoperability. The new design needs to keep the OP Stack lean and become future-proof.

## Proposed Solution

### Design Principles

The presented design has been thought out based on the following principles:

- Stick with a minimal set of changes.
- Remain token-agnostic and completely flexible for any use cases.
- Preserve OP Stack [foundational properties](https://specs.optimism.io/background.html#foundations), which also make [Stage 2](https://medium.com/l2beat/introducing-stages-a-framework-to-evaluate-rollups-maturity-d290bb22befe) requirements achievable.

### Architecture

**`isCustomGasToken()` in L1 and L2**

The `isCustomGasToken()` view function becomes the unique one that brings awareness to the system about whether or not the CGT mode is being used. It is a boolean flag that other functions in other contracts use to check if they can receive ETH values or not.

In L1 placed in `SystemConfig` to be read by `OptimismPortal`, it will block `depositTransaction` calls that contain `msg.value`. The same will occur in their pair functions `sendMessage` in `L1CrossDomainMessenger` and ETH bridging in `L1StandardBridge`.

In L2 placed in `L1Block` to be read by `L2ToL1MessagePasser`, it will block `initiateWithdrawal` calls, that contain `msg.value` in `L2ToL1MessagePasser`. The same will occur in `sendMessage` in `L2CrossDomainMessenger` and ETH bridging methods in `L2StandardBridge` and `FeeVaults`.

As a consequence, native asset mints and burns are decoupled from deposits (system transactions) and withdrawals in normal operations, and those are moved into the application layer with the newly introduced predeploys.

**`NativeAssetLiquidity` and `LiquidityController` Predeploys**

A arbitrary amount of native assets is pre-minted and stored in `NativeAssetLiquidity`, which is managed through the `LiquidityController`, which has the rights to withdraw and deposit to this contract.

A code example of both contracts would look like this:

```solidity
contract NativeAssetLiquidity {
	
	// Reduce native supply
	function deposit() external payable {
	  if (msg.sender != Predeploys.LIQUIDITY_CONTROLLER) revert Unauthorized();
	}
	
	// Increase native supply
	function withdraw(uint256 _amount) external {
	  if (msg.sender != Predeploys.LIQUIDITY_CONTROLLER) revert Unauthorized();
	  new SafeSend{ value: _amount }(payable(msg.sender));
	}
}
```

```solidity
contract LiquidityController {

	// Authorized addresses to manage the liquidity of NativeAssetLiquidity
	mapping(address => bool) public minters;

    // Metadata is set in contract initialization, for backward compatibility with L1Block and WETH
    string public gasPayingTokenName;

    string public gasPayingTokenSymbol;

    }

	// Authorizes an address from performing liquidity control operations
	function authorizeMinter(address _minter) external {
		if (msg.sender != IProxyAdmin(Predeploys.PROXY_ADMIN).owner()) revert Unauthorized();
		minters[_minter] = true;
		(...)
	}

	// Deauthorizes an address from performing liquidity control operations
	function deauthorizeMinter(address _minter) external {
		if (msg.sender != IProxyAdmin(Predeploys.PROXY_ADMIN).owner()) revert Unauthorized();
		if (!minters[_minter]) revert NotMinter(_minter);

		delete minters[_minter];
		(...)
	}

	// Authorized minter can unlock native asset
	function mint(address _to, uint256 _amount) external {
		if (!minters[msg.sender]) revert Unauthorized();
		INativeAssetLiquidity(Predeploys.NATIVE_ASSET_LIQUIDITY).withdraw(_amount);
		(...)
	}
	
	// Authorized minter can lock native asset 
	function burn() external payable {
		if (!minters[msg.sender]) revert Unauthorized();
        INativeAssetLiquidity(Predeploys.NATIVE_ASSET_LIQUIDITY).deposit{ value: msg.value }();
		(...)
	}
	
}
```

Chain Governors will be responsible for granting minter rights in the `LiquidityController`, which has full flexibility to integrate with any new or existing ERC20 wrappers, bridges, or novel tokenomics mechanisms. As an **_early example_** that demonstrates how it can be coupled with an external contract, a Chain Governor might grant an _`ERC20Converter`_ the minter role, enabling 1:1 exchanges between the native asset and a desired ERC20 representation:

```solidity
contract ERC20Converter {

  // Lock or burn the appointed ERC20 and get msg.value
	function getNativeAsset(address _to, uint256 amount) external {
		(...)
	}

	// Send msg.value and unlock or mint the ERC20
	function getERC20(address _to) external payable {
		(...)
	}
}
```

The flow between the predeploys and the external contract as a minter would look like this:

```mermaid
sequenceDiagram
    participant Address A as Privileged Actor
    participant LiquidityController
    participant Liquidity as NativeAssetLiquidity
    participant ERC20Converter
    participant Address B as Regular User

    %% --- Flow A: direct mint of native asset ---
    Note over Address A,recipient: [Flow A] Mint native asset
    Address A->>LiquidityController: mint(amount, recipient)
    LiquidityController->>LiquidityController: isAuthorizedMinter()
    LiquidityController->>Liquidity: withdraw(amount, recipient)
    Liquidity-->>recipient: native asset

    %% --- Flow B: lock ERC20 and mint native asset ---
    Note over LiquidityController,recipient: [Flow B] Convert ERC20 → native asset
    Address B->>ERC20Converter: getNativeAsset(ERC20, amount, recipient)
    ERC20Converter->>ERC20Converter: lock / burn ERC20
    ERC20Converter->>LiquidityController: mint(amount, recipient)
    LiquidityController->>Liquidity: withdraw(amount, recipient)
    Liquidity-->>recipient: native asset

    %% --- Flow C: burn native asset and unlock ERC20 ---
    Note over LiquidityController,recipient: [Flow C] Convert native asset → ERC20
    Address B->>ERC20Converter: getERC20{msg.value = amount}(recipient)
    ERC20Converter->>LiquidityController: burn() (payable)
    LiquidityController->>Liquidity: deposit() (payable)
    LiquidityController-->>recipient: ERC20

```

### Native Asset Value

Since native assets are pre-minted from genesis, the chain governor can decide how to assign value or give it meaning. This is achieved by understanding where the supply originates and how it is reflected in `NativeAssetLiquidity` and `LiquidityController` management. There are at least three main cases:

- The simplest is to make an existing ERC20 the native asset, bridged by any mean.
- Introduce an ERC20 representation in L2 that contains utility functions (governance, DeFi); meanwhile, their native asset version serves as the medium to pay for gas.
- Do not depend on an existing asset, emulating how some L1 currently works.

The design allows for the inclusion of any desired features, such as rate limits, emission schedules, and a max. cap the supply, coupled with any bridge mechanism, etc. The architecture also doesn’t restrict the use of any token framework (e.g., OFT, xERC20, SuperchainERC20, etc.) to be coupled with the native asset eventually.

Existing CGT chains using the old design can perform a hard fork to set such contracts and seed the native asset liquidity.

### Wrapped Native Asset

The `WETH` (now the wrapped version of the native asset) predeploy would stay the same as its old version, to preserve backward compatibility with existing chains.

To align with the new design, the metadata, specifically the `name()` and `symbol()` functions, will now be sourced from the `LiquidityController` instead of the `SystemConfig` contract used in the previous design. These functions will return the name and symbol of the wrapped native asset, prefixed with "W" and "Wrapped".

### Gas-related Components

The CGT sets `Types.WithdrawalNetwork.L2` as chain type in each `FeeVaults` contracts on the constructor, without requiring contract changes. Funds are going to be sent to a desired L2 address.

To accurately charge the fees for execution and data availability, chain governors may now adjust a `minBaseFee` and `operatorFee`. In the future, other parameters or mechanisms might be included to properly account for the cost for each activity, such as adding a price oracle that converts data cost expressed in ETH into the ASSET.

### Chain Deployment

As part of the feature flag system, during the deployment in genesis, the CGT version comes up with the `isCustomGasToken()` flag activated, adding the new `NativeAssetLiquidity` and `LiquidityController` predeploy contracts, and seeding an amount of liquidity into the  `NativeAssetLiquidity`.

The deployment flow would look like this:

```mermaid
sequenceDiagram
    participant Genesis   as Deploy Genesis
    participant OP Stack  as OP Stack
    participant Liquidity as NativeAssetLiquidity
    participant Controller as LiquidityController
    participant Governor  as ProxyAdmin Owner
    participant Deployer  as Address1, Address2,...

    rect rgba(128, 128, 128, 0.1)
		note over Liquidity: Genesis Phase
    Genesis->>OP Stack: Genesis core actions
    Genesis->>Liquidity: create & pre-fund native supply
    Genesis->>Controller: create & set token metadata
    end
    Governor->>Controller: authorize (this address or any other) and mint
    Controller->>Liquidity: withdraw
    Liquidity-->>Governor: native asset
    Governor->>Deployer: transfer native asset
    Deployer->>Deployer: deploy external contracts (a bridge, a converter...)

```

These predeploys are specific to CGT-enabled chains, meaning OP Stack chains not running in CGT will not include them. Rollup config files in `op-node` must be aware of whether CGT mode is being used.

By default, the `ProxyAdmin` owner can release the supply at the beginning and distribute the minimal supply needed to ensure the CGT-specific contracts related to their use case are deployed. The chain governor may also opt to withdraw any excess native supply and burn it to reduce the total supply recorded in the chain’s state, for example by using the `burn()` function held in `L2ToL1MessagePasser`.

Chain Governors must define the `SYMBOL` and `NAME` of the `WETH` in `LiquidityController`.

### ERC20 Coupling and External Bridging

As the native asset isn’t enshrined to any token or bridge mechanism from the start and is instead managed through external contracts, its representation in L2 and LX can be defined at the Chain Governor’s discretion.

For example, if the ERC20 already lives in L1, a hypothetical _`CGTBridge` set of contracts_ might allow depositing the ERC20 and releasing native assets to the user. This design makes any other kind of bridging (from any blockchain and security model) possible.

```mermaid
graph TD

    %% -------- External actor --------
    User1(User)
    User2(User)

    %% -------- Ethereum L1 (horizontal flow) --------
    subgraph "Ethereum L1"
        direction LR
        L1Bridge(_L1CGTBridge_)
        L1Messenger(L1CrossDomainMessenger)
        OptimismPortal(OptimismPortal)
    end

    %% -------- CGT Chain L2 (horizontal flow) --------
    subgraph "CGT Chain L2"
        direction LR
        L2Messenger(L2CrossDomainMessenger)
        L2Bridge(_L2CGTBridge_)
        LiquidityController(LiquidityController)
        NativeAssetLiquidity(NativeAssetLiquidity)
    end

    %% -------- Interactions --------
    User1 -->|<b>1.</b> depositCGT| L1Bridge
    L1Bridge -->|<b>2.</b> sendMessage| L1Messenger
    L1Messenger -->|<b>3.</b> depositTransaction| OptimismPortal
    OptimismPortal -.->|<b>4.</b> relayMessage| L2Messenger
    L2Messenger -->|<b>5.</b> mintCGT| L2Bridge
    L2Bridge -->|<b>6.</b> mint| LiquidityController
    LiquidityController -->|<b>7.</b> withdraw| NativeAssetLiquidity
    NativeAssetLiquidity -->|<b>8.</b> send| LiquidityController
    LiquidityController -->|<b>9.</b> deliver native asset| User2

```

### Resource Usage

No major changes are expected. However, the resource pricing needs to be adjusted to cover the expected costs on execution and data availability properly.

### Single Point of Failure and Multi Client Considerations

Since the `NativeAssetLiquidity` can contain a large amount of pre-minted native assets, managing the supply properly becomes fundamental. Failures would affect the value of the native asset, ruining their economics and severely affecting the user-experience.

Regarding existing clients, no major impact is expected from this design. The respective rollup config files must be updated to reflect whether the CGT mode is activated or not.

## Failure Mode Analysis

TBD.

## Impact on Developer Experience

The developers will present several changes with respect to an OP Stack chain that uses ETH as native asset:

- Since native asset management is decoupled from core components, obtaining native assets might vary chain-by-chain. That means that bridging, conversions, and features utilized would be different. It’s suggested to create some library or templates that cover basic uses cases (e.g., L2-native, bridging from L1) to standardize most implementations.
    - Chain Governors are encouraged to audit their implementations.
    - Developers would need to look at the chain’s docs to understand how to obtain native assets.
- `OptimismPortal` and `L2ToL1MessagePasser` don’t contain the API anymore to bridge native assets.
- Native ETH bridging is disabled, now enabled through L1-WETH, minting an `OptimismMintableERC20` representation.
- Supersim needs to be aware when CGT is being used.

## Alternatives Considered

**Maintain the old design.**

The old design required a token to exist in L1 previously. This also needed to introduce a `depositERC20Transaction` as well as the `GasPayingToken` library that contains the relevant gas token functions to derive in L2. This design restricts native asset management by only minting new assets through a `depositTransaction`, digested by the `op-node` through a system transaction (type 126).

**Special mint deposit function in L1.**

It is possible to add a `mint` function in `OptimismPortal` that grants permissions to an `authorizedMinter`, similar to the `LiquidityController`, as a minimalist design of the old one, since it wouldn't require the `GasPayingToken` library and ERC20-related code. However, this path is not considered a winning solution since it provides async minting, which is not suitable for all use cases, and doesn't have a clean correspondence with a withdrawal flow, while preserving the same risk tree as in `LiquidityController`. However, this mechanism can be thought of as a way to avoid hard forks if more asset is required to be minted for some reason as a future feature.

**Modify derivation rules to allow custom minting rules.**

Since tx type 126 only allows native assets mints through a `depositTransaction`, customizing this feature to be flexible enough would require major changes on derivation rules, `op-node`, and execution clients to permit new supply creation. Adding new tx types or new precompiles would cause issues such as divergence in current software, unnecessary breaking of existing tooling, and additional maintenance.

**Modify the execution environment and client software.**

EVM-equivalence means the native asset doesn’t have any custom feature other than the described in the EVM specs. It could be possible for the native asset to be modified to allow features such as blacklisting, whitelisting, staking, and voting at the cost of losing EVM equivalence. This would likely affect the current proof systems and fundamental APIs such as `depositTransaction`. This alternative would case similar issues on software maintenance and existing toolings, as stated above.

**Merge `NativeAssetLiquidity` and `LiquidityController`.**

As an alternative to the proposed architecture, both predeploys might be merged into a single one. 

## Risks & Uncertainties

- Any minters granted to interact with the `LiquidityController` require proper audits to ensure native asset logic can’t be broken in production, as the recovery would be costly.
    - There are two potential paths to minimize the impact: adding rate limits, and burn the unneeded supply after chain's deployment.
- There is an open discussion on how to support tokens that have other than 18 decimals. This is a concern for the minter that is placed on top of the `LiquidityController`.
    - One possible solution might be adding `decimals()` in the `LiquidityController`, or fully handling it through the minters.
- Existing OP Stack chains using old designs would need to upgrade to the new version, the solution of which is actively being architected.
- The design is proposed to facilitate its standardization, but the discussion regarding enabling interoperability is still open.
    - For example, `ETHLiquidity` and `NativeAssetLiquidity` might share the same address, and the ERC20 version of the native asset might be created with a deterministic method to enable interoperability through a `SuperchainERC20` version.
    - A new `SuperchainWETH` version might be introduced to enable interoperability for the ETH transfer.
- `L1FeeVault` might not receive the amount in value to pay for data availability, since the mismatch is corrected via `minBaseFee` and `operatorFees`. The introduction of parameters or oracles is actively being discussed.
- The CGT predeploys serve as a set of audited contracts that are ready to use, but nothing prevents the chain governor from withdrawing arbitrary amounts of liquidity and using their own contracts without relying on the predeploys at all.
- Native asset custom features are in early research phase, but the design is minimal enough not to block a future upgrade of this kind.

### NativeAssetLiquidity Drained
A bug or a compromise on `NativeAssetLiquidity` or the `LiquidityController` could lead to a massive asset loss either in the canonical bridge or third party bridges. Given the low complexity of `LiquidityController`, the most likely vector would be an access control bug or attack.

This is already mitigated by the airgap in each bridge, and by the existing measures to avoid bugs and L2 predeploy compromises. No new mitigations are required for this risk.

### CGTs unduly minted or burned
Errors or compromises affecting chain operators could cause CGTs to be minted or burned not following the protocol. The most likely would be:
- L1 or L2 misconfigurations of `isCustomGAsToken` or `LiquidityController`.
- AccessControl, Replayability, ERC20 or CGT bridge smart contract bugs.
- `NativeAssetLiquidity` balance overflows
- Misconfiguration of `L1ToL2MessagePasser`

These risks would be mostly borne by chain operators, and the developer documentation should reflect these concerns so that they are not overlooked. Additionally, we will need automated checks to verify that the `isCustomGAsToken` always matches in the L1 and L2 parts of a chain configuration.

### Very Inaccurate Fees
L1 misconfigurations could lead to very inaccurate fees being charged. This will need to be highlighted in the developer documentation for chain operators to be aware.

### Chain operators see wrong amounts of wrapped asset
The current version of CGT only accepts 18-decimal CGTs, or the wrapper contract will malfunction. This will need to be highlighted in the documentation.
## Appendix

### Appendix A: Old vs. New CGT design

Since this design aims to be a better, minimal —yet-flexible— version of the [previous CGT design](https://github.com/ethereum-optimism/specs/blob/5887eb2f74821b4125c2f47523d6d54379fd46d8/specs/experimental/custom-gas-token.md), it is worth comparing the two to understand the trade-offs involved in each. Below is a quick comparative table that contrasts both approaches:

| Item | Old Design | New Design |
| --- | --- | --- |
| Approach | L1 token is used as gas asset. | The native asset exists in its own and may be convertible into an ERC20. |
| Native asset is bridgeable? | Yes, through `depositERC20Transaction` and native withdrawals. | Yes, but not enshrined in core contracts. The chain governor must implement custom logic to enable it, e.g., coupling bridging with the `LiquidityController`. |
| Token Implementation Flexibility | Restricted to 18 decimals and transfer properties. | There are potentially no limitations, as long as it is coupled adequately with `NativeAssetLiquidity`. |
| WETH predeploy | Reserved for the wrapped version of the custom gas token, taking metadata from `L1Block` (`SystemConfig`). | Reserved for the wrapped version of the custom gas token. Metadata is taken from `L1Block` (`LiquidityController`). |
| Native Asset Supply | Held in `OptimismPortal`, originating from the original L1 token contract. | Minted at genesis via `NativeAssetLiquidity`. Manageable under any rules, through the controller, which may depend on an existing ERC20 or any kind of release mechanism. |
| LX-LY/Deposit and Withdrawal Experience | Simple, via `OptimismPortal` and `L2ToL1MessagePasser`, respectively, analogous to ETH in standard chains. | Depending on the specific implementation, it could rely on a custom bridge built on top of OP Stack, a third-party bridge, or no bridge at all. |
| Token Upgradability / Chain Adaptability | No changes are recommended after chain deployment. | Allows upgrades and adaptability, since the native asset is not tied to an L1 token or specific bridge mechanism. |
| Ease of Deployment | Choose a L1 Token and deploy the chain as usual. | Deploy the chain as usual, and chain governors must deploy the relevant contracts to manage the native asset. |

### Appendix B: Flow Examples

**Native ETH L1 Bridging in CGT Mode**

When CGT mode is activated, all bridging methods related to ETH on L1 are disabled, since it is not used as the native asset.

```mermaid
 sequenceDiagram
    actor U as User
    participant SB as L1StandardBridge
    participant XDM as L1CrossDomainMessenger
    participant OP as OptimismPortal
	participant SC as SystemConfig
    
    U->>SB: depositETH / bridgeETH / receive(...)
    SB->>XDM: sendMessage(...)
    XDM->>OP: depositTransaction(...)
    OP->>SC: isCustomGasToken()
	SC-->>OP: returns true
    OP->>OP: revert

    U->>XDM: sendMessage(...), value > 0
    XDM->>OP: depositTransaction(...)
    OP->>SC: isCustomGasToken()
	SC-->>OP: returns true
    OP->>OP: revert

    U->>OP: depositTransaction / receive(...), value > 0
    OP->>SC: isCustomGasToken()
	SC-->>OP: returns true
    OP->>OP: revert

```

**Native Asset Withdrawals in CGT Mode**

Similarly, when CGT mode is used, native asset withdrawals are disabled, since it corresponds to ETH when used as the native asset, in order to maintain consistency with L1.

```mermaid
sequenceDiagram
    actor U as User
    participant SB as L2StandardBridge
    participant XDM as L2CrossDomainMessenger
    participant OP as L2ToL1MessagePasser
    participant LB as L1Block
    
    U->>SB: withdraw / bridgeETH / receive(...)
    SB->>XDM: sendMessage(...)
    XDM->>OP: initiateWithdrawal(...)
    OP->>LB: isCustomGasToken()
	LB-->>OP: returns true
    OP->>OP: revert

    U->>XDM: sendMessage(...), value > 0
    XDM->>OP: initiateWithdrawal(...)
    OP->>LB: isCustomGasToken()
	LB-->>OP: returns true
    OP->>OP: revert

    U->>OP: initiateWithdrawal(...), value > 0
    OP->>LB: isCustomGasToken()
	LB-->>OP: returns true
    OP->>OP: revert

```

 

**Native Asset Coupled to an L2 Token**

When the liquidity of the native asset is coupled with an ERC20 on L2, exchanges between them occur.

```mermaid
sequenceDiagram
    actor U as User
    participant ERC20
    participant ERC20C as ERC20Converter
    participant LC as LiquidityController
    participant NAL as NativeAssetLiquidity
    
    Note over U,NAL: [Flow A] Convert ERC20 → native asset
    U->>ERC20: approve(...)
    U->>ERC20C: getNativeAsset(...)
    ERC20C->>ERC20: transferFrom(...)
    ERC20C->>LC: mint(...)
    LC->>NAL: withdraw(...)
    NAL-->>U: native asset

    Note over U,NAL: [Flow B] Convert native asset → ERC20
    U->>ERC20C: getERC20(...)
    ERC20C->>LC: burn(...) (payable)
    LC->>NAL: deposit(...) (payable)
    LC->>ERC20: mint(...)
    ERC20-->>U: ERC20

```

**Native Asset Coupled to an L1 Token**

When the liquidity of the native asset is coupled with an ERC20 on L1, there should be a bridging mechanism between them, and its nature or security model is not constrained by this design.

```mermaid
sequenceDiagram
    actor U1 as User in L1
    participant L1T as ERC20
    participant BR as L1Bridge
    participant L2B as L2Bridge
    participant LC as LiquidityController
    participant NAL as NativeAssetLiquidity
    actor U2 as User in L2

    Note over U1,U2: [Flow A] Convert L1 ERC20 -> native asset
    U1->>L1T: approve(...)
    U1->>BR: deposit(...)
    BR->>L1T: transferFrom(...)
    BR-->>BR: lock or burn
    BR->>L2B: verify and relay deposit
    L2B->>LC: mint(...)
    LC->>NAL: withdraw(...)
    NAL-->>U2: native asset

    Note over U1,U2: [Flow B] Convert native asset -> L1 ERC20
    U2->>L2B: withdraw(...)
    L2B->>LC: burn(...) payable
    LC->>NAL: deposit(...) payable
    L2B->>BR: verify and relay deposit
    BR-->>BR: unlock or mint
    BR->>L1T: transfer
    L1T-->>U1: ERC20 on L1

```

**Mixed and Other Cases**

An L2 token coupled to native asset liquidity may also be bridgeable, or the native asset may have another token representation, as already occurs with the `WETH` predeploy contract. The desired supply management will determine the supply relationship between an ERC20 and the native asset. It is up to the governor to decide where the supply originates, under which mechanism it is created, and how the native asset is released.

Also, it is entirely possible to not depend on an ERC20 and be released by any supply liberation mechanism, such as mining based on other actions, conditions, etc.
