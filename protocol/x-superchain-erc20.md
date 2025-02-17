# xSuperchainERC20 - Design Doc

# **Purpose**

This design document aims to provide a clear implementation path for token issuers who want to make their tokens interoperable across both the Superchain and other ecosystems, while maintaining compatibility with existing bridge infrastructure.

# **Summary**

xSuperchainERC20 is a token implementation that combines xERC20 ([ERC-7281](https://ethereum-magicians.org/t/erc-7281-sovereign-bridged-tokens/14979)) and SuperchainERC20 ([ERC-7802](https://ethereum-magicians.org/t/erc-7802-crosschain-token-interface/21508)) functionality. This allows tokens to be immediately usable with existing bridge infrastructure while being compatible with the Superchain interop cluster.

# **Problem Statement + Context**

Token issuers face two main challenges:

- Need for cross-ecosystem interoperability (Superchain + other ecosystems)
- Need for immediate token interoperability

# **Proposed Solutions**

## **A. Non-deployed or upgradable tokens**

For tokens that have not yet been deployed, or can be upgraded, we recommend implementing xSuperchainERC20 - a token that combines both ERC-7281 (xERC20) and ERC-7802 standards. This solution is ideal when:

- You want your token to work both within the Superchain (via ERC-7802)
- AND outside the Superchain using third-party bridges (via ERC-7281)

This approach provides maximum flexibility as:

1. Within the Superchain: Uses the SuperchainTokenBridge for fast (1-block) and secure transfers
2. Outside the Superchain: Allows configuration of any EVM-compatible third-party bridge through xERC20's mint/burn permissions

## **B. Deployed, non-upgradable tokens (Non-xERC20)**

For tokens that are already deployed and cannot be upgraded, we propose using a Lockbox mechanism:

1. Deploy a xSuperchainERC20 contract that will be the wrapped version of the original token
2. Deploy a Lockbox contract that:
   - Locks the original ERC20 tokens
   - Mints xSuperchainERC20 tokens at a 1:1 ratio
   - Can be deployed deterministically to have the same address across all chains

Key benefits:

- Maintains the original token's supply while enabling Superchain functionality
- The deterministic deployment ensures the wrapped token has the same address on all chains
- Enables usage of the SuperchainTokenBridge since the wrapped token's address is consistent across the network
- Users can always unwrap their tokens to return to the original ERC20

## **C. Deployed, non-upgradable xERC20**

For tokens that are already deployed as xERC20 and cannot be upgraded to implement ERC-7802. This is particularly important as many projects have already deployed xERC20 tokens and need a path to integrate with the Superchain while maintaining their existing bridge infrastructure.

### **SuperchainERC20Adapter**

This approach introduces an adapter contract that implements SuperchainERC20's crosschainBurn and crosschainMint functions, converting these calls into the corresponding xERC20 mint/burn operations.

Raw Example:

```solidity
function crosschainMint(address _to, uint256 _amount) external {
    if (msg.sender != Predeploys.SUPERCHAIN_TOKEN_BRIDGE) revert Unauthorized();

		// Instead of internally calling the _mint() function it calls the xERC20 mint().
    IXERC20(XERC20_ADDRESS).mint(_to, _amount);

    emit CrosschainMint(_to, _amount, msg.sender);
}
```

Key benefits:

- The most simple/straightforward solution, token no need to have the same address.
- Uses SuperchainTokenBridge

Cons:

- Token owner has to deploy one adapter for each chain using the same address.

Setup Diagram:

```mermaid
sequenceDiagram
  actor A0 as Owner
  box L2 Chain
	  participant P1 as Factory
    participant P2 as Adapter
    participant P3 as XERC20
  end

	A0 ->> P1: deployAdapter()
	P1 ->> P2: constructor(XERC20)
  A0 ->> P3: setLimit(Adapter...)

```

Usage Diagrams:

```mermaid
sequenceDiagram

  actor A1 as Alice
  box L2 Chain A
    participant P1 as SuperchainTokenBridge
    participant P2 as Adapter
    participant P3 as XERC20 A
  end

  A1 ->> P1: sendERC20(Adapter...)
  P1 ->> P2: crosschainBurn()
  P2 ->> P3: burn()
  P1 -->> P1: SendERC20()

```

```mermaid
sequenceDiagram

  box L2 Chain B
    participant P5 as XERC20 B
    participant P6 as Adapter'
    participant P7 as SuperchainTokenBridge
  end
  actor A2 as Relayer

  A2 ->> P7: relayERC20(Adapter'...)
  P7 ->> P6: crosschainMint()
  P6 ->> P5: mint()
  P7 -->> P7: RelayERC20()
```

## xSuperchainERC20Factory

The Factory contract serves as a central deployment mechanism for all the components in our system. It provides methods to:

1. Deploy new xSuperchainERC20 tokens
2. Deploy Lockboxes for existing tokens
3. Deploy Adapters for existing xERC20 tokens

**Key Features:**

- Deterministic deployments using CREATE3
- Registry of deployed contracts
- Permission management for deployments
- Single entry point for all deployments

---

## Alternatives Considered for Scenario C

_Worth mentioning that the Lockbox solution described in section B is also compatible with this scenario, providing an alternative option if the token issuer prefers to wrap their xERC20 into a new xSuperchainERC20 token_

### **xSuperchainERC20Converter**

It is a xSuperchainERC20 contract that can mint/burn xERC20 tokens. Similar to the Lockbox mechanism, but instead of locking tokens, it performs a direct conversion.

_It would be good to add a `pause` modifier on the convert functions to force the liquidity to be moved to one direction._

Pre-requisite:

- Give mint/burn permissions to the converter.
- Token owner has to deploy the contract and set it up.

Key benefits:

- Same approach that for non xERC20 migration, reduced complexity on development process. Allows native usage of xSuperchainERC20/7802

Cons:

- Fragmented liquidity if the xERC20 coexists with the xSuperchainERC20
- User needs two transactions to cross-transfer

Setup Diagram:

```mermaid
sequenceDiagram
  actor A0 as Owner
  box L2 Chain A
    participant P2 as SuperchainERC20Converter
    participant P3 as XERC20 A
  end
  box L2 Chain B
    participant P5 as XERC20 B
    participant P6 as SuperchainERC20Converter'
  end

  A0 ->> P3: setLimit(Converter...)
  A0 ->> P5: setLimit(Converter'...)
  A0 ->> P2: setToken(XERC20 A)
  A0 ->> P6: setToken(XERC20 B)

```

Usage Diagrams:

```mermaid
sequenceDiagram
  actor A1 as Alice
  box L2 Chain A
    participant P2 as SuperchainERC20Converter
    participant P3 as XERC20 A
    participant P1 as SuperchainTokenBridge
  end

  A1 ->> P2: convert()
  P2 ->> P3: burn()
  P2 ->> P2: mint()
  A1 ->> P1: sendERC20()
  P1 ->> P2: crosschainBurn()

```

```mermaid
sequenceDiagram
  box L2 Chain B
    participant P5 as XERC20 B
    participant P6 as SuperchainERC20Converter'
    participant P7 as SuperchainTokenBridge
  end
  actor A2 as Relayer

  A2 ->> P7: relayERC20()
  P7 ->> P6: crosschainMint()
  A2 ->> P6: undo()
  P6 ->> P6: burn()
  P6 ->> P5: mint()
```

### **SuperchainXERC20Bridge**

This approach introduces a new predeploy contract that acts as a bridge between xERC20 tokens across the Superchain. The implementation requires:

Pre-requisite:

- xERC20 addresses MUST be the same in both chains
- Configuring the xERC20 token to grant mint/burn permissions to this predeploy

Key benefits:

- Single contract for every token
- Utilizing the L2ToL2CrossDomainMessenger for secure cross-chain communication

Cons:

- Discourages the adoption, prioritizing xERC20 over SuperchainERC20
- Build and maintain a new predeploy.

Setup Diagram:

```mermaid
sequenceDiagram
    actor A0 as Owner
  box L2 Chain A
    participant P2 as XERC20
  end

  box L2 Chain B
      participant P4 as XERC20'
  end
    A0 ->> P2: setLimits(SuperchainXERC20Bridge...)
    A0 ->> P4: setLimits(SuperchainXERC20Bridge'...)
```

Usage Diagrams:

```mermaid
sequenceDiagram
  actor A1 as Alice
  box L2 Chain A
    participant P1 as SuperchainXERC20Bridge
    participant P2 as XERC20
    participant P3 as L2ToL2CDM
  end

  A1 ->> P1: sendXERC20()
  P1 ->> P2: burn()
  P1 ->> P3: sendMessage()
  P3 -->> P3: SentMessage()
```

```mermaid
sequenceDiagram
  box L2 Chain B
      participant P4 as XERC20'
      participant P5 as SuperchainXERC20Bridge'
      participant P6 as L2ToL2CDM'
  end
  actor A2 as Bob

  A2 ->> P6: relayMessage()
  P6 ->> P5: relayXERC20()
  P5 ->> P4: mint()

  P6 -->> P6: RelayedMessage()
```
