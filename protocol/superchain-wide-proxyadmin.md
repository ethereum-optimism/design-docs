# Purpose

Outlines the process for transferring control of all L1 Proxy contracts to a single L1 `ProxyAdmin`.

This work assumes the single L1 `ProxyAdmin` matches the [current
implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/universal/ProxyAdmin.sol#L1)
without changes.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

# Problem Statement + Context

Each OP Chain is comprised of
[several](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L1/SystemConfig.sol#L45-L50)
L1 contracts, all of which are upgradeable. Most of these contracts implement upgradeability using
the same [ERC-1967 proxy
pattern](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/universal/Proxy.sol),
however two of them use non-standard proxies with a unique interface:

- The `L1StandardBridge` uses the [`L1ChugSplashProxy`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/legacy/L1ChugSplashProxy.sol)
- The `L1CrossDomainMessenger` uses the
  [`ResolvedDelegateProxy`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/legacy/ResolvedDelegateProxy.sol),
  which depends on an
  [`AddressManager`](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/legacy/AddressManager.sol)
  contract.

To facilitate upgrading all of these contracts, each OP-Chain has its own `ProxyAdmin` contract, which
provide the following benefits:

1. Centralizes the authorization to administer upgrades for all contracts in an OP Chain into a
   single contract, by setting the upgrade admin of all L1 contracts to the `ProxyAdmin`.
2. Abstracts away the different interfaces into a single standard interface for upgrading all contracts.

However the `ProxyAdmin` also presents some challenges in the Superchain environment:

1. It adds one more contract per OP Chain to manage
2. It creates an inconsistency between chains, due to the fact that the `SuperchainConfig` and
   `ProtocolVersions` contracts share their `ProxyAdmin` with OP Mainnet

If we are ever going to transition away from this `ProxyAdmin` it should be sooner than later, as
the number of OP Chains will increase rapidly and the complexity of this project will increase.

# Proposed Solution

We propose transferring all L1 Contracts across all OP Chains to a single L1 ProxyAdmin, that being
the one for OP Mainnet and the shared Superchain contracts at
[0x543bA4AADBAb8f9025686Bd03993043599c6fB04](https://etherscan.io/address/0x543bA4AADBAb8f9025686Bd03993043599c6fB04).

We will use an OPCM based upgrade process, as detailed in the [L1 Upgrades Release
Process](./l1-upgrades.md#release-process).

In this case the `upgrade()` function will have the following pseudocode:

```solidity
error TransferFailed();

struct L1SystemAddresses {
  ProxyAdmin proxyAdmin;
  address l1CrossDomainMessenger;
  address addressManager;
  address l1ERC721Bridge;
  address l1StandardBridge;
  address optimismPortal;
  address optimismMintableERC20Factory;
  address disputeGameFactory;
  address anchorStateRegistry;
  address delayedWeth1;
  address delayedWeth2;
}

function upgrade(ProxyAdmin newProxyAdmin, L1SystemAddresses[] _l1SystemAddresseses) {
  for (uint i = 0; i < _l1SystemAddresseses.length; i++) {
    // for each address in the L1SystemAddresses struct, change the admin via the ProxyAdmin
    // interface
    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(l1CrossDomainMessenger, address(newProxyAdmin));
    _l1SystemAddresseses.proxyAdmin.setProxyType(l1CrossDomainMessenger, ProxyType.RESOLVED);
    _l1SystemAddresseses.proxyAdmin.setAddressManager(addressManager);
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(l1CrossDomainMessenger) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(l1StandardBridge, address(newProxyAdmin));
    _l1SystemAddresseses.proxyAdmin.setProxyType(l1StandardBridge, ProxyType.CHUGSPLASH);
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(l1StandardBridge) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(l1ERC721Bridge, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(l1ERC721Bridge) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(optimismPortal, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(optimismPortal) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(optimismMintableERC20Factory, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(optimismMintableERC20Factory) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(disputeGameFactory, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(disputeGameFactory) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(anchorStateRegistry, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(anchorStateRegistry) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(delayedWeth1, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(delayedWeth1) != address(newProxyAdmin)) {
      revert TransferFailed();
    };

    _l1SystemAddresseses.proxyAdmin.changeProxyAdmin(delayedWeth2, address(newProxyAdmin));
    if(_l1SystemAddresseses.proxyAdmin.getProxyAdmin(delayedWeth2) != address(newProxyAdmin)) {
      revert TransferFailed();
    };
  }
}
```

## Challenges

Unfortunately, this solution does not work if the `newProxyAdmin` uses the existing ProxyAdmin
implementation, as it has the following limitations:

1. Requires the `ProxyAdmin.owner()` to call `setProxyType()` and for the two non-standard proxies. This
   means that the process only works if the new `ProxyAdmin` owner is the same as the current
   `ProxyAdmin` owner, therefore this OPCM could not be used to a key handover.
2. Each `RESOLVED` proxy type has its own `AddressManager`, but the current `ProxyAdmin` design only
   supports a single `AddressManager`.

# Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->
