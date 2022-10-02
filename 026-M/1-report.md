Waze
# Unsafe transfer when using transferNativeTokenOut() from GenericToken Lib can result in revert.

## Summary
Unsafe transfer happen when using transferNativeTokenOut(). Transfer need to be check for value indicating success. 
## Vulnerability Detail
To transfer native token, the return value must be checked if transfer native token is successful or not. Transfer native token with transfer is no longer recommended. transferNativeTokenOut in GenericToken Lib doesn't accommodate that. 
## Impact
to avoid transfer failed but return false instead. token thats dont actually perform the transfer and return false are till counted as a correct transfer. 
## Code Snippet
https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L454

https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L638
## Tool used

Manual Review

## Recommendation
implement check value in transferNativeTokenOut() from GenericToken lib using call() instead of transfer().
Example:
(bool succeeded, ) = account.call{value: _amount}("");
require(succeeded, "Transfer failed.");