Bnke0x0
# Malicious governance can use updateVault()/updateSecondaryBorrowCapacity() to steal WETH from buyers

## Summary
A malicious or compromised governance can set the transfer gas cost to an unreasonable amount and steal approved WETH from buyers.

## Vulnerability Detail

## Impact
A malicious or compromised governance can set the transfer gas cost to an unreasonable amount and steal approved WETH from buyers.

## Code Snippet
1. https://github.com/None/blob/None/contracts-v2/contracts/external/actions/VaultAction.sol#L27-L36

               'function updateVault(
                        address vaultAddress,
                        VaultConfigStorage calldata vaultConfig,
                        uint80 maxPrimaryBorrowCapacity
                       ) external override onlyOwner {
                        VaultConfiguration.setVaultConfig(vaultAddress, vaultConfig);
                        VaultConfiguration.setMaxBorrowCapacity(vaultAddress, vaultConfig.borrowCurrencyId, maxPrimaryBorrowCapacity);
                        bool enabled = (vaultConfig.flags & VaultConfiguration.ENABLED) == VaultConfiguration.ENABLED;
                        emit VaultUpdated(vaultAddress, enabled, maxPrimaryBorrowCapacity);
                              }'

1. https://github.com/None/blob/None/contracts-v2/contracts/external/actions/VaultAction.sol#L57-L79

               'function updateSecondaryBorrowCapacity(
                        address vaultAddress,
                        uint16 secondaryCurrencyId,
                        uint80 maxBorrowCapacity
                        ) external override onlyOwner {
                        VaultConfig memory vaultConfig = VaultConfiguration.getVaultConfigStateful(vaultAddress);
                        // Tokens with transfer fees create lots of issues with vault mechanics, we prevent them
                        // from being listed here.
                        Token memory assetToken = TokenHandler.getAssetToken(secondaryCurrencyId);
                        Token memory underlyingToken = TokenHandler.getUnderlyingToken(secondaryCurrencyId);
                        require(!assetToken.hasTransferFee && !underlyingToken.hasTransferFee); 

                        // The secondary borrow currency must be white listed on the configuration before we can set a max
                        // capacity.
                        require(
                        secondaryCurrencyId == vaultConfig.secondaryBorrowCurrencies[0] ||
                        secondaryCurrencyId == vaultConfig.secondaryBorrowCurrencies[1],
                        "Invalid Currency"
                        );

                        VaultConfiguration.setMaxBorrowCapacity(vaultAddress, secondaryCurrencyId, maxBorrowCapacity);
                        emit VaultUpdateSecondaryBorrowCapacity(vaultAddress, secondaryCurrencyId, maxBorrowCapacity);

                                       }'

## Tool used
Manual Review

## Recommendation
Set a sanity check in updateVault()/updateSecondaryBorrowCapacity() so governance can't set it to unreasonable value. Consider using timelock for setting governance settings.