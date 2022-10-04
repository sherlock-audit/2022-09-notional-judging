cccz
# When tokenType != Ether, need to check msg.value == 0

## Summary
When tokenType != Ether, need to check msg.value == 0
## Vulnerability Detail
In the depositForRollPosition function of the VaultAccountLib library and the transferUnderlyingToVaultDirect and _redeem functions of the VaultConfiguration library, there is no check for msg.value == 0 when tokenType != Ether. If the user mistakenly sends ETH to the contract when tokenType != Ether, the ETH will be locked in the contract.
## Impact
If the user mistakenly sends ETH to the contract when tokenType != Ether, the ETH will be locked in the contract.
## Code Snippet
```solidity
        if (underlyingToken.tokenType == TokenType.Ether) {
            require(depositAmountExternal == msg.value, "Invalid ETH");
            amountTransferred = msg.value;
        } else {
            amountTransferred = underlyingToken.transfer(
                vaultAccount.account, vaultConfig.borrowCurrencyId, depositAmountExternal.toInt()
            ).toUint();
        }
...
        if (underlyingToken.tokenType == TokenType.Ether) {
            require(msg.value == depositAmountExternal, "Invalid ETH");
            // Forward all the ETH to the vault
            GenericToken.transferNativeTokenOut(vault, msg.value);

            return msg.value;
        } else {
            GenericToken.safeTransferFrom(underlyingToken.tokenAddress, transferFrom, vault, depositAmountExternal);
            return depositAmountExternal;
        }
...
            if (underlyingToken.tokenType == TokenType.Ether) {
                require(residualRequired <= msg.value, "Insufficient repayment");
                // Transfer out the unused part of msg.value, we've received all underlying external required
                // at this point
                GenericToken.transferNativeTokenOut(params.account, msg.value - residualRequired);
                amountTransferred = underlyingExternalToRepay;
            } else {
                // actualTransferExternal is a positive number here to signify assets have entered
                // the protocol
                int256 actualTransferExternal = underlyingToken.transfer(
                    params.account, vaultConfig.borrowCurrencyId, residualRequired.toInt()
                );
                amountTransferred = amountTransferred.add(actualTransferExternal.toUint());
            }
```
## Tool used

Manual Review

## Recommendation
```diff
        if (underlyingToken.tokenType == TokenType.Ether) {
            require(depositAmountExternal == msg.value, "Invalid ETH");
            amountTransferred = msg.value;
        } else {
+          require(msg.value == 0);
            amountTransferred = underlyingToken.transfer(
                vaultAccount.account, vaultConfig.borrowCurrencyId, depositAmountExternal.toInt()
            ).toUint();
        }
...
        if (underlyingToken.tokenType == TokenType.Ether) {
            require(msg.value == depositAmountExternal, "Invalid ETH");
            // Forward all the ETH to the vault
            GenericToken.transferNativeTokenOut(vault, msg.value);

            return msg.value;
        } else {
+          require(msg.value == 0);
            GenericToken.safeTransferFrom(underlyingToken.tokenAddress, transferFrom, vault, depositAmountExternal);
            return depositAmountExternal;
        }
...
            if (underlyingToken.tokenType == TokenType.Ether) {
                require(residualRequired <= msg.value, "Insufficient repayment");
                // Transfer out the unused part of msg.value, we've received all underlying external required
                // at this point
                GenericToken.transferNativeTokenOut(params.account, msg.value - residualRequired);
                amountTransferred = underlyingExternalToRepay;
            } else {
+             require(msg.value == 0);
                // actualTransferExternal is a positive number here to signify assets have entered
                // the protocol
                int256 actualTransferExternal = underlyingToken.transfer(
                    params.account, vaultConfig.borrowCurrencyId, residualRequired.toInt()
                );
                amountTransferred = amountTransferred.add(actualTransferExternal.toUint());
            }
```