0x52

medium

# UniV2Adapter#getExecutionData doesn't properly handle native ETH swaps

## Summary

UniV2Adapter#getExecutionData doesn't properly account for native ETH trades which makes them impossible. Neither method selected supports direct ETH trades, and sender/target are not set correctly for TradingUtils_executeTrade to automatically convert

## Vulnerability Detail

    spender = address(Deployments.UNIV2_ROUTER);
    target = address(Deployments.UNIV2_ROUTER);
    // msgValue is always zero for uniswap

    if (
        tradeType == TradeType.EXACT_IN_SINGLE ||
        tradeType == TradeType.EXACT_IN_BATCH
    ) {
        executionCallData = abi.encodeWithSelector(
            IUniV2Router2.swapExactTokensForTokens.selector,
            trade.amount,
            trade.limit,
            data.path,
            from,
            trade.deadline
        );
    } else if (
        tradeType == TradeType.EXACT_OUT_SINGLE ||
        tradeType == TradeType.EXACT_OUT_BATCH
    ) {
        executionCallData = abi.encodeWithSelector(
            IUniV2Router2.swapTokensForExactTokens.selector,
            trade.amount,
            trade.limit,
            data.path,
            from,
            trade.deadline
        );
    }

UniV2Adapter#getExecutionData either returns the swapTokensForExactTokens or swapExactTokensForTokens, neither of with support native ETH. It also doesn't set spender and target like UniV3Adapter, so _executeTrade won't automatically convert it to a WETH call. The result is that all Uniswap V2 calls made with native ETH will fail. Given that Notional operates in native ETH rather than WETH, this is an important feature that currently does not function.

## Impact

Uniswap V2 calls won't support native ETH

## Code Snippet

[UniV2Adapter.sol#L12-L52](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/adapters/UniV2Adapter.sol#L12-L52)

## Tool used

Manual Review

## Recommendation

There are two possible solutions:

1) Change the way that target and sender are set to match the implementation in UniV3Adapter
2) Modify the return data to return the correct selector for each case (swapExactETHForTokens, swapTokensForExactETH, etc.)

Given that the infrastructure for Uniswap V3 already exists in TradingUtils_executeTrade the first option would be the easiest, and would give the same results considering it's basically the same as what the router is doing internally anyways.
