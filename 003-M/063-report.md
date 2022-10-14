xiaoming90

medium

# `Transfer()` Is Used Instead If `Call()` To Transfer ETH

## Summary

`Transfer()` is used instead of `Call()` to transfer ETH which may break the code under some conditions.

## Vulnerability Detail

The use of .transfer() or .send() to send ether is now considered bad practice as gas costs can change, which would break the code.

The transaction will fail inevitably when:

- Target smart contract does not implement a payable function.
- Target smart contract does implement a payable fallback which uses more than 2300 gas unit.
- Target smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call’s gas usage above 2300.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/BaseStrategyVault.sol#L180

```solidity
File: BaseStrategyVault.sol
180:         if (_UNDERLYING_IS_ETH) {
181:             if (transferToReceiver > 0) payable(receiver).transfer(transferToReceiver);
182:             if (transferToNotional > 0) payable(address(NOTIONAL)).transfer(transferToNotional);
183:         } else {
184:             if (transferToReceiver > 0) _UNDERLYING_TOKEN.checkTransfer(receiver, transferToReceiver);
185:             if (transferToNotional > 0) _UNDERLYING_TOKEN.checkTransfer(address(NOTIONAL), transferToNotional);
186:         }
```

## Impact

Assets will fail to transfer to Notional or vault shareholders.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/BaseStrategyVault.sol#L180

## Tool used

Manual Review

## Recommendation

Use call instead of transfer to send ether. And return value must be checked if sending ether is successful or not. make sure to check for reentrancy.