# Summary

*The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

A factory to spin up `ISuperchainERC20` Mirrors of `OptimismMintableERC20Tokens` is required. This document aims to provide a design for such a factory. The Mirrors need to be proxied to address any future addition of features. The factory will deploy these proxies. Therefore, from now on, when we say `Mirror` we refer to the Proxy and not the implementation.

# Problem Statement + Context

A vast amount of `OptimismMintableERC20Tokens` currently exist in the ecosystem. These tokens do not have interoperable properties. Therefore, they require a migration to become interoperable. The migration will be performed through Mirrors, as specified by [Mirror Standard](https://github.com/ethereum-optimism/design-docs/pull/36). These Mirrors must be easily deployed through a predefined factory to ensure all Mirrors share the same functionality and address across chains and also to verify their legitimacy.

The factory must facilitate the creation of a single legitimate Mirror per `OptimistismMintableERC20Token` to avoid liquidity fragmentation and unnecessary complexity.

# Alternatives Considered
- Using `CREATE`. Discarded due to nonce dependability.
- Using `CREATE3`. Viable, yet not preferred due to increased complexity.

# Proposed Solution

Limiting the scope to `OptimismMintableERC20Tokens` allows using `CREATE2` instead of `CREATE3`, which is more expensive and complex. This is because the variables CREATE2 uses to define the address of the deployed contract provides all the properties we need to maintain the same address accross chains and easily verify whether a mirror was deployed by the official factory or not:

- `Salt`: If we use the `token` address, we can link the `Mirror` to this `token`. This will allow other contracts to know whether a caller is a legitimate `Mirror` or not.
- `Sender Address`: In this case, it will be the Factory address itself, or the address of the factory's proxy. This works well, as it the factory can be deployed at the same address accross different chains. For chains in the superchain, it will be a predeploy. This predeploy, however, won't have a pretty address. Instead, it has to be deployed at the address where the L1 Factory is deployed so the address is maintained for all chains.
- `Init Code`: The same proxy's init code must be used accross chains.

The salt doesn't need to be more than the `OptimismMintableERC20Token` address. The `name` and `symbol`, and `decimals` can be fetched with the `token` address alone in the implementation. The factory will be the `Mirror`'s admin, and the proxy will have to either call `setStorage` immediately with the `token` at the time of deployment, or add a `TOKEN_SLOT` and update it on the constructor.

The factory should be upgradable or have setters to allow the modification of the implementations for the current `Mirrors`. The deployment and update of already-deployed `Mirrors` to new implementations should be open to anyone.

We can track versioning through numbers or the addresses of the implementations used.

This design allows the `StandardBridge` or any contract that needs to check the legitimacy of a `Mirror` to call `MIRROR_FACTORY.getMirrorAddress()` and validate whether the calling user is a mirror that corresponds to a given `token`. As an illustrative example of a potential function of the `StandardBridge`:

```solidity
function mirrorMint(address _token, uint256 _amount) external {
	address _mirror = MIRROR_FACTORY.getMirrorAddress(_token);
	if (_mirror != msg.sender) revert();
	///...mint logic
}

```

Example code:

```solidity
contract MirrorFactory {
	string public constant MIRROR_VERSION = "1.0.0";
	Mirror public constant MIRROR_IMPL = <PLACEHOLDER_ADDRESS>;

	function deployMirror(address _token) external returns (address) {
		require(_isOptimismMintableERC20(_token), "not OptimismMintableERC20Token");

		// Note: There's no _token slot in this proxy
		Proxy _mirror = address(new L2ChugSplashProxy{salt: keccak256(abi.encode(_token))}(address(this), _token));

		// Note: This function does not exist on L2ChugSplashProxy.
		_mirror.setImplementation(MIRROR_IMPL);
	}

	function upgradeMirror(address _mirror) external {
		// Note: This function does not exist on L2ChugSplashProxy.
		_mirror.setImplementation(MIRROR_IMPL);
	}

	function _isOptimismMintableERC20(address _token) internal view returns (bool) {
        return ERC165Checker.supportsInterface(_token, type(ILegacyMintableERC20).interfaceId)
            || ERC165Checker.supportsInterface(_token, type(IOptimismMintableERC20).interfaceId);
    }

	function getMirrorAddress(address _token) external view returns (address _mirror) {
		_mirror = address(uint160(uint(keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            keccak256(abi.encode(_token)),
            keccak256(abi.encodePacked(
                type(L2ChugSplashProxy).creationCode,
                abi.encode(_token)
            ))
        )))))
	} 
}

```

# Risks & Uncertainties

This only accounts for `OptimismMintableERC20Tokens`. The design space is limited for simplicity. Legacy tokens are not considered. New interoperable tokens using the `Superc20` standard without Mirrors, are not taken into account for this factory.

If the design space were to be extended `Create3` and extra predeploys would be needed.

Current `OptimismMintableERC20Tokens` deployed by OP and Ethereum mainnet factories have different versions and, therefore, different bytecodes. This should not be a problem as long as these tokens grant `burn` and `mint` permissions to the `StandardBridge` and conform to the `ERC20` interface.