# Summary

*The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

A new factory predeploy to spin up `ISuperchainERC20` `MirrorsERC20s` of `OptimismMintableERC20Tokens` is required. This document aims to provide a design for such a factory.

# Overview

- The `OptimismMintableERC20Factory` is a predeploy proxy with a corresponding implementation.
- `OptimismMintableERC20Factory` deploys `OptimismMintableERC20` contracts. The `OptimismMintableERC20s` are not proxied.
- The `MirrorERC20Factory` is a non-proxied predeploy. It deploys `MirrorERC20` tokens—which are proxied—using the beacon pattern. This allows upgrading all `MirrorERC20` implementations at once if needed.
- The `MirrorERC20Factory` will act as the beacon for the `MirrorERC20s`.


# Problem Statement + Context

A vast amount of `OptimismMintableERC20Tokens` currently exist in the ecosystem. These tokens do not have interoperable properties. Therefore, they require a migration to become interoperable. The migration will be performed through `MirrorERC20s`, as specified by [Mirror Standard](https://github.com/ethereum-optimism/design-docs/pull/36). These `MirrorERC20s` must be easily deployed through a predefined factory to ensure they share the same functionality and address across chains and also to verify their legitimacy.

The factory must facilitate the creation of a single legitimate `MirrorERC20` per `OptimistismMintableERC20Token` to avoid liquidity fragmentation and unnecessary complexity.

# Alternatives Considered
- Using `CREATE`. Discarded due to nonce dependability.
- Using `CREATE2`. This solution is viable as long as the creation code is never modified. Because this is not a guarantee, this alternative was discarded in favor of `CREATE3`.

# Proposed Solution

The factory will be a predeploy that will be in charge of:
- Deploying the proxies behind the `Mirrors`.
- Holding the `Mirror` implementation address.
- Tracking what `Mirrors` it deployed.

The factory will use `CREATE3` for the deployment of the `Mirrors`. `CREATE3` relies on the address of the factory—which must be the same across the superchain—and on the chosen salt to determine the address where the `Mirror` will be deployed. It eliminates any ties with the `Mirror`'s creation code, which ensures the address calculation will stay the same should a change take place in a future upgrade. This implies that if the `token` is used as the salt, and the factory is the same across the superchain, then the address of the `Mirror` will be the same on all chains. This is a required property for the `Mirror'`s interoperability features to work correctly.

Furthermore, the factory will hold the implementation address of the `Mirror`. In other words, it will act as beacon of all the `Mirrors` it deploys. This carries the benefit of allowing the implementation to be updated when the factory is upgraded and not requiring an extra predeploy to act as beacon.

Lastly, the factory will store every `Mirror` it deploys. This will allow integrating contracts to perform validate whether a `Mirror` is a child of the official factory or not. This is an example of how the `L2StandardBridge` could implement the check in the `mirrorMint` function required by the `Mirror Standard`:

```solidity
function mirrorMint(address _token, uint256 _amount) external {
  address _mirror = MIRROR_FACTORY.mirrors(_token);
  if (_mirror != msg.sender) revert();
  /// rest of the logic
}

```

For this to work and be secure, certain invariants must hold:
- The factory MUST only deploy a single `Mirror` per token.
- The factory MUST store ALL `Mirrors` it deploys. 
- Upgrades must not modify the values in the `mirrors` mapping.


Having these characteristics in mind, at the time of deployment of the `Mirrors`, the factory address and the corresponding token will be passed appended to the creation code so they are interpreted as constructor arguments. The factory address is needed to tell the proxy where to look for the implementation, so it will act as the beacon, and the token is needed to tell the implementation what token is extending.    

Example code:

```solidity
contract MirrorERC20Factory is Semver {
  address public implementation;
  mapping(address token => address mirror) public mirrors;

  event MirrorDeployed(address indexed mirror, address indexed token);

  constructor() is Semver(1,0,0) {
    implementation = new MirrorSuperchainERC20();
  }

  function deploy(address _token) external returns (address _mirror) {
    require(_isOptimismMintableERC20(_token), "not OptimismMintableERC20Token");

    bytes memory _creationCode = abi.encodePacked(type(Proxy).creationCode, abi.encode(address(this), _token));

    _mirror = CREATE3.deploy({salt: _token, creationCode: _creationCode, value: 0 });

    mirrors[_token] = _mirror;

    emit MirrorDeployed(_mirror, _token);
  }

  function _isOptimismMintableERC20(address _token) internal view returns (bool) {
    return ERC165Checker.supportsInterface(_token, type(ILegacyMintableERC20).interfaceId)
    || ERC165Checker.supportsInterface(_token, type(IOptimismMintableERC20).interfaceId);
  } 
}

```

# User Stories
- A user wants an existing `OptimismMintableERC20Token` to become interoperable. To do this he can deploy the token's corresponding `MirrorERC20` by interacting with the `MirrorERC20Factory`.
- A user creates an `OptimismMintableERC20Token` through the `OptimismMintableERC20Factory`. Later on, he, or any other user, can deploy its corresponding `MirrorERC20` token through the `MirrorERC20Factory`.
- A user wants to make a token that does not comform to the `IOptimismMintableERC20` interface interoperable. The `MirrorERC20Factory` will not be of use, as it checks for this condition. Even if it were to deploy a `MirrorERC20Factory` for a token that fakes the interface compatibility, if that token does not give the `L2StandardBridge` permissions to `burn` and `mint`, its `transfer` and `transferFrom` functions would not work. 

# Open Questions
- Should the factory be proxied, or is it enough for it to be upgraded during hardforks?
- Do we need to have versioning in the factory?
- Is the `_isOptimismMintableERC20` check needed? The mirror of a token that doesn't grant the `L2StandardBridge` `burn` and `mint` permissions will have its `transfer` and `transferFrom` functions revert. So it will be useless. 
- How will the Proxy behind the `Mirrors` look like? A modified BeaconProxy to accomodate for the token will be sufficient? Should we add its design to the Mirror Standard PR?

# Risks & Uncertainties

This only accounts for `OptimismMintableERC20Tokens`. Legacy tokens, that is tokens that implement the `ILegacyMintableERC20` interface will work as well. The design space is limited for simplicity. New interoperable tokens using the `Superc20` standard without Mirrors, are not taken into account for this factory. Tokens that don't comply to the `IOptimismMintableERC20` interface will not work with mirrors due to the permissions needed.