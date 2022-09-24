Arbitrary-Execution
# `deleverageAccount` can still be called when a vault is paused

## Summary
`deleverageAccount` can still be called when a vault is paused

## Vulnerability Detail
Every vault has an `ENABLED` flag that can be toggled on an off, and is used to prevent certain vault account functions from being called in `VaultAccountAction.sol` when a vault is 'Paused'; these functions include: `enterVault` and `rollVaultPosition`. However, `deleverageAccount` is still able to be called even when a vault is paused.

## Impact
When the `ENABLED` flag is not set, meaning a vault is paused, liquidators will still be able to liquidate vault account positions. However, users are still able to call `exitVault` to either fully exit their position or lower their collateral ratio if necessary to avoid liquidation.

## Code Snippet
Failing test (add to `tests/stateful/vaults/test_vault_deleverage.py`):
```python
def test_deleverage_paused(environment, accounts, vault):
    environment.notional.updateVault(
        vault.address,
        get_vault_config(currencyId=2, flags=set_flags(0, ENABLED=False)),
        100_000_000e8,
    )
    maturity = environment.notional.getActiveMarkets(1)[0][1]

    environment.notional.enterVault(
        accounts[1], vault.address, 25_000e18, maturity, 100_000e8, 0, "", {"from": accounts[1]}
    )
    vault.setExchangeRate(0.85e18)
    (cr, _, _) = environment.notional.getVaultAccountCollateralRatio(accounts[1], vault)
    assert cr < 0.2e9

    # would expect this call to revert when a vault is paused
    with brownie.reverts("Cannot Enter"):
        environment.notional.deleverageAccount(
            accounts[1], vault.address, accounts[2], 25_000e18, False, "", {"from": accounts[2]}
        )
```

## Tool used

Manual Review

## Recommendation
Consider adding the following `require` statement to `deleverageAccount`:
```solidity
require(vaultConfig.getFlag(VaultConfiguration.ENABLED), "Cannot Enter");
```
