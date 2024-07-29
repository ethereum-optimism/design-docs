## Summary

This document presents the requirements for a factory that deploys `SuperchainERC20` tokens for tokens native to L1.

## Overview

The `SuperchainERC20Factory` will be a proxied predeploy. It will deploy proxied `SuperchainERC20` tokens using the Beacon pattern. This structure allows upgrading all `SuperchainERC20` implementations at once if needed.

Even though this document is specific for L1 native tokens, it should serve as a basis for other cases.

## Problem Statement + Context

The `SuperchainERC20` [standard](https://github.com/ethereum-optimism/specs/blob/aee0b2b2b45447daedef4b09bedc1fe7794d645d/specs/interop/token-bridging.md) will make ERC20s interoperable across the Superchain. However, the standard does not solve one of the main challenges in the future of Superchain: dealing with synchronized deployments and upgrades.

Moreover, the `SuperchainERC20` corresponding to a legacy token will require a liquidity migration conversion method, as presented in the corresponding [spec document](https://github.com/ethereum-optimism/specs/pull/294). The system requires a registry to track the allowed representations for the conversion to be safe.

Both these problems can be solved by introducing factories that comply with the following requirements:

- **Same address deployment:** the `SuperchainERC20` standard requires the same address across chains.
  - Notice this is not strictly necessary: allowing different addresses of `SuperchainERC20`s to communicate using a registry is possible. However, maintaining a registry would be a demanding task.
- **Implementation upgradability:** it should be easy to upgrade `SuperchainERC20` implementations across chains.
- **Factory upgradability:** The factory should be upgradable to follow OP Factories structure.
- **Deployment history:** The factory should keep a history of deployed tokens, acting as a registry.

> 💡
> The presented solution is focused on tokens that are native to L1. For L2 native tokens, the factory implementation should be modified.

## Proposed Solution

In what follows, we will list each requirement and detail how the factory should look to accomplish them.

### Same address: Predeploys + `CREATE3`

**Predeploys**

Factories will live in each L2 as a predeploy on the same address, which is crucial to achieving the same address for `SuperchainERC20`s across chains.

If the same address for the `SuperchainERC20` on L1 is also needed, then the address at which the predeploy is initialized should match the factory's L1 address.

**Creation method**

The `SuperchainERC20` addresses should be independent of the implementations. `CREATE` was discarded due to nonce dependability, and `CREATE2` was discarded because of its dependence on the creation code. For these reasons, the factory should use `CREATE3`.

The salt will be composed of the L1 address and the token metadata. This implies that the same L1 token can have multiple SuperchainERC20 representation, as long as the metadata also changes.

### Implementation Upgradability: Beacon Proxies

**BeaconProxies**

The factory will deploy `SuperchainERC20`s as BeaconProxies, as this is the easiest way to upgrade multiple proxies simultaneously. Each BeaconProxy will delegate-call to the implementation address provided by the Beacon Contract.

**Beacon Contract**

The Beacon Contract holds the implementation address for the BeaconProxies. It will be a predeploy (consistent with the current architecture of the OP stack).

**Implementations**

Each type of ERC20 (vanilla, vote token, rebasing, xERC20, etc) will require their unique implementation and, therefore, factory. Optimism should deploy and support the most common implementations. Anyone can deploy their custom implementations and factories but without predeploys.

In an implementation update, the upgraded chains must perform a hardfork to change the constant value in the Beacon Contract representing the implementation address.

### Factory Upgradability

The factory will be a simple Proxy contract pointing to the current implementation. In case of an upgrade to the factory, the upgrading chains will need to deploy the new implementation and perform a hardfork to update the implementation address.

### Deployment history

The factory will store a `deployments` mapping that the `L2StandardBridge` will check for access control when converting between the SuperchainERC20 and their corresponding legacy representations (see the [liquidity migration specs](https://github.com/ethereum-optimism/specs/pull/294) for more details).

```solidity
mapping(address => address) public deployments;
```

that maps each deployed SuperchainERC20 address to the corresponding L1 address (remote address).

## Implementation

**Proxy Factory**

Will follow the [`Proxy.sol` implementation](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/packages/contracts-bedrock/src/universal/Proxy.sol) and `delegatecall()` to the factory implementation address.

**Factory Implementation**

```solidity
contract SuperchainERC20Factory is Semver {
  mapping(address => address) public deployments;

  event SuperchainERC20Deployed(address indexed superchainERC20, address indexed remoteToken, string name, string symbol, uint8 decimals);

  constructor() Semver(1, 0, 0) {}

  function deploy(address _remoteToken, string memory _name, string memory _symbol, uint8 _decimals) external returns (address _superchainERC20) {
    bytes memory _creationCode = abi.encodePacked(
        type(BeaconProxy).creationCode,
        abi.encode(Predeploy.SuperchainERC20Beacon, abi.encodeCall(SUPERCHAIN_IMPL.initialize.selector, (_remoteToken, _name, _symbol, _decimals)))
    );

    bytes32 salt = keccak256(abi.encode(_remoteToken, _name, _symbol, _decimals));
    _superchainERC20 = CREATE3.deploy(salt, _creationCode);

    deployments[_superchainERC20] = _remoteToken;

    emit SuperchainERC20Deployed(_superchainERC20, _remoteToken _name, _symbol, _decimals);
  }
}
```

The name `_remoteToken` was chosen over `_l1Token` to contemplate future L2 native cases.

**BeaconProxy (SuperchainERC20)**

The Token should follow a simple BeaconProxy implementation, like Openzeppelin:

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/beacon/BeaconProxy.sol

**Beacon Contract**

```solidity
contract SuperchainERC20Beacon is IBeacon {
  /// TODO: Replace with real implementation address
  address internal constant IMPLEMENTATION_ADDRESS = 0x0000000000000000000000000000000000000000;

  function implementation() external pure override returns (address) {
    return IMPLEMENTATION_ADDRESS;
  }
}
```

**Implementation**
The implementation should include

- An `initialize` function that takes `(address _remoteToken, string _name, string _symbol, uint8 decimals)` and stores these in the storage of the BeaconProxy.
- A function to reinitialize after an upgrade.

## User Stories

### `SuperchainERC20` deployment

1. Anyone can deploy a SuperchainERC20 by calling the `deploy` function in the `SuperchainERC20Factory` contract with `_remoteToken`, `_name`, `_symbol` and `decimals` as inputs.
2. The factory stores the `_remoteToken` address for each deployed SuperchainERC20 address.

```mermaid
sequenceDiagram
  participant Alice
  participant FactoryProxy
  participant FactoryImpl
  participant CREATE3Proxy
  participant Token BeaconProxy
  participant Beacon Contract
  participant Implementation

  Alice->>FactoryProxy: deploy(remoteToken, name, symbol, decimals)
  FactoryProxy->>FactoryImpl: delegatecall()
  FactoryProxy-->FactoryProxy: deployments[superchainERC20]=remoteToken
  FactoryProxy->>CREATE3Proxy: create2()
  CREATE3Proxy->>Token BeaconProxy: create()
  Token BeaconProxy-->Beacon Contract: reads implementation()
  Token BeaconProxy->>Implementation: delegatecall()
```

### `SuperchainERC20` implementation upgrade

1. Anyone can deploy a new implementation on each chain to be upgraded.
2. Core developers and Foundation will organize a hardfork between the upgraded chains.
3. The hardfork will update the implementation address on the Beacon Contract.

## Open Questions

- How should different ERC20 implementations be handled? One or multiple implementations per Beacon Contract predeploy?
- How should native token deployment in L2 behave?
