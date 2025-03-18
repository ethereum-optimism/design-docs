# Purpose

This document proposes converting `SuperchainWETH` into a dedicated ETH bridge for facilitating `ETH` transfers within the interop cluster - `SuperchainETHBridge`.

# Summary

The `SuperchainWETH` contract will be renamed to `SuperchainETHBridge` and stripped of its `WETH98` and `IERC7802` functionality, retaining only ETH bridging capabilities.

# Problem Statement + Context

`SuperchainWETH` was initially introduced to facilitate `ETH` transfers within the interop cluster ([design doc](./interoperable-ether.md)). Sending `ETH` between chains in the interop cluster required three steps:
1. wrap `ETH` to `SuperchainWETH`
2. send the `SuperchainWETH` to the destination through the `SuperchainTokenBridge`
3. unwrap the `SuperchainWETH` back to `ETH`

To simplify this process, the `SuperchainWETH` contract was updated to support direct ETH bridging through `sendETH` and `relayETH` functions ([design doc](./interoperable-ether-transfers.md)).

With this implementation of `SuperchainWETH`, several issues exist:

- Having two versions of `WETH` (`WETH` and `SuperchainWETH`) creates confusion for developers and users
- Liquidity could become fragmented between `WETH` and `SuperchainWETH` (e.g., through separate DeFi liquidity pools)
- Both tokens share the same symbol, making them difficult to distinguish from a user perspective
- Poor separation of concerns: `SuperchainWETH` acts as both a token and a bridge
- Cross-chain ETH transfer indexing requires tracking events across multiple contracts (`SuperchainTokenBridge` and `SuperchainWETH`)

# Proposed Solution

The solution involves the following changes:

1. Remove the `WETH98` and `IERC7802` extensions from the contract, leaving only `sendETH` and `relayETH` functions.
2. Rename `SuperchainWETH` to `SuperchainETHBridge` to better reflect its primary purpose

## Implementation Details

The simplified contract would look like:

```solidity:contracts/SuperchainETHBridge.sol
// Previous SuperchainWETH contract, stripped down to only ETH bridging functionality
contract SuperchainETHBridge {
    function sendETH(address to, uint256 chainId) external payable returns (bytes32 msgHash_) {
        // ... existing implementation ...
    }

    function relayETH(address _from, address _to, uint256 _amount) external {
        // ... existing implementation ...
    }
}
```

### Advantages

- Clear separation of concerns between `ETH` bridging and wrapped `ETH`
- Simplified developer experience
- Reduced risk of user confusion

# Risks & Uncertainties

- The protocol will not provide a `SuperchainERC20` compatible `WETH` predeploy. Cross-chain `WETH` transfers would require either unwrapping/wrapping `ETH` or relying on third-party `WETH` implementations that support the `SuperchainERC20` interface.

# Alternatives Considered

## Upgrade Existing `WETH` to be a `SuperchainERC20`

### Advantages
- Single representation of `WETH`

### Disadvantages
- Does not solve the confusion around dual bridge/token functionality
- The `WETH` contract is not upgradeable, so this would require manual intervention to upgrade the contract.

# Timeline

The changes should be implemented before interop reaches testnet.