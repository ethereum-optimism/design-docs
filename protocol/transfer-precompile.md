# Purpose

Since its inception, the Celo blockchain provides a feature called "[Token
Duality](https://specs.celo.org/token_duality.html)", which allows the native
token to be used both in the usual ETH-like way and as ERC-20 token.

We would like to bring this feature to more OP-Stack based chains and bring the
Celo L2 closer to vanilla OP-stack by adding the transfer precompile to
op-geth.

# Summary

Add a precompile to op-geth that allows transferring native tokens.
This precompile can only be called by a single privileged contract.

# Problem Statement + Context

Many users and contracts have to deal with both ERC20 tokens and native tokens.
Since they can't be accessed by the same interface, they either have to be
handled as two different cases or be explicitly wrapped by an ERC-20 token
wrapping contract.
Wrapping tokens is not an ideal solution either, since the wrapped token is
of limited use (e.g. can't be used to pay gas) and wrapping/unwrapping is
annoying and incurs additional gas charges.

This can be solved by having a contract that provides an ERC-20 interface
directly to the native token. To implement such a contract, the blockchain must
provide a way to transfer native tokens on behalf of the tx sender.

# Proposed Solution

Add one new blockchain parameter `TransferPrecompileAllowed` and a new precompile.
The parameter defines the address from which the precompile must be called.

Precompile name: `transfer`  
Precompile address: 0x0000000000000000000000000000000000000101 (directly after the `P256VERIFY` precompile from Fjord)  
Parameters (abi-encoded): `address from, address to, uint256 value`  
Gas costs: 9000  
Result: `value` native tokes will be transferred from `from` to `to`.  
No return value  

The precompile can fail for the following reasons:
* `TransferPrecompileAllowed` is not configured for the blockchain
* It is not called from the contract at the `TransferPrecompileAllowed` address
* The `from` address has less than `value` native tokens
* The input is not exactly 96 bytes long
* The input parameters can't be parsed correctly

Example usage in solidity:

```solidity
address constant TRANSFER = address(0x0000000000000000000000000000000000000101);

function _transfer(address to, uint256 value) internal {
	(success, ) = TRANSFER.call.value(0).gas(gasleft())(abi.encode(msg.sender, to, value));
	require(success, "transfer failed");
	emit Transfer(msg.sender, to, value);
}
```

The `from` parameter should always be set to `msg.sender` so that each address
can only transfer its own funds.
Removing the `from` parameter from the precompile is not an option because 
`msg.sender` can't be accessed from inside the precompile.
There, the caller is set to the contract doing `TRANSFER.call`.

# Risks & Uncertainties

* The privileged contract at `TransferPrecompileAllowed` can execute arbitrary transfers, so it must be a well known and safe contract.
* Like all precompiles, it must be added as part of a hardfork.

# Open questions
* Where is the right place for the `TransferPrecompileAllowed` configuration? Should it be part of `OptimismConfig`?
* If `TransferPrecompileAllowed` is not configured, is it ok to have the precompile, although it will always revert? Or would it be better to remove it from the list of active precompiles?

# History

The concept outlined in this document is in production use on the Celo mainnet since its launch in May 2020.
As part of Celo's L2 migration, support for it has been added to a fork of the OP-stack.

Code examples:
* [The transfer precompile in the op-geth fork](https://github.com/celo-org/op-geth/blob/celo7/core/vm/celo_contracts.go)
* [The CELO token using the precompile](https://github.com/celo-org/celo-monorepo/blob/2ace1d08b9da5d2618e945eef7663cc385917a2d/packages/protocol/contracts/common/GoldToken.sol#L302-L315) to provide access to the native token
