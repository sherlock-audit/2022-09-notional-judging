0x52
# TradingUtils#_approve is problematic for tokens like USDT that requires allowance to be zero before calling approve

## Summary

Exact out swaps will break the ability to swap any ERC20 tokens that require allowance to be zero before calling approve, like USDT. After an exact out swap, there will be residual allowance left for spender which will cause all subsequent trades to the spender to revert.

## Vulnerability Detail

[TradingUtils.sol#L113-L116](https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradingUtils.sol#L113-L116)

    function _approve(Trade memory trade, address spender) private {
        uint256 allowance = _isExactIn(trade) ? trade.amount : trade.limit;
        IERC20(trade.sellToken).checkApprove(spender, allowance);
    }

_approve is called each time _executeInternal is called which is subsequently called during each swap. For exact in swaps, the lines above will never be an issue since the entire allowance will always be used up. For exact out swaps the above lines approve the spender up to the trade limit. In a majority of cases it won't use the full allowance leaving a residual allowance after the trade is completed. Some ERC20 tokens, notably including Tether, require that the current allowance for the spender to be zero or else the approval call will revert. The result is that subsequent swaps will revert for that token after an exact out swap.

## Impact

All subsequent swaps for the affected token to the same spender will revert

## Code Snippet

[TokenUtils.sol#L18-L23](https://github.com/None/blob/None/leveraged-vaults/contracts/utils/TokenUtils.sol#L18-L23)

## Tool used

Manual Review

## Recommendation

Change TokenUtils#checkApprove to first reset allowance to 0 then set to desired allowance, which will resolve incompatibilities with the tokens described above:

    function checkApprove(IERC20 token, address spender, uint256 amount) internal {
        if (address(token) == address(0)) return;

    ++  IEIP20NonStandard(address(token)).approve(spender, 0);
        IEIP20NonStandard(address(token)).approve(spender, amount);

        _checkReturnCode();
    }