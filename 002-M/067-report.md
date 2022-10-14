xiaoming90

high

# Rely On Balancer Oracle Which Is Not Updated Frequently

## Summary

The vault relies on Balancer Oracle which is not updated frequently.

## Vulnerability Detail

> Note: This issue affects the MetaStable2 balancer leverage vault

Within the `TwoTokenPoolUtils._getOraclePairPrice` function, it compute the pair price from the Balancer Oracle by calling the `BalancerUtils._getTimeWeightedOraclePrice` function which will in turn call the `IPriceOracle(pool).getTimeWeightedAverage` function to get the  time-weighted average pair prices (e.g. stETH/ETH). The Balancer pool that will be polled for the pair price can be found at https://etherscan.io/address/0x32296969Ef14EB0c6d29669C550D4a0449130230.

The issue is that this pool only handled ~1.5 transactions per day based on the last 5 days' data. In terms of average, the price will only be updated once every 16 hours. There are also many days that there is only 1 transaction. The following shows the number of transactions for each day within the audit period.

- 5 Oct 2022 - 3 transactions
- 4 Oct 2022 - 1 transaction
- 3 Oct 2022 - 1 transaction
- 2 Oct 2022 - 2 transactions
- 1 Oct 2022 - 1 transaction

Note that the price will only be updated whenever a transaction (e.g. swap) within the Balancer pool is triggered. Due to the lack of updates, the price provided by Balancer Oracle will not reflect the true value of the assets. Considering the stETH/ETH Balancer pool, the price of the stETH or ETH provided will not reflect the true value in the market.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L66

```solidity
File: TwoTokenPoolUtils.sol
066:     /// @notice Gets the oracle price pair price between two tokens using a weighted
067:     /// average between a chainlink oracle and the balancer TWAP oracle.
068:     /// @param poolContext oracle context variables
069:     /// @param oracleContext oracle context variables
070:     /// @param tradingModule address of the trading module
071:     /// @return oraclePairPrice oracle price for the pair in 18 decimals
072:     function _getOraclePairPrice(
073:         TwoTokenPoolContext memory poolContext,
074:         OracleContext memory oracleContext, 
075:         ITradingModule tradingModule
076:     ) internal view returns (uint256 oraclePairPrice) {
077:         // NOTE: this balancer price is denominated in 18 decimal places
078:         uint256 balancerWeightedPrice;
079:         if (oracleContext.balancerOracleWeight > 0) {
080:             uint256 balancerPrice = BalancerUtils._getTimeWeightedOraclePrice(
081:                 address(poolContext.basePool.pool),
082:                 IPriceOracle.Variable.PAIR_PRICE,
083:                 oracleContext.oracleWindowInSeconds
084:             );
085: 
086:             if (poolContext.primaryIndex == 1) {
087:                 // If the primary index is the second token, we need to invert
088:                 // the balancer price.
089:                 balancerPrice = BalancerConstants.BALANCER_PRECISION_SQUARED / balancerPrice;
090:             }
091: 
092:             balancerWeightedPrice = balancerPrice * oracleContext.balancerOracleWeight;
093:         }
094: 
095:         uint256 chainlinkWeightedPrice;
096:         if (oracleContext.balancerOracleWeight < BalancerConstants.BALANCER_ORACLE_WEIGHT_PRECISION) {
097:             (int256 rate, int256 decimals) = tradingModule.getOraclePrice(
098:                 poolContext.primaryToken, poolContext.secondaryToken
099:             );
100:             require(rate > 0);
101:             require(decimals >= 0);
102: 
103:             if (uint256(decimals) != BalancerConstants.BALANCER_PRECISION) {
104:                 rate = (rate * int256(BalancerConstants.BALANCER_PRECISION)) / decimals;
105:             }
106: 
107:             // No overflow in rate conversion, checked above
108:             chainlinkWeightedPrice = uint256(rate) * 
109:                 (BalancerConstants.BALANCER_ORACLE_WEIGHT_PRECISION - oracleContext.balancerOracleWeight);
110:         }
111: 
112:         oraclePairPrice = (balancerWeightedPrice + chainlinkWeightedPrice) / 
113:             BalancerConstants.BALANCER_ORACLE_WEIGHT_PRECISION;
114:     }
```

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/BalancerUtils.sol#L21

```solidity
File: BalancerUtils.sol
21:     function _getTimeWeightedOraclePrice(
22:         address pool,
23:         IPriceOracle.Variable variable,
24:         uint256 secs
25:     ) internal view returns (uint256) {
26:         IPriceOracle.OracleAverageQuery[]
27:             memory queries = new IPriceOracle.OracleAverageQuery[](1);
28: 
29:         queries[0].variable = variable;
30:         queries[0].secs = secs;
31:         queries[0].ago = 0; // now
32: 
33:         // Gets the balancer time weighted average price denominated in the first token
34:         return IPriceOracle(pool).getTimeWeightedAverage(queries)[0];
35:     }
```

## Impact

The price provided by the function will not reflect the true value of the assets. It might be overvalued or undervalued. The affected function is being used in almost all functions within the vault. For instance, this function is part of the critical `_convertStrategyToUnderlying` function that computes the value of the strategy token in terms of its underlying assets. As a result, it might cause the following:

  - Vault Settlement - Vault settlement requires computing the underlying value of the strategy tokens. It involves dealing with a large number of assets, and thus even a slight slippage in the price will be significantly amplified.
  - Deleverage/Liquidation of Account - If the price provided does not reflect the true value, users whose debt ratio is close to the liquidation threshold might be pre-maturely deleveraged/liquidated since their total asset value might be undervalued.
  - Borrowing - If the price provided does not reflect the true value, it might be possible that the assets of some users might be overvalued, and thus they are able to over-borrow from the vault.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L66
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/BalancerUtils.sol#L21

## Tool used

Manual Review

## Recommendation

Although it is not possible to obtain a price pair that truly reflects the true value of an asset in the real world, the vault should attempt to minimize inaccuracy and slippage as much as possible. This can be done by choosing and using a more accurate Oracle that is updated more frequently instead of using the Balancer Oracle that is infrequently updated. 

Chainlink should be used as the primary Oracle for price pair. If a secondary Oracle is needed for a price pair, consider using [Teller](https://tellor.io/) Oracle instead of Balancer Oracle. Some example of how Chainlink and Tellor works together in a live protocol can be found [here](https://www.liquity.org/blog/price-oracles-in-liquity)

Obtaining the time-weight average price of BTP LP token from Balancer Oracle is fine as the Balancer pool is the source of truth. However, getting the price of ETH or stETH from Balancer Oracle would not be a good option. 

On a side note, it was observed that the weightage of the price pair is Balancer Oracle - 60% and Chainlink - 40%. Thus, this theoretically will reduce the impact of inaccurate prices provided by Balancer Oracle by around half. However, the team should still consider using a better Oracle as almost all the functions within the vault depends on the accurate price of underlying assets to operate.

Note: For the stETH/ETH balancer leverage vault, the price pair is computed based on a weighted average of Balancer Oracle and Chainlink. Based on the test script, the weightage is Balancer Oracle - 60% and Chainlink - 40%.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/scripts/BalancerEnvironment.py#L45

```python
File: BalancerEnvironment.py
45:             "maxRewardTradeSlippageLimitPercent": 5e6,
46:             "balancerOracleWeight": 0.6e4, # 60%
47:             "settlementCoolDownInMinutes": 60 * 6, # 6 hour settlement cooldown
```