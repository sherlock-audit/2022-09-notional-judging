0x52
# TradingUtils#_executeTrade contains logical error that can cause loss of funds if trade.buyToken is ETH or WETH

## Summary

TradingUtils#_executeTrade contains a logical error that leads to incorrect deposits (withdrawals) of ETH (WETH) balance when the buyToken is WETH (ETH). These deposits (withdrawals) are treated as swap outputs which can result in either valid transactions reverting or failure of TradingUtils#_executeTrade to enforce desired slippage bounds. If this error occurs when using ZeroEx, the user may lose part or all of their slippage protection which can cause loss of user funds.

## Vulnerability Detail

[TradingUtils.sol#L140-L159](https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradingUtils.sol#L140-L159)

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

The above lines are executed after the contract call in TradingUtils#_executeTrade. The purpose it to deposit (withdraw) ETH (WETH) to match the token requested by the user. The logic contains an error and will always result in the user's entire balance of ETH (WETH) being deposited (withdrawn) when the buyToken is WETH (ETH). When the sell token isn't ETH or WETH preTradeBalance will not be initialized correctly and will be 0. If the user had any ETH (WETH) before the trade it won't be accounted for by preTradeBalance and will be deposited (withdrawn) after the call. This will artificially inflate the postTradeBuyBalance in TradingUtils#_executeInternal potentially causing several issues:

1) Exact out swaps will revert because in TradingUtils#_postValidate amountReceived will be greater than trade.amount
2) Slippage protection enforced by TradingUtils#_postValidate will not function correctly because part of the balance wasn't from the trade. For most exchanges, this is not an issue because slippage bounds are enforced by the swap router. When using ZeroEx, slippage bounds are not enforced on the swap level and instead rely on the protection from TradingUtils#_postValidate. The result is that when using ZeroEx with a buy token of ETH or WETH, intended slippage protection can be lost, resulting in loss of user funds.

Example:

_executeTrade is called with trade.sellToken = USDC and trade.buyToken = WETH. sellToken is USDC so L127 and L132 return false and preTradeBalance = 0. Assuming the user has a balance of ETH from before the trade. postTradeBalance now returns their ETH balance. Since preTradeBalance = 0, their full balance will be deposited into WETH. When _getBalances queries their WETH balance it will count the deposited ETH balance as value obtained from the swap, even though it was just their own balance deposited. 

Dev comments in ZeroExAdapter state: "executeTrade validates pre and post trade balances and also sets and revokes all approvals. We are also only calling a trusted zero ex proxy in this case. Therefore no order validation is done to allow for flexibility." By design, when using ZeroExAdapter slippage values are not enforces as they are with other adapters, leaving validation of slippage exclusively to _executeTrade. As shown above, _executeTrade slippage checks can fail leaving ZeroEx swaps completely vulnerable to sandwich attacks which could cause loss of funds for users

## Impact

Loss of user funds or valid transactions reverting

## Code Snippet

[TradingUtils.sol#L118-L160](https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradingUtils.sol#L118-L160)

## Tool used

Manual Review

## Recommendation

If buyToken is ETH/WETH the preTradeBalance should be logged as shown below:

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
    ++  } else if (trade.buyToken == address(Deployments.WETH) {
    ++      preTradeBalance = address(this).balance;
    ++  } else if (trade.buyToken == Deployments.ETH_ADDRESS) {
    ++      preTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
        }
    