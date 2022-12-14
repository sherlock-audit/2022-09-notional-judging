lemonmon

high

# `StrategyUtils::_executeDynamicTradeExactIn` does not wrap steth

## Summary

`StrategyUtils::_executeDynamicTradeExactIn` will return bought steth value instead of wrapped steth. Also, steth is not wrapped as it is supposed to be.

## Vulnerability Detail

In the `StrategyUtils::_executeDynamicTradeExactIn`, if `params.tradeUnwrapped` is true, and `buyToken` is `WRAPPED_STETH`, the `buyToken` will be updated to be `WRAPPED_STETH.stETH()`, which is basically `STHETH` (not wrapped). (line 62 in StrategyUtils). So, it buys `stETH` in the trade, and the `amountBought` will be the amount of `stETH` bought. But the `amountBought` was expected to be the `WRAPPED_STETH` amount, as the buyToken given as `WRAPPED_STETH`. The `WRAPPED_STETH` and `STETH` are not 1 to 1, so an user can get more amountBought or less amountBought depending on the market than what is actually bought in `WRAPPED_STETH`.

Later in the same function (line 80-90), if `params.tradeUnwrapped` is true and `buyToken` is `WRAPPED_STETH` and the `amountBought` is bigger than zero, it will wrap the bought stETH to `WARPPED_STETH` and update the `amountBought` to the `WRAPPED_STETH` value. However, this code will be never reached, because if the first two conditions are met, the buyToken would be updated to the `stETH` in the above (line 62).

For example, 100 Wrapped steth will give 108 steth. So, If I choose trade unwrapped to be true, I will get 108 steth, which will be 100 wrapped steth, and get 100 amountBought. But, since my steth bought will not be wrapped, the amountBought returned will be 108, with the buyToken wrapped steth.

## Impact

`StrategyUtils::_executeDynamicTradeExactIn` will return not correct `amountBought`, which will be used in other parts of contract.
If `WRAPPED_STETH` is more expensive, which appears to be the case currently, then an attacker can get more than what is actually sold.

Also, the steth is not wrapped as it is supposed to be, and it leads to accounting error.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/strategy/StrategyUtils.sol?plain=1#L41-L91

```solidity
FILE: StrategyUtils.sol

 41     function _executeDynamicTradeExactIn(
 42         DynamicTradeParams memory params,
 43         ITradingModule tradingModule,
 44         address sellToken,
 45         address buyToken,
 46         uint256 amount
 47     ) internal returns (uint256 amountSold, uint256 amountBought) {
 48         require(
 49             params.tradeType == TradeType.EXACT_IN_SINGLE || params.tradeType == TradeType.EXACT_IN_BATCH
 50         );
 51
 52         // stETH generally has deeper liquidity than wstETH, setting tradeUnwrapped
 53         // to lets the contract trade in stETH instead of wstETH
 54         if (params.tradeUnwrapped && sellToken == address(Deployments.WRAPPED_STETH)) {
 55             sellToken = Deployments.WRAPPED_STETH.stETH();
 56             uint256 unwrappedAmount = IERC20(sellToken).balanceOf(address(this));
 57             // NOTE: the amount returned by unwrap is not always accurate for some reason
 58             Deployments.WRAPPED_STETH.unwrap(amount);
 59             amount = IERC20(sellToken).balanceOf(address(this)) - unwrappedAmount;
 60         }
 61         if (params.tradeUnwrapped && buyToken == address(Deployments.WRAPPED_STETH)) {
 62             buyToken = Deployments.WRAPPED_STETH.stETH();
 63         }
 64
 65         // Sell residual secondary balance
 66         Trade memory trade = Trade(
 67             params.tradeType,
 68             sellToken,
 69             buyToken,
 70             amount,
 71             0,
 72             block.timestamp, // deadline
 73             params.exchangeData
 74         );
 75
 76         (amountSold, amountBought) = trade._executeTradeWithDynamicSlippage(
 77             params.dexId, tradingModule, params.oracleSlippagePercent
 78         );
 79
 80         if (
 81             params.tradeUnwrapped &&
 82             buyToken == address(Deployments.WRAPPED_STETH) &&
 83             amountBought > 0
 84         ) {
 85             IERC20(buyToken).checkApprove(address(Deployments.WRAPPED_STETH), amountBought);
 86             uint256 wrappedAmount = Deployments.WRAPPED_STETH.balanceOf(address(this));
 87             /// @notice the amount returned by wrap is not always accurate for some reason
 88             Deployments.WRAPPED_STETH.wrap(amountBought);
 89             amountBought = Deployments.WRAPPED_STETH.balanceOf(address(this)) - wrappedAmount;
 90         }
 91     }
```

## Tool used

Manual Review

## Recommendation

Rather than overwriting the buyToken, use a new local variable for the stEth, to differentiate when `tradeUnwrapped` is true and buyToken was steth, vs `tradeUnwrapped` is true and buyToken was wrapped steth, which was then overwritten to steth.
Only when `tradeUnwrapped` is true and the original `buyToken` is wrapped steth, convert bought steth to wrapped steth.

Or do not allow `tradeUnwrapped` to be true when buyToken is steth, and overwrite buyToken as steth. Then, if tradeUnwrapped is true and buyToken is steth, wrap the bought steth.

