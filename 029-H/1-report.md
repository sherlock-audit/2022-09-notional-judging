Waze
# safetransferfrom doesn't check the codesize of the token address, which may lead to fund loss.

## Summary
safetransferfrom doesn't check the existence of code at the token address. This is a known issue while using GenericToken libraries.
## Vulnerability Detail
This vulnerability  may lead to miscalculation of funds and may lead to loss of funds , because if  safetransferfrom() are called on a token address that doesn't have contract in it, it will always return success, bypassing the return value check. Due to this protocol will think that funds has been transferred and successful , and records will be accordingly calculated, but in reality funds were never transferred. So this will lead to miscalculation and possibly loss of funds.
## Impact
lead to miscalculation and possibly loss of funds due to safetransferfrom doesn't check the existence of code at the token address.
## Code Snippet
https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L458
## Tool used

Manual Review

## Recommendation
implement a code existence check at the token address.