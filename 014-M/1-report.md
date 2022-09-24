Arbitrary-Execution
# Users are required to redeem a non-zero `vaultSharesToRedeem` when calling `exitVault` prior to maturity

## Summary
Users are required to redeem a non-zero `vaultSharesToRedeem`when calling `exitVault` prior to maturity

## Vulnerability Detail
`exitVault` in `VaultAccountAction.sol` can be called prior to an account's maturity timestamp to either completely exit a vault or improve the collateral ratio of a position. In the scenario that a user wants to improve the collateral ratio of their position in a vault, they can choose to decrease their fCash borrow by 'lending' a specified amount of fCash and subsequently paying off the lend immediately. Additionally, users must supply a `vaultSharesToRedeem` value, which will be converted to `strategyTokens` and subsequently used to help pay off the fCash lend:

```solidity
uint256 strategyTokens = vaultState.exitMaturity(vaultAccount, vaultSharesToRedeem);
...
if (strategyTokens > 0) {
    underlyingToReceiver = vaultConfig.redeemWithDebtRepayment(
        vaultAccount, receiver, strategyTokens, vaultState.maturity, exitVaultData
    );
}
```

However, in order to call the function `redeemWithDebtRepayment` which contains the logic to repay the new fCash lend, a user must supply a value for `vaultSharesToRedeem` such that when converted to `strategyTokens` the value is non-zero. This logic is incorrect, as `redeemWithDebtRepayment` is able to recover a deficit by transferring tokens directly out of an account address. Since `redeemWithDebtRepayment` is not called, `vaultAccount.tempCashBalance` is not cleared as it would be set to 0 in that function. Therefore, when `setVaultAccount` is called to save the state of the account after all the operations in `exitVault` have occurred, it will fail the following `require` statement as `tempCashBalance` is still non-zero:

```solidity
// The temporary cash balance must be cleared to zero by the end of the transaction
require(vaultAccount.tempCashBalance == 0); // dev: cash balance not cleared
```

## Impact
A user is required to sell a non-zero amount of `strategyTokens` when improving the collateral ratio of their position by calling `exitVault` prior to maturity. While the workaround is simple (have `vaultSharesToRedeem` be a small number), `exitVault` failing when a user is able to improve their collateral ratio accordingly is still an unexpected `revert` and should be fixed.

## Code Snippet
Failing test (add to `tests/stateful/vaults/test_vault_exit.py`):
```python
def test_exit_vault_transfer_from_account_no_strategy_token_sell(environment, vault, accounts):
    environment.notional.updateVault(
        vault.address,
        get_vault_config(flags=set_flags(0, ENABLED=True), currencyId=2),
        100_000_000e8,
    )
    maturity = environment.notional.getActiveMarkets(1)[0][1]

    environment.notional.enterVault(
        accounts[1], vault.address, 100_000e18, maturity, 100_000e8, 0, "", {"from": accounts[1]}
    )

    (collateralRatioBefore, _, _) = environment.notional.getVaultAccountCollateralRatio(
        accounts[1], vault
    )
    vaultAccountBefore = environment.notional.getVaultAccount(accounts[1], vault).dict()
    balanceBefore = environment.token["DAI"].balanceOf(accounts[1])

    (amountUnderlying, _, _, _) = environment.notional.getDepositFromfCashLend(
        2, 100_000e8, maturity, 0, chain.time()
    )

    # This is the failing call that should work
    # Once the code change is made you will need to lower the min borrow of the vault or it will still fail
    environment.notional.exitVault(
        accounts[1], vault.address, accounts[1], 0, 50_000e8, 0, "", {"from": accounts[1]}
    )

    balanceAfter = environment.token["DAI"].balanceOf(accounts[1])
    vaultAccount = environment.notional.getVaultAccount(accounts[1], vault).dict()
    (collateralRatioAfter, _, _) = environment.notional.getVaultAccountCollateralRatio(
        accounts[1], vault
    )
    vaultState = environment.notional.getVaultState(vault, maturity)

    assert pytest.approx(balanceBefore - balanceAfter, rel=1e-8) == amountUnderlying - 50_000e18
    assert collateralRatioBefore < collateralRatioAfter

    assert vaultAccount["fCash"] == 50_000e8
    assert vaultAccount["maturity"] == maturity
    assert vaultAccount["vaultShares"] == vaultAccountBefore["vaultShares"] # not selling vaultShares

    assert vaultState["totalfCash"] == 50_000e8
    assert vaultState["totalAssetCash"] == 0
    assert vaultState["totalStrategyTokens"] == vaultAccount["vaultShares"]
    assert vaultState["totalStrategyTokens"] == vaultState["totalVaultShares"]

    check_system_invariants(environment, accounts, [vault])
```

## Tool used

Manual Review

## Recommendation
Consider removing the `if (strategyTokens > 0)` statement and always call `redeemWithDebtRepayment` in `exitVault`.
