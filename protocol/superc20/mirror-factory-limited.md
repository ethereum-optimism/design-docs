# Summary

*The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

A factory to spin up `ISuperchainERC20` Mirrors of `OptimismMintableERC20Tokens` is required. This document aims to provide a design for such a factory. The Mirrors need to be proxied to address any future addition of features. The factory will deploy these proxies. Therefore, from now on, when we say `Mirror` we refer to the Proxy and not the implementation.

# Problem Statement + Context

A vast amount of `OptimismMintableERC20Tokens` currently exist in the ecosystem. These tokens do not have interoperable properties. Therefore, they require a migration to become interoperable. The migration will be performed through Mirrors, as specified by [Mirror Standard](https://github.com/ethereum-optimism/design-docs/pull/36). These Mirrors must be easily deployed through a predefined factory to ensure all Mirrors share the same functionality and address across chains and also to verify their legitimacy.

The factory must facilitate the creation of a single legitimate Mirror per `OptimistismMintableERC20Token` to avoid liquidity fragmentation and unnecessary complexity.

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

Having these characteristics in mind, at the time of deployment of the `Mirrors`, the factory address and the corresponding token will be passed appended to the creation code so they are interpreted as constructor arguments. The factory address is needed to tell the proxy where to look for the implementation, so it will act as the beacon, and the token is needed to tell the implementation what token is extending.    

Example code:

```solidity
contract MirrorFactory {
  string public constant MIRROR_VERSION = "1.0.0";

  address public implementation;
  mapping(address token => address mirror) public mirrors;

  event MirrorDeployed(address indexed mirror, address indexed token);

  constructor() {
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
# Open Questions
- Do we need to have versioning in the factory?
- Is the `_isOptimismMintableERC20` check needed? The mirror of a token that doesn't grant the `L2StandardBridge` `burn` and `mint` permissions should be useless.
- How will the Proxy behind the `Mirrors` look like? A modified ERC1967Proxy will be sufficient? Should we add its design to the Mirror Standard PR?

# Risks & Uncertainties

This only accounts for `OptimismMintableERC20Tokens`. The design space is limited for simplicity. Legacy tokens are not considered. New interoperable tokens using the `Superc20` standard without Mirrors, are not taken into account for this factory.
Current `OptimismMintableERC20Tokens` deployed by OP and Ethereum mainnet factories have different versions and, therefore, different bytecodes. This should not be a problem as long as these tokens grant `burn` and `mint` permissions to the `L2StandardBridge` and conform to the `ERC20` interface.