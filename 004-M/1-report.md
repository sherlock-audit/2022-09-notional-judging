Bnke0x0
# User's may accidentally overpay in depositVaultCashToStrategyTokens()/repaySecondaryCurrencyFromVault()/_redeem() and the excess will be paid to the vault creator

## Vulnerability Detail

## Impact
It is possible for a user purchasing an option to accidentally overpay the premium during `depositVaultCashToStrategyTokens()/repaySecondaryCurrencyFromVault()/_redeem() `.

Any excess funds paid for in excess of the premium will be transferred to the vault creator.

## Code Snippet

1. https://github.com/None/blob/None/contracts-v2/contracts/external/actions/VaultAction.sol#L191

            'require(vaultConfig.minCollateralRatio <= collateralRatio, "Insufficient Collateral");'

2. https://github.com/None/blob/None/contracts-v2/contracts/external/actions/VaultAction.sol#L336

            ' require(balanceTransferred >= underlyingExternalToRepay, "Insufficient Repay");'

3. https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L635

            'require(residualRequired <= msg.value, "Insufficient repayment");'

4. https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L651

            ' require(amountTransferred >= underlyingExternalToRepay, "Insufficient repayment");'

## Tool used

Manual Review

## Recommendation
Consider changing '>=' to '=='
