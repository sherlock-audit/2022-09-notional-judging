Waze
# Low-level delegatecall function may not revert as it fails.

## Summary
The low-level delegatecall will not notice anything went wrong. This may result in user funds lost because funds were transferred. Low-level delegatecall fails but doesn't revert.
## Vulnerability Detail
It is stated in the solidity documentation: https://docs.soliditylang.org/en/v0.8.16/control-structures.html#error-handling-assert-require-revert-and-exceptions.
In the warning part, it is written as: "The low-level functions call, delegatecall, and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed."
## Impact
The low-level function "delegatecall" may return true for the boolean parameter "success" even if it was a failure. This could result in fund loss.
## Code Snippet
https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradeHandler.sol#L19-L22
https://github.com/None/blob/None/leveraged-vaults/contracts/trading/TradeHandler.sol#L36-L37
## Tool used

Manual Review

## Recommendation
Checking the address before the delegatecall occurs