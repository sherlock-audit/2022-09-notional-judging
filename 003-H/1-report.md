Bnke0x0
# underlyingToken can be stuck into the Strategy contract


## Vulnerability Detail

## Impact
At every '_redeem()'/'repaySecondaryCurrencyFromVault()' it checks the balance before the claim and after, to calculate the auraBAL earned, so every underlyingToken  transferred to the strategy address, not during this call, won't be swapped to Notional.

## Code Snippet
1. https://github.com/None/blob/None/contracts-v2/contracts/external/actions/VaultAction.sol#L329-L337

             'uint256 balanceBefore = underlyingToken.balanceOf(address(this));
            // Tells the vault will redeem the strategy token amount and transfer asset tokens back to Notional
            returnData = IStrategyVault(msg.sender).repaySecondaryBorrowCallback(
                underlyingToken.tokenAddress, underlyingExternalToRepay, callbackData
            );
            uint256 balanceAfter = underlyingToken.balanceOf(address(this));
            balanceTransferred = balanceAfter.sub(balanceBefore);
            require(balanceTransferred >= underlyingExternalToRepay, "Insufficient Repay");
        }'

2. https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L600-L624

               'uint256 amountTransferred;
        if (params.strategyTokens > 0) {
            uint256 balanceBefore = underlyingToken.balanceOf(address(this));
            // There are four possibilities here during the transfer:
            //   1. If the account == vaultConfig.vault then the strategy vault must always transfer
            //      tokens back to Notional. underlyingToReceiver will equal 0, amountTransferred will
            //      be the value of the redemption.
            //   2. If the account has debt to repay and is redeeming sufficient tokens to repay the debt,
            //      the vault will transfer back underlyingExternalToRepay and transfer underlyingToReceiver
            //      directly to the receiver.
            //   3. If the account has redeemed insufficient tokens to repay the debt, the vault will transfer
            //      back as much as it can (less than underlyingExternalToRepay) and underlyingToReceiver will
            //      be zero. If this occurs, then the next if block will be triggered where we attempt to recover
            //      the shortfall from the account's wallet.
            //   4. During liquidation, the liquidator will redeem their strategy token profits without any debt
            //      to repay (underlyingExternalToRepay == 0). This means that all the profits will be returned
            //      to the liquidator (params.receiver) from the vault (underlyingToReceiver will be the full value
            //      of the redemption) and amountTransferred will equal 0. A similar scenario will occur when
            //      accounts exit post maturity and have no debt associated with their account.
            underlyingToReceiver = IStrategyVault(vaultConfig.vault).redeemFromNotional(
                params.account, params.receiver, params.strategyTokens, params.maturity, underlyingExternalToRepay, data
            );
            uint256 balanceAfter = underlyingToken.balanceOf(address(this));
            amountTransferred = balanceAfter.sub(balanceBefore);
        }'

## Tool used

Manual Review

## Recommendation
Instead of calculating the balance before and after the claim
