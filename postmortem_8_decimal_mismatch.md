Welcome to another postmortem. here i will explain how different decimals can lead to incorrect conversion and calculation.

We knows that each ERC20 tokens has different decimals, for example USDC and USDT has 6 decimals, WETH has 18 decimals etc.

And assuming every token has same decimal while calculations can lead to reward loss and ronding issue. there could be many other consequences.

I want to ask you what if same token has different decimals? strage right? it is possible in case of USDC as far as i know. The issue was found on sherlock contest `Super DCA Liquidity Network`. Where the protocol assuming USDC token decimals as `6` for all chain it was deployed on.

Among all chains there was BNB on which this protocol is gonna deployed. However the USDC token has `18` decimals on BNB chain. While the protocol was assuming USDC decimals as `6` for all chains and using hardcoded scaling factor `1e12` for conersion as shown below

```solidity
  /// @notice Convert an amount from 1e18 precision (flow rate) to 1e6 precision (USDC)
  /// @param amount The amount in 1e18 precision
  /// @return convertedAmount The amount converted to 1e6 precision
  function _convertToUSDCPrecision(uint256 amount) internal pure returns (uint256 convertedAmount) {
    convertedAmount = amount / 1e12;
  }
```


due to this conversion fund loss could be happen.

> Always keep in mind, while auditing always consider the chains on which the protocol decided to deploy on. Also consider different consequences regarding each chain and their impact on the protocol.