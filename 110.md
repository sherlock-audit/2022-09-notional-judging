hansfriese

high

# `TradingUtils._executeTrade()` doesn't check `preTradeBalance` properly.

## Summary
`TradingUtils._executeTrade()` doesn't check `preTradeBalance` properly.

## Vulnerability Detail
`TradingUtils._executeTrade()` doesn't check `preTradeBalance` properly.

```solidity
function _executeTrade(
    address target,
    uint256 msgValue,
    bytes memory params,
    address spender,
    Trade memory trade
) private {
    uint256 preTradeBalance;

    if (trade.sellToken == address(Deployments.WETH) && spender == Deployments.ETH_ADDRESS) {
        preTradeBalance = address(this).balance;
        // Curve doesn't support Deployments.WETH (spender == address(0))
        uint256 withdrawAmount = _isExactIn(trade) ? trade.amount : trade.limit;
        Deployments.WETH.withdraw(withdrawAmount);
    } else if (trade.sellToken == Deployments.ETH_ADDRESS && spender != Deployments.ETH_ADDRESS) {
        preTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
        // UniswapV3 doesn't support ETH (spender != address(0))
        uint256 depositAmount = _isExactIn(trade) ? trade.amount : trade.limit;
        Deployments.WETH.deposit{value: depositAmount }();
    }

    (bool success, bytes memory returnData) = target.call{value: msgValue}(params);
    if (!success) revert TradeExecution(returnData);

    if (trade.buyToken == address(Deployments.WETH)) {
        if (address(this).balance > preTradeBalance) {
            // If the caller specifies that they want to receive Deployments.WETH but we have received ETH,
            // wrap the ETH to Deployments.WETH.
            uint256 depositAmount;
            unchecked { depositAmount = address(this).balance - preTradeBalance; }
            Deployments.WETH.deposit{value: depositAmount}();
        }
    } else if (trade.buyToken == Deployments.ETH_ADDRESS) {
        uint256 postTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
        if (postTradeBalance > preTradeBalance) {
            // If the caller specifies that they want to receive ETH but we have received Deployments.WETH,
            // unwrap the Deployments.WETH to ETH.
            uint256 withdrawAmount;
            unchecked { withdrawAmount = postTradeBalance - preTradeBalance; }
            Deployments.WETH.withdraw(withdrawAmount);
        }
    }
}
```

It uses `preTradeBalance` to manage the WETH/ETH deposits and withdrawals.

But it doesn't save the correct `preTradeBalance` for some cases.

- Let's assume `trade.sellToken = some ERC20 token(not WETH/ETH), trade.buyToken = WETH`
- Before executing the trade, `preTradeBalance` will be 0 as both `if` conditions are false.
- Then all ETH inside the contract will be converted to WETH and considered as a `amountBought` [here](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L143-L149) and [here](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L61).
- After all, all ETH of the contract will be lost.
- All WETH of the contract will be lost also when `trade.sellToken = some ERC20 token(not WETH/ETH), trade.buyToken = ETH` [here](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L151-L158).

## Impact
All of ETH/WETH balance of the contract might be lost in some cases.

## Code Snippet
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L118-L160

## Tool used
Manual Review

## Recommendation
We should check `preTradeBalance` properly. We can remove the current code for `preTradeBalance` and insert the below code before executing the trade.

```solidity
if (trade.buyToken == address(Deployments.WETH)) {
    preTradeBalance = address(this).balance;
} else if (trade.buyToken == Deployments.ETH_ADDRESS) {
    preTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
}
```
