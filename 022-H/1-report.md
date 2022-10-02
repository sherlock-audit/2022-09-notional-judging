Arbitrary-Execution
# Idiosyncratic fCash can prevent a user from improving their collateral ratio

## Summary
Idiosyncratic fCash can prevent a user from improving their collateral ratio

## Vulnerability Detail
`enterVault` in `VaultAccountAction.sol` can be used by users to instantiate a position on a vault, increase the borrow on a position in a vault, and decrease the borrow position on a vault by supplying a non-zero `depositAmountExternal` with a zero additional fCash borrow. Importantly, using `enterVault` to decrease a borrow position allows a user to decrease their collateral ratio and avoid a liquidation. Within `enterVault`, `borrowAndEnterVault` in `VaultAccount.sol` is ultimately the function that will borrow fCash as requested and set a user's vault account position post-calculations. Inside `borrowAndEnterVault` there is a check to ensure that if a user does not borrow any fCash that the maturity they are requesting to enter a position into is still valid:

```solidity
if (fCashToBorrow > 0) {
    ...
} else {
    // Ensure that the maturity is a valid one if we are not borrowing (borrowing will fail)
    // against an invalid market.
    VaultConfiguration.checkValidMaturity(
        vaultConfig.borrowCurrencyId,
        maturity,
        vaultConfig.maxBorrowMarketIndex,
        block.timestamp
    );
}
```

Note that this check will happen so long as a user is not attempting to borrow fCash, i.e. `fCashToBorrow = 0`. Most notably this check will still happen even when a user has a valid maturity and the user is attempting to lower their collateral ratio by supplying external tokens to their position without borrowing more fCash.

The reason why this is problematic is due to the logic in `checkValidMaturity`:

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

Therefore, when a maturity for a position is idiosyncratic, `checkValidMaturity` will revert. This will cause any function that uses `checkValidMaturity` to also revert, so `borrowAndEnterVault` and by extension `enterVault` will revert when a maturity is idiosyncratic.

## Impact
When a user's vault account position maturity is idiosyncratic, they will be unable to improve their collateral ratio using `enterVault` and thus are extremely susceptible to liquidations.

## Code Snippet
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

PoC (add to `tests/stateful/vaults/test_vault_entry.py`):
```python3
def test_reenter_vault_idiosyncratic_maturity(environment, vault, accounts):
    environment.notional.updateVault(
        vault.address,
        get_vault_config(flags=set_flags(0, ENABLED=True), currencyId=2, maxBorrowMarketIndex=3),
        100_000_000e8,
    )

    # retrieve the 12-month maturity timestamp that the account will enter
    maturity = environment.notional.getActiveMarkets(1)[2][1]

    # accounts[1] enters a vault with a 12-month maturity
    environment.notional.enterVault(
        accounts[1], vault.address, 25_000e18, maturity, 0, 0, "", {"from": accounts[1]}
    )

    # block time is rolled 1 quarter
    chain.mine(1, timestamp=chain.time() + SECONDS_IN_QUARTER)
    # after 1 quarter each Notional market is re-initialized
    environment.notional.initializeMarkets(2, False)

    # 1 year maturity is now idiosyncratic, attempt to enter the vault again
    environment.notional.enterVault(
        accounts[1], vault.address, 25_000e18, maturity, 0, 0, "", {"from": accounts[1]}
    )
```

## Tool used

Manual Review

## Recommendation
Consider adding another check before calling `checkValidMaturity` in `borrowAndEnterVault` that ensures a maturity is only checked when a user is entering into a position for the first time i.e. `vaultAccount.maturity == 0`. This is safe to add even when `rollVaultPosition` in `VaultAccountAction.sol` is occurring because `rollVaultPosition` forces a user to borrow a non-zero fCash amount into the new position thus it will always enter the `if` block in `borrowAndEnterVault` and therefore check that the new maturity is valid.
