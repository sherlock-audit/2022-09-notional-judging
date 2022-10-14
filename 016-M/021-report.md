Arbitrary-Execution

high

# Idiosyncratic fCash can prevent users from exiting a vault pre-maturity

## Summary
Idiosyncratic fCash can prevent users from exiting a vault prior to maturity

## Vulnerability Detail

When `exitVault` in `VaultAccountAction.sol` is called prior to the maturity timestamp of a vault account position, `lendToExitVault` is called in order to 'lend' an fCash amount back to Notional. This in turn sets the `tempCashBalance` for a vault account which can then be paid off in a number of ways. Within `lendToExitVault`, `executeTrade` is called to determine the actual asset cash cost to lend the specified fCash amount. Before `executeTrade` crafts the trade to send to the Notional AMM, it checks to ensure the given maturity is valid via the `checkValidMaturity` function call in `VaultConfiguration.sol`:

```solidity
function checkValidMaturity(...) ... {
    bool isIdiosyncratic;
    uint8 maxMarketIndex = CashGroup.getMaxMarketIndex(currencyId);
    (marketIndex, isIdiosyncratic) = DateTime.getMarketIndex(maxMarketIndex, maturity, blockTime);
    require(marketIndex <= maxBorrowMarketIndex, "Invalid Maturity");
    require(!isIdiosyncratic, "Invalid Maturity");
}
```

Notice that one of the checks ensures that the given maturity is not idiosyncratic. According to the [Notional docs](https://docs.notional.finance/notional-v2/quarterly-rolls/idiosyncratic-fcash), idiosyncratic fCash occurs when an fCash asset's maturity does not correspond to an active liquidity pool. The Notional docs list an example of when this can occur: 

> For example, consider a user who took out a loan from the one-year liquidity pool. At the time of the next quarterly roll, that user's fCash would mature in nine months, and the nine-month liquidity pool would reset after the quarterly roll to be a new one-year liquidity pool. This would result in the user's nine-month fCash becoming idiosyncratic. 

In the above example, a 12-month fCash maturity will be idiosyncratic for at least one quarter (3 months). Since the `checkValidMaturity` call reverts when it encounters an idiosyncratic fCash maturity, `lendToExitVault`, and by extension `exitVault`, will not work for the duration of time that a maturity is idiosyncratic. 

## Impact
For users who have vault account positions where the maturity may become idiosyncratic (maturity date of 1 year or greater), there will be some length of time where they will not be able to call `exitVault`. In the event that a vault is compromised or any other scenario where a user needs to exit prior to maturity but their maturity is idiosyncratic, the user will be unable to do so. This is true for all users of an idiosyncratic maturity, even if a user has not borrowed any fCash in their vault position.

## Code Snippet
High-level call that reverts in `lendToExitVault`: https://github.com/notional-finance/contracts-v2/blob/cf05d8e3e4e4feb0b0cef2c3f188c91cdaac38e0/contracts/internal/vaults/VaultAccount.sol#L235

Underlying call that reverts in `executeTrade`: https://github.com/notional-finance/contracts-v2/blob/cf05d8e3e4e4feb0b0cef2c3f188c91cdaac38e0/contracts/internal/vaults/VaultConfiguration.sol#L935

Update the following values in `scripts/config.py`:
```python3
CurrencyDefaults = {
    ...
    "maxMarketIndex": 3,
    ...
}

nTokenDefaults = {
    "Deposit": [
        # Deposit shares
        [int(0.25e8), int(0.35e8), int(0.4e8)],
        # Leverage thresholds
        [int(0.80e9), int(0.80e9), int(0.81e9)],
    ],
    "Initialization": [
        # Annualized anchor rate
        [int(0.03e9), int(0.03e9), int(0.03e9)],
        # Target proportion
        [int(0.55e9), int(0.55e9), int(0.55e9)],
    ],
    ...
}
```

PoC (add to `tests/stateful/vaults/test_vault_exit.py`):
```python3
def test_exit_vault_idiosyncratic_maturity(environment, vault, accounts):
    environment.notional.updateVault(
        vault.address,
        get_vault_config(flags=set_flags(0, ENABLED=True), currencyId=2, maxBorrowMarketIndex=3),
        100_000_000e8,
    )

    # retrieve the 12-month maturity timestamp that the account will enter
    maturity = environment.notional.getActiveMarkets(2)[2][1]

    # accounts[1] enters a vault with a 12-month maturity and only deposits their own collateral
    # borrowing fCash is not required to enter a vault
    environment.notional.enterVault(
        accounts[1], vault.address, 25_000e18, maturity, 0, 0, "", {"from": accounts[1]}
    )
    accountPosition = environment.notional.getVaultAccount(accounts[1], vault)

    # block time is rolled 1 quarter
    chain.mine(1, timestamp=chain.time() + SECONDS_IN_QUARTER)
    # after 1 quarter each Notional market is re-initialized
    environment.notional.initializeMarkets(2, False)

    # due to some unforeseen circumstance (hack, poor vault strategy, etc.) accounts[1] attempts to
    # exit the vault prior to the maturity being settled and retrieve their collateral
    # despite not having borrowed any fCash, the maturity they entered into is now idiosyncratic and
    # thus exitVault will fail
    environment.notional.exitVault(
        accounts[1],
        vault.address,
        accounts[1],
        accountPosition["vaultShares"],
        0,
        0,
        "",
        {"from": accounts[1]},
    )
```

## Tool used

Manual Review

## Recommendation
`exitVault` requires a user to lend fCash via `lendToExitVault` in order to facilitate paying off any debts and exiting a vault. However, the actual function that repays the debt in `exitVault` is `redeemWithDebtRepayment` in `VaultConfiguration.sol`, which has logic to repay debts via the use of transferring underlying tokens from a user to the vault. Therefore, consider adding logic to `exitVault` that would enable users to directly exit a vault and pay off debt exclusively using underlying tokens, which would avoid the issue of attempting to execute a trade when their fCash maturity is idiosyncratic. Additionally, consider adding logic to `lendToExitVault` to return early when the passed-in fCash value is 0.
