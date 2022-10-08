xiaoming90

medium

# Did Not Approve To Zero First

## Summary

Allowance was not set to zero first before changing the allowance.

## Vulnerability Detail

Some ERC20 tokens (like USDT) do not work when changing the allowance from an existing non-zero allowance value. For example Tether (USDT)'s `approve()` function will revert if the current approval is not zero, to protect against front-running changes of approvals.

The following  attempt to call the `approve()` function without setting the allowance to zero first.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L18

```solidity
File: TokenUtils.sol
18:     function checkApprove(IERC20 token, address spender, uint256 amount) internal {
19:         if (address(token) == address(0)) return;
20: 
21:         IEIP20NonStandard(address(token)).approve(spender, amount);
22:         _checkReturnCode();
23:     }
```

However, if the token involved is an ERC20 token that does not work when changing the allowance from an existing non-zero allowance value, it will break a number of key functions or features of the protocol as the `TokenUtils.checkApprove` function is utilised extensively within the vault as shown below.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L159

```solidity
File: TwoTokenPoolUtils.sol
157:     function _approveBalancerTokens(TwoTokenPoolContext memory poolContext, address bptSpender) internal {
158:         IERC20(poolContext.primaryToken).checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
159:         IERC20(poolContext.secondaryToken).checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
160:         // Allow BPT spender to pull BALANCER_POOL_TOKEN
161:         IERC20(address(poolContext.basePool.pool)).checkApprove(bptSpender, type(uint256).max);
162:     }
```

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/Boosted3TokenPoolUtils.sol#L225

```solidity
File: Boosted3TokenPoolUtils.sol
222:     function _approveBalancerTokens(ThreeTokenPoolContext memory poolContext, address bptSpender) internal {
223:         poolContext.basePool._approveBalancerTokens(bptSpender);
224: 
225:         IERC20(poolContext.tertiaryToken).checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
226: 
227:         // For boosted pools, the tokens inside pool context are AaveLinearPool tokens.
228:         // So, we need to approve the _underlyingToken (primary borrow currency) for trading.
229:         IBoostedPool underlyingPool = IBoostedPool(poolContext.basePool.primaryToken);
230:         address primaryUnderlyingAddress = BalancerUtils.getTokenAddress(underlyingPool.getMainToken());
231:         IERC20(primaryUnderlyingAddress).checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
232:     }
```

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L115

```solidity
File: TradingUtils.sol
110:     /// @notice Approve exchange to pull from this contract
111:     /// @dev approve up to trade.amount for EXACT_IN trades and up to trade.limit
112:     /// for EXACT_OUT trades
113:     function _approve(Trade memory trade, address spender) private {
114:         uint256 allowance = _isExactIn(trade) ? trade.amount : trade.limit;
115:         IERC20(trade.sellToken).checkApprove(spender, allowance);
116:     }
```

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/strategy/StrategyUtils.sol#L85

```solidity
File: StrategyUtils.sol
85:             IERC20(buyToken).checkApprove(address(Deployments.WRAPPED_STETH), amountBought);
86:             uint256 wrappedAmount = Deployments.WRAPPED_STETH.balanceOf(address(this));
87:             /// @notice the amount returned by wrap is not always accurate for some reason
88:             Deployments.WRAPPED_STETH.wrap(amountBought);
89:             amountBought = Deployments.WRAPPED_STETH.balanceOf(address(this)) - wrappedAmount;
```

## Impact

A number of features within the vaults will not work if the `approve` function reverts.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L18
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L159
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/Boosted3TokenPoolUtils.sol#L225
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L115
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/strategy/StrategyUtils.sol#L85

## Tool used

Manual Review

## Recommendation

It is recommended to set the allowance to zero before increasing the allowance and use safeApprove/safeIncreaseAllowance.