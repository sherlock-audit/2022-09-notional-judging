xiaoming90

medium

# All Borrowed Assets Can Be Charged As Protocol Fees Due To Uncapped Fee

## Summary

All borrowed assets can be charged as protocol fees due to uncapped fees.

## Vulnerability Detail

Whenever a user borrows fCash to enter a vault, they have to pay a fee to Notional. The fee is charged against the total amount of assets borrowed from Notional.

https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L259

```solidity
File: VaultConfiguration.sol
252:     /// @notice Assess fees to the vault account. The fee based on time to maturity and the amount of fCash. Fees
253:     /// will be accrued to the nToken cash balance and the protocol cash reserve.
254:     /// @param vaultConfig vault configuration
255:     /// @param vaultAccount modifies the vault account temp cash balance in memory
256:     /// @param assetCashBorrowed the amount of cash the account has borrowed
257:     /// @param maturity maturity of fCash
258:     /// @param blockTime current block time
259:     function assessVaultFees(
260:         VaultConfig memory vaultConfig,
261:         VaultAccount memory vaultAccount,
262:         int256 assetCashBorrowed,
263:         uint256 maturity,
264:         uint256 blockTime
265:     ) internal {
266:         // The fee rate is annualized, we prorate it linearly based on the time to maturity here
267:         int256 proratedFeeRate = vaultConfig.feeRate
268:             .mul(maturity.sub(blockTime).toInt())
269:             .div(int256(Constants.YEAR));
270: 
271:         int256 netTotalFee = assetCashBorrowed.mulInRatePrecision(proratedFeeRate);
272: 
273:         // Reserve fee share is restricted to less than 100
274:         int256 reserveFee = netTotalFee.mul(vaultConfig.reserveFeeShare).div(Constants.PERCENTAGE_DECIMALS);
275:         int256 nTokenFee = netTotalFee.sub(reserveFee);
276: 
277:         BalanceHandler.incrementFeeToReserve(vaultConfig.borrowCurrencyId, reserveFee);
278:         BalanceHandler.incrementVaultFeeToNToken(vaultConfig.borrowCurrencyId, nTokenFee);
279: 
280:         vaultAccount.tempCashBalance = vaultAccount.tempCashBalance.sub(netTotalFee);
281:         emit VaultFeeAccrued(vaultConfig.vault, vaultConfig.borrowCurrencyId, maturity, reserveFee, nTokenFee);
282:     }
```

Per the `Types` contract below, it was understood that the `feeRate5BPS` is allowed up to a 12.75% annualized fee only.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/global/Types.sol#L461

```solidity
File: Types.sol
461: struct VaultConfigStorage {
462:     // Vault Flags (documented in VaultConfiguration.sol)
463:     uint16 flags;
464:     // Primary currency the vault borrows in
465:     uint16 borrowCurrencyId;
466:     // Specified in whole tokens in 1e8 precision, allows a 4.2 billion min borrow size
467:     uint32 minAccountBorrowSize;
468:     // Minimum collateral ratio for a vault specified in basis points, valid values are greater than 10_000
469:     // where the largest minimum collateral ratio is 65_536 which is much higher than anything reasonable.
470:     uint16 minCollateralRatioBPS;
471:     // Allows up to a 12.75% annualized fee
472:     uint8 feeRate5BPS;
473:     // A percentage that represents the share of the cash raised that will go to the liquidator
474:     uint8 liquidationRate;
475:     // A percentage of the fee given to the protocol
476:     uint8 reserveFeeShare;
477:     // Maximum market index where a vault can borrow from
478:     uint8 maxBorrowMarketIndex;
479:     // Maximum collateral ratio that a liquidator can push a an account to during deleveraging
480:     uint16 maxDeleverageCollateralRatioBPS;
481:     // An optional list of secondary borrow currencies
482:     uint16[2] secondaryBorrowCurrencies;
483:     // 96 bytes left
484: }
```

The configuration of the vault is set by calling the `VaultAction.updateVault` function. Following is an example taken from the test script.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/tests/test_cross_currency.py#L13

```python
File: test_cross_currency.py
12: @pytest.fixture(scope="module", autouse=True)
13: def usdcDaiVault(env, CrossCurrencyfCashVault, nProxy, accounts):
14:     impl = CrossCurrencyfCashVault.deploy(env.notional.address, env.tradingModule.address, {"from": accounts[0]})
15:     initializeCallData = impl.initialize.encode_input(
16:         "USDC/DAI Cross Currency fCash",
17:         3, 2, 0.995e18
18:     )
19:     proxy = nProxy.deploy(impl.address, initializeCallData, {"from": accounts[0]})
20:     vault = Contract.from_abi("CrossCurrency", proxy.address, abi=CrossCurrencyfCashVault.abi)
21: 
22:     env.notional.updateVault(
23:         vault.address,
24:         get_vault_config(
25:             flags=set_flags(0, ENABLED=True, ONLY_VAULT_SETTLE=True, ALLOW_REENTRANCY=True),
26:             currencyId=3,
27:             feeRate5BPS=1, # 5 BPS fee
28:             minCollateralRatioBPS=500, # 5% collateral ratio
29:             maxBorrowMarketIndex=3 # allow up to 1 year borrows
30:         ),
31:         100_000_000e8,
32:         {"from": env.notional.owner()},
33:     )
34: 
35:     env.tokens['USDC'].transfer(accounts[0], 30_000e6, {"from": env.whales['USDC']})
36:     env.tokens['USDC'].approve(env.notional.address, 2 ** 255, {"from": accounts[0]})
37: 
38:     return vault
```

When the `VaultAction.updateVault` function is triggered, it will in turn call the `VaultConfiguration.setVaultConfig` function.

https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/external/actions/VaultAction.sol#L27

```solidity
File: VaultAction.sol
23:     /// @notice Updates or lists a deployed vault along with its configuration.
24:     /// @param vaultAddress address of deployed vault
25:     /// @param vaultConfig struct of vault configuration
26:     /// @param maxPrimaryBorrowCapacity maximum borrow capacity
27:     function updateVault(
28:         address vaultAddress,
29:         VaultConfigStorage calldata vaultConfig,
30:         uint80 maxPrimaryBorrowCapacity
31:     ) external override onlyOwner {
32:         VaultConfiguration.setVaultConfig(vaultAddress, vaultConfig);
33:         VaultConfiguration.setMaxBorrowCapacity(vaultAddress, vaultConfig.borrowCurrencyId, maxPrimaryBorrowCapacity);
34:         bool enabled = (vaultConfig.flags & VaultConfiguration.ENABLED) == VaultConfiguration.ENABLED;
35:         emit VaultUpdated(vaultAddress, enabled, maxPrimaryBorrowCapacity);
36:     }
```

Within the `VaultConfiguration.setVaultConfig` function, it will perform various input validation checks to ensure that the configuration is appropriate. However, it was observed that the function does not perform any validation check against the `vaultConfig.feeRate5BPS` parameter. Therefore, the fee is unbounded, and it is possible to set it to an extremely high value causing most or all of the borrowed assets to be charged as fee.

https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L163

```solidity
File: VaultConfiguration.sol
163:     function setVaultConfig(
164:         address vaultAddress,
165:         VaultConfigStorage calldata vaultConfig
166:     ) internal {
167:         mapping(address => VaultConfigStorage) storage store = LibStorage.getVaultConfig();
168:         VaultConfig memory existingVaultConfig = _getVaultConfig(vaultAddress);
169:         // Cannot change borrow currency once set
170:         require(vaultConfig.borrowCurrencyId != 0);
171:         require(existingVaultConfig.borrowCurrencyId == 0 || existingVaultConfig.borrowCurrencyId == vaultConfig.borrowCurrencyId);
172: 
173:         // Liquidation rate must be greater than or equal to 100
174:         require(Constants.PERCENTAGE_DECIMALS <= vaultConfig.liquidationRate);
175:         // This must be true or else when deleveraging we could put an account further towards insolvency
176:         require(vaultConfig.minCollateralRatioBPS < vaultConfig.maxDeleverageCollateralRatioBPS);
177:         // minCollateralRatioBPS to RATE_PRECISION is minCollateralRatioBPS * BASIS_POINT (1e5)
178:         // liquidationRate to RATE_PRECISION  is liquidationRate * RATE_PRECISION / PERCENTAGE_DECIMALS (net 1e7)
179:         //    (liquidationRate - 100) * 1e9 / 1e2 < minCollateralRatioBPS * 1e5
180:         //    (liquidationRate - 100) * 1e2 < minCollateralRatioBPS
181:         uint16 liquidationRate = uint16(
182:             uint256(vaultConfig.liquidationRate - uint256(Constants.PERCENTAGE_DECIMALS)) * uint256(1e2)
183:         );
184:         // Ensure that liquidation rate is less than minCollateralRatio so that liquidations are not locked
185:         // up causing accounts to remain insolvent
186:         require(liquidationRate < vaultConfig.minCollateralRatioBPS);
187: 
188:         // Reserve fee share must be less than or equal to 100
189:         require(vaultConfig.reserveFeeShare <= Constants.PERCENTAGE_DECIMALS);
190:         require(vaultConfig.maxBorrowMarketIndex != 0);
191: 
192:         // Secondary borrow currencies cannot change once set
193:         require(
194:             existingVaultConfig.secondaryBorrowCurrencies[0] == 0 ||
195:             existingVaultConfig.secondaryBorrowCurrencies[0] == vaultConfig.secondaryBorrowCurrencies[0]
196:         );
197:         require(
198:             existingVaultConfig.secondaryBorrowCurrencies[1] == 0 ||
199:             existingVaultConfig.secondaryBorrowCurrencies[1] == vaultConfig.secondaryBorrowCurrencies[1]
200:         );
201: 
202:         // The borrow currency cannot be duplicated as a secondary borrow currency
203:         require(vaultConfig.borrowCurrencyId != vaultConfig.secondaryBorrowCurrencies[0]);
204:         require(vaultConfig.borrowCurrencyId != vaultConfig.secondaryBorrowCurrencies[1]);
205:         if (vaultConfig.secondaryBorrowCurrencies[0] != 0 && vaultConfig.secondaryBorrowCurrencies[1] != 0) {
206:             // Check that these values are not duplicated if set
207:             require(vaultConfig.secondaryBorrowCurrencies[0] != vaultConfig.secondaryBorrowCurrencies[1]);
208:         }
209: 
210:         // Tokens with transfer fees create lots of issues with vault mechanics, we prevent them
211:         // from being listed here.
212:         Token memory assetToken = TokenHandler.getAssetToken(vaultConfig.borrowCurrencyId);
213:         Token memory underlyingToken = TokenHandler.getUnderlyingToken(vaultConfig.borrowCurrencyId);
214:         require(!assetToken.hasTransferFee && !underlyingToken.hasTransferFee); 
215: 
216:         store[vaultAddress] = vaultConfig;
217:     }
```

## Impact

The entire borrowed assets can be charged as a protocol fee due to the lack of an upper bound on the fee. Loss of assets for the users as all or majority of their borrowed assets went to the protocol as a fee.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L259
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/global/Types.sol#L461
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/tests/test_cross_currency.py#L13
https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/external/actions/VaultAction.sol#L27
https://github.com/sherlock-audit/2022-09-notional/blob/main/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L163

## Tool used

Manual Review

## Recommendation

It is recommended to set an absolute cap on the maximum fee that can be charged.

Since the requirement is to allow up to a 12.75% annualized fee as per the comment within the `Types` contract below (See Line 471), consider including a validation check within the `VaultConfiguration.setVaultConfig` function to ensure that the fee falls within the range (fee <= 12.75% annualized fee).

```solidity
File: Types.sol
461: struct VaultConfigStorage {
462:     // Vault Flags (documented in VaultConfiguration.sol)
463:     uint16 flags;
464:     // Primary currency the vault borrows in
465:     uint16 borrowCurrencyId;
466:     // Specified in whole tokens in 1e8 precision, allows a 4.2 billion min borrow size
467:     uint32 minAccountBorrowSize;
468:     // Minimum collateral ratio for a vault specified in basis points, valid values are greater than 10_000
469:     // where the largest minimum collateral ratio is 65_536 which is much higher than anything reasonable.
470:     uint16 minCollateralRatioBPS;
471:     // Allows up to a 12.75% annualized fee
472:     uint8 feeRate5BPS;
473:     // A percentage that represents the share of the cash raised that will go to the liquidator
474:     uint8 liquidationRate;
475:     // A percentage of the fee given to the protocol
476:     uint8 reserveFeeShare;
477:     // Maximum market index where a vault can borrow from
478:     uint8 maxBorrowMarketIndex;
479:     // Maximum collateral ratio that a liquidator can push a an account to during deleveraging
480:     uint16 maxDeleverageCollateralRatioBPS;
481:     // An optional list of secondary borrow currencies
482:     uint16[2] secondaryBorrowCurrencies;
483:     // 96 bytes left
484: }
```

```diff
function setVaultConfig(
    address vaultAddress,
    VaultConfigStorage calldata vaultConfig
) internal {
    mapping(address => VaultConfigStorage) storage store = LibStorage.getVaultConfig();
    VaultConfig memory existingVaultConfig = _getVaultConfig(vaultAddress);
    // Cannot change borrow currency once set
    require(vaultConfig.borrowCurrencyId != 0);
    require(existingVaultConfig.borrowCurrencyId == 0 || existingVaultConfig.borrowCurrencyId == vaultConfig.borrowCurrencyId);

    // Liquidation rate must be greater than or equal to 100
    require(Constants.PERCENTAGE_DECIMALS <= vaultConfig.liquidationRate);
    // This must be true or else when deleveraging we could put an account further towards insolvency
    require(vaultConfig.minCollateralRatioBPS < vaultConfig.maxDeleverageCollateralRatioBPS);
    // minCollateralRatioBPS to RATE_PRECISION is minCollateralRatioBPS * BASIS_POINT (1e5)
    // liquidationRate to RATE_PRECISION  is liquidationRate * RATE_PRECISION / PERCENTAGE_DECIMALS (net 1e7)
    //    (liquidationRate - 100) * 1e9 / 1e2 < minCollateralRatioBPS * 1e5
    //    (liquidationRate - 100) * 1e2 < minCollateralRatioBPS
    uint16 liquidationRate = uint16(
        uint256(vaultConfig.liquidationRate - uint256(Constants.PERCENTAGE_DECIMALS)) * uint256(1e2)
    );
    // Ensure that liquidation rate is less than minCollateralRatio so that liquidations are not locked
    // up causing accounts to remain insolvent
    require(liquidationRate < vaultConfig.minCollateralRatioBPS);
    
+   // Ensure that the annualized fee must be 12.75% or less. 12.75% = 0.05 (5 BPS fee) * 255
+	require(vaultConfig.feeRate5BPS <= 255);    

    // Reserve fee share must be less than or equal to 100
    require(vaultConfig.reserveFeeShare <= Constants.PERCENTAGE_DECIMALS);
    require(vaultConfig.maxBorrowMarketIndex != 0);

    // Secondary borrow currencies cannot change once set
    require(
        existingVaultConfig.secondaryBorrowCurrencies[0] == 0 ||
        existingVaultConfig.secondaryBorrowCurrencies[0] == vaultConfig.secondaryBorrowCurrencies[0]
    );
    require(
        existingVaultConfig.secondaryBorrowCurrencies[1] == 0 ||
        existingVaultConfig.secondaryBorrowCurrencies[1] == vaultConfig.secondaryBorrowCurrencies[1]
    );

    // The borrow currency cannot be duplicated as a secondary borrow currency
    require(vaultConfig.borrowCurrencyId != vaultConfig.secondaryBorrowCurrencies[0]);
    require(vaultConfig.borrowCurrencyId != vaultConfig.secondaryBorrowCurrencies[1]);
    if (vaultConfig.secondaryBorrowCurrencies[0] != 0 && vaultConfig.secondaryBorrowCurrencies[1] != 0) {
        // Check that these values are not duplicated if set
        require(vaultConfig.secondaryBorrowCurrencies[0] != vaultConfig.secondaryBorrowCurrencies[1]);
    }

    // Tokens with transfer fees create lots of issues with vault mechanics, we prevent them
    // from being listed here.
    Token memory assetToken = TokenHandler.getAssetToken(vaultConfig.borrowCurrencyId);
    Token memory underlyingToken = TokenHandler.getUnderlyingToken(vaultConfig.borrowCurrencyId);
    require(!assetToken.hasTransferFee && !underlyingToken.hasTransferFee); 

    store[vaultAddress] = vaultConfig;
}
```