0xSmartContract
# Low-level transfer via call() can fail silently

## Summary

[TradingUtils.sol#L139](https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradingUtils.sol#L139)


## Vulnerability Detail

Solidity docs:"The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed."

Therefore, transfers may fail silently.

in the `TradingUtils.sol` a call is executed with the following code in `_executeTrade` function

`call` function on line 139
```js
  118:     function _executeTrade(
  119:         address target,
  120:         uint256 msgValue,
  121:         bytes memory params,
  122:         address spender,
  123:         Trade memory trade
  124:     ) private {
  125:         uint256 preTradeBalance;
  126: 
  127:         if (trade.sellToken == address(Deployments.WETH) && spender == Deployments.ETH_ADDRESS) {
  128:             preTradeBalance = address(this).balance;
  129:             // Curve doesn't support Deployments.WETH (spender == address(0))
  130:             uint256 withdrawAmount = _isExactIn(trade) ? trade.amount : trade.limit;
  131:             Deployments.WETH.withdraw(withdrawAmount);
  132:         } else if (trade.sellToken == Deployments.ETH_ADDRESS && spender != Deployments.ETH_ADDRESS) {
  133:             preTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
  134:             // UniswapV3 doesn't support ETH (spender != address(0))
  135:             uint256 depositAmount = _isExactIn(trade) ? trade.amount : trade.limit;
  136:             Deployments.WETH.deposit{value: depositAmount }();
  137:         }
  138: 
  139:         (bool success, bytes memory returnData) = target.call{value: msgValue}(params);
  140:         if (!success) revert TradeExecution(returnData);
  141: 
  142:         if (trade.buyToken == address(Deployments.WETH)) {
  143:             if (address(this).balance > preTradeBalance) {
  144:                 // If the caller specifies that they want to receive Deployments.WETH but we have received ETH,
  145:                 // wrap the ETH to Deployments.WETH.
  146:                 uint256 depositAmount;
  147:                 unchecked { depositAmount = address(this).balance - preTradeBalance; }
  148:                 Deployments.WETH.deposit{value: depositAmount}();
  149:             }
  150:         } else if (trade.buyToken == Deployments.ETH_ADDRESS) {
  151:             uint256 postTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
  152:             if (postTradeBalance > preTradeBalance) {
  153:                 // If the caller specifies that they want to receive ETH but we have received Deployments.WETH,
  154:                 // unwrap the Deployments.WETH to ETH.
  155:                 uint256 withdrawAmount;
  156:                 unchecked { withdrawAmount = postTradeBalance - preTradeBalance; }
  157:                 Deployments.WETH.withdraw(withdrawAmount);
  158:             }
  159:         }
  160:     }
  161  
```

## Tool used

Manual Review

## Recommendation

Check for the account's existence prior to low level call
call is not recommended in most situations for contract function calls because it bypasses type checking, function existence check, and argument packing. It is preferred to import the interface of the contract to call functions on it.

