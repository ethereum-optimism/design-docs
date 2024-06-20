# Summary

*The work present below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

A factory to spin up `ISuperchainERC20` Mirrors of `OptimismMintableERC20Tokens` is required. This document aims to provide a design for such a factory.

# Problem Statement + Context

A vast amount of `OptimismMintableERC20Tokens` currently exists in the ecosystem. These tokens do not have interoperable properties. Therefore, they require a migration to become interoperable. The migration will be performed through Mirrors, as specified by [Mirror Standard](https://github.com/ethereum-optimism/design-docs/pull/36). These Mirrors must be easily deployed through a predefined factory to ensure all Mirrors share the same functionality and address across chains and also to verify their legitimacy.

The factory must facilitate the creation of a single legitimate Mirror per `OptimistismMintableERC20Token` to avoid liquidity fragmentation and unnecessary complexity.

# Alternatives Considered

- Creation Methods:
    - Using `CREATE`. Discarded due to nonce dependability.
    - Using `CREATE3`. Discarded due to unnecessary complexity.
- Upgradability
    - Deploying the `Mirror` without a `Proxy` and simply updating the `mirrors`' mapping to point to a new address when the Mirror is upgraded was discarded. Maintaining the `Mirrors'` addresses across chains is crucial *at all times,* and this approach could cause synchronization issues where one `Mirror` was updated in a given chain but not in another, resulting in different mirror addresses for the same token.

# Proposed Solution

Limiting the scope to `OptimismMintableERC20Tokens` allows using `CREATE2` instead of `CREATE3`, which is more expensive and complex. This is because the variables CREATE2 uses to define the address are salt, the child's init code, and the sender's address, which provide us with all the needed properties.

The `Mirror` needs to be upgradable. Therefore, a Proxy contract will take the `_token` as an argument and `delegatecall` to the `Mirror` implementation.

> [!NOTE]
>💡 From now on, when we say `Mirror`, we’re referring to the proxy, not the implementation.

Using the `OptimismMintableERC20Token` address as the salt allows linking the `Mirror` to this specific token. The sender's address will be the factory. The factory will be a pre-deploy which must be maintained across chains. This means the same `Mirror` address for the given salt/token will be kept.

This gives us:

- Address predictability. For each token, we know the corresponding Mirror.
- Consistent addresses across chains.
- Simple factory code.

> [!NOTE]
> 💡 An important note is that L1 must also be able to deploy `Mirror` contracts. We don’t have control over the factory address on L1. Therefore, to ensure the same `Mirror` address for a given token is maintained across chains, the factory pre-deploy will not follow the `0x42.....N` standard, but it will have to match the factory’s deployment address on L1 in all chains.

The salt doesn't need to be more than the `OptimismMintableERC20Token` address. The `name` and `symbol` can be queried from the implementation code. The `Mirror` will take in the token it extends as an argument and the factory address as its admin to allow for upgrades.

The factory should be upgradable or have setters to allow the creation of `Mirrors` with new implementations. This means the factory should also allow old, already-deployed `Mirrors` to be updated to new ones. A function `upgradeMirror` is introduced to accomplish this.

We can track versioning through numbers or the addresses of the implementations used.

The mirror implementation contract should be deployed beforehand. The Proxy must properly match storage slots with the mirror implementation storage.

This design allows the `StandardBridge` to call `mirrors(_token)` when a `Mirror` contract performs a `transfer` and validates the mirror belongs to the `_token`. As an illustrative example:

```solidity
function mirrorMint(address _token, uint256 _amount) external {
	address _mirror = MIRROR_FACTORY.mirrors(_token);
	if (_mirror != msg.sender) revert();
	///...mint logic
}

```

Example code:

```solidity
contract MirrorFactory {
	string public constant MIRROR_VERSION = "1.0.0";
	Mirror public constant MIRROR_IMPL = <PLACEHOLDER_ADDRESS>;

	mapping(address token => address _mirror) public mirrors;
	// this could be a mapping to IMPLs
	mapping(address token => string version) public mirrorVersions;

	function deployMirror(address _token) external returns (address) {
		// Check if its `OptimismMintableERC20Token`?
		if (mirrors[_token] != address(0)) revert();
		Proxy _mirror = address(new Proxy{salt: keccak256(abi.encode(_token))}(address(this), _token));
		_updateMirror(_mirror, _token);
	}

	function upgradeMirror(address _mirror) external {
		if (mirror[token] == address(0)) revert();
		if (mirrorVersions[mirror] == MIRROR_VERSION) revert();
		_updateMirror(_mirror, _token);
		//..........
	}

	function _updateMirror(address _mirror, address _token) internal {
		mirrors[_token] = _mirror;
		mirrorVersions[_mirror] = MIRROR_VERSION;
		_mirror.setImplementation(MIRROR_IMPL);
	}

	// Note: we could fuse both functions into deployMirror.
	// Separating them here for clarity.
}

```

# Risks & Uncertainties

This only accounts for `OptimismMintableERC20Tokens`. The design space is limited for simplicity. Legacy tokens with different bytecodes are not considered. New interoperable tokens using the `Superc20` standard, which will have a different bytecode as well, are not taken into account for this factory.

If the design space were to be extended `Create3` and extra predeploys would be needed.

Current `OptimismMintableERC20Tokens` deployed by OP and Ethereum mainnet factories have different versions and, therefore, different bytecodes. This should not be a problem as the Mirror does not care about these changes.