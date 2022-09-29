Waze
# Use call instead of transfer in transferNativeTokenOut().

## Summary
Transfer need to be check for value indicating success.
## Vulnerability Detail
Use call instead of transfer to transfer native token, and return value must be checked if transfer native token is successful or not. Transfer native token with transfer is no longer recommended.
## Impact
to avoid transfer failed but return false instead. token thats dont actually perform the transfer and return false are till counted as a correct transfer.
## Code Snippet
https://github.com/None/blob/None/contracts-v2/contracts/internal/balances/protocols/GenericToken.sol#L29
## Tool used

Manual Review

## Recommendation
add this :
(bool success, ) = account.call{value: amount}("");
require(success,"failed to transfer Token");