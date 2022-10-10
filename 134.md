GimelSec

medium

# `VaultAccountStorage.maturity` is uint32, which will break in 2106

## Summary

`VaultAccountStorage.maturity` stores the maturity when the vault shares and fCash will mature. But it only stores `uint32` which will break in 2106 due to the uint32.max is 4294967295.

## Vulnerability Detail

Solidity docs defined that `block.timestamp` is `uint256`, but `VaultAccountStorage.maturity` is `uint32`. The uint32.max is 4294967295 which is `Sun Feb 07 2106`, it means that the protocol will break when the maturity is greater than 2106.

https://ethereum.stackexchange.com/questions/28969/what-uint-type-should-be-declared-for-unix-timestamps

## Impact

The protocol will break if maturity is greater than 2106.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/global/Types.sol#L572
https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/global/Types.sol#L580

## Tool used

Manual Review

## Recommendation

Define as uint256 or larger bits than 32.
