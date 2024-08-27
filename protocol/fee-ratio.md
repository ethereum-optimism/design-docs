# Purpose

The purpose of this document is to address the exchange rate consideration between the native token and ether on the custom gas token chain.

# Summary

The `feeRatio` field is introduced in the `GasPriceOracle` contract on L2, with real-time updates provided by an oracle to represent the exchange rate between ether and the native token. 

# Problem Statement + Context
On a custom gas token chain, there are two types of user costs that need to be reconsidered:
- L1Cost: It requires ether to purchase data availability for Ethereum.
- L2Cost: This cost involves transactions similar to those in Ethereum, executed by the EVM on L2. Since L2 and L1 have equivalent EVM execution, it should also be paid in ether.

For L2 fees, the purpose cannot be achieved by simply adjusting gas used or gas price, as this could confuse users. Therefore, handling should only occur during asset balance verification, deduction and addition.

Both of these considerations can be resolved by introducing a fee ratio:
- L1Cost = l1CostFn * feeRatio
- L2Cost = ((gas * gasPrice) + (blobGas * blobGasPrice)) * feeRatio

> For chains where a custom gas token is not set, the feeRatio is 1, and the original calculations remain unaffected.

# Proposed Solution
## Involve feeRatio
Modify the precompile contract `GasPriceOracle` by adding the feeRatio and a role for chain operator. The exchange ratio will be updated by an oracle, with data sourced from exchanges, Uniswap, or any other valid price provider
```solidity
// Allows the operator to modify the fee ratio.
function setFeeRatio(uint256 _feeRatio) external onlyOperator {
    uint256 previousFeeRatio = feeRatio;
    feeRatio = _feeRatio;
    emit FokenRatioUpdated(previousFeeRatio, feeRatio);
}
```
The `feeRatio` is read and updated as needed, similar to other fields in the L1Block. However, unlike other L1 fields, the feeRatio is not regularly updated in each `l1infotx`. Therefore, during the execution of L2 transactions, it is necessary to check the msg information. 

`if (msg.to == GasOracleAddr && bytes.Equal(msg.data[0:4], SetFeeRatiosSelector))`, meaning the operator has updated the feeRatio, the slot field needs to be re-loaded.

## L1cost Adaptation
The logic for calculating L1Cost in smart contracts and EVM execution needs to be updated accordingly.

In smart contracts：
```solidity
function getL1Fee(bytes memory _data) external view returns (uint256) {
    if (isFjord) {
    return _getL1FeeFjord(_data) * feeRatio;
    } else if (isEcotone) {
    return _getL1FeeEcotone(_data) * feeRatio;
    }
    return _getL1FeeBedrock(_data) * feeRatio;
}
```
During EVM execution, the `newL1CostFunc*` for each hardforks should be multiplied by the feeRatio.

## Transaction Validation and Fee Deduction
The logic mainly focuses on the txpool and state_transition, where the final effect changes the actual cost of a transaction from `(gas * gasPrice) + (blobGas * blobGasPrice) + value` to `((gas * gasPrice) + (blobGas * blobGasPrice)) * feeRatio + value`.

Key areas that require handling include:
- When the user spends native tokens to purchase gas during transaction execution.
- When returning the remaining gas to the user after transaction execution.
- When verifying the legitimacy of transactions related to account state checks.
- When accumulating L2Cost in `Coinbase`, `OptimismBaseFeeRecipient`, and `list.local`.

# Alternatives Considered
Updating the feeRatio in the L1 SystemConfig, with the following update path: L1: SystemConfig -> op-node: l1InfoTx -> L2: L1Block. This approach would reduce code changes in execution layer but could result in higher L1 costs and less flexibility.

# Risks & Uncertainties
Before executing a transaction, Wallets and SDKs need to estimate the user’s balance, both L1 and L2 fees should be considered. The L1 fee can be obtained using getL1Fee, while the L2 fee can only be estimated as the lowest possible gas limit via `eth_estimateGas`, and it should be multiplied by the feeRatio.