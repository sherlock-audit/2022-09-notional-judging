Arbitrary-Execution
# Vault owner has outsized control over user accounts

## Summary
Vault owner has outsized control over user accounts

## Vulnerability Detail
Each vault has a number of different configuration flags that can be set. In addition to a standard 'pause' style flag, there are several other flags that can be used to restrict which actions can be performed on a vault and by whom the actions can be performed. The flags can be seen in `VaultConfiguration.sol` and below:

```solidity
uint16 internal constant ENABLED                         = 1 << 0;
uint16 internal constant ALLOW_ROLL_POSITION             = 1 << 1;
// These flags switch the authentication on the vault methods such that all
// calls must come from the vault itself.
uint16 internal constant ONLY_VAULT_ENTRY                = 1 << 2;
uint16 internal constant ONLY_VAULT_EXIT                 = 1 << 3;
uint16 internal constant ONLY_VAULT_ROLL                 = 1 << 4;
uint16 internal constant ONLY_VAULT_DELEVERAGE           = 1 << 5;
uint16 internal constant ONLY_VAULT_SETTLE               = 1 << 6;
// External vault methods will have re-entrancy protection on by default, however, some
// vaults may need to call back into Notional so we can whitelist them for re-entrancy.
uint16 internal constant ALLOW_REENTRANCY                = 1 << 7;
```

However, allowing the vault to have such large control over user's actions is restrictive. This is especially pertinent for the `ONLY_VAULT_EXIT` flag as when this flag is set only the vault itself is allowed to call `exitVault`.

## Impact
Setting any of the `ONLY_VAULT_*` flags above could greatly impact users, especially `ONLY_VAULT_EXIT` which would prevent users from being able to exit their position in a vault and withdraw their collateral.

## Code Snippet
N/A

## Tool used

Manual Review

## Recommendation
Consider removing the `ONLY_VAULT_EXIT` flag and corresponding logic as contracts should never be able to prevent users from withdrawing their collateral when their position is healthy.

