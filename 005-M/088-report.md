xiaoming90

high

# No Validation Check Against Decimal Of Secondary Token

## Summary

There is no validation check against the decimal of the secondary token due to a typo. Thus, this will cause the vault to be broken entirely or the value of the shares to be stuck if a secondary token with more than 18 decimals is added.

## Vulnerability Detail

There is a typo in Line 65 within the `TwoTokenPoolMixin` contract. The validation at Line 65 should perform a check against the `secondaryDecimals` instead of the `primaryDecimals`. As such, no validation was performed against the secondary token.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/TwoTokenPoolMixin.sol#L65

```solidity
File: TwoTokenPoolMixin.sol
23:     constructor(
24:         NotionalProxy notional_, 
25:         AuraVaultDeploymentParams memory params
26:     ) PoolMixin(notional_, params) {
..SNIP..
55:         // If the underlying is ETH, primaryBorrowToken will be rewritten as WETH
56:         uint256 primaryDecimals = IERC20(primaryAddress).decimals();
57:         // Do not allow decimal places greater than 18
58:         require(primaryDecimals <= 18);
59:         PRIMARY_DECIMALS = uint8(primaryDecimals);
60: 
61:         uint256 secondaryDecimals = address(SECONDARY_TOKEN) ==
62:             Deployments.ETH_ADDRESS
63:             ? 18
64:             : SECONDARY_TOKEN.decimals();
65:         require(primaryDecimals <= 18);
66:         SECONDARY_DECIMALS = uint8(secondaryDecimals);
67:     }
```

If the decimal of the secondary tokens is more than 18, the `Stable2TokenOracleMath._getSpotPrice` will stop working as the code will revert in Line 24 below because the decimal of secondary tokens is more than 18.

When the `Stable2TokenOracleMath._getSpotPrice` function stop working, the vaults will be broken entirely because the settle vault and reinvest rewards functions will stop working too. This is because the settle vault and reinvest rewards functions will call the `Stable2TokenOracleMath._getSpotPrice` function internally, resulting in a revert.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/math/Stable2TokenOracleMath.sol#L16

```solidity
File: Stable2TokenOracleMath.sol
16:     function _getSpotPrice(
17:         StableOracleContext memory oracleContext, 
18:         TwoTokenPoolContext memory poolContext, 
19:         uint256 tokenIndex
20:     ) internal view returns (uint256 spotPrice) {
21:         // Prevents overflows, we don't expect tokens to be greater than 18 decimals, don't use
22:         // equal sign for minor gas optimization
23:         require(poolContext.primaryDecimals < 19); /// @dev primaryDecimals overflow
24:         require(poolContext.secondaryDecimals < 19); /// @dev secondaryDecimals overflow
25:         require(tokenIndex < 2); /// @dev invalid token index
```

## Impact

The `Stable2TokenOracleMath._getSpotPrice` will stop working, which will in turn cause the settle vault and reinvest rewards functions to stop working too. Since a vault cannot be settled, the vault is considered broken. If the reinvest rewards function cannot work, the value of users' shares will be stuck as the vault relies on reinvesting rewards to buy more BPT tokens from the market.

In addition, there might be some issues when calculating the price of the tokens since the vault assumes that both primary and secondary tokens have a decimal equal to or less than 18 OR some overflow might occur when processing the token value.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/TwoTokenPoolMixin.sol#L65
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/math/Stable2TokenOracleMath.sol#L16

## Tool used

Manual Review

## Recommendation

Update the code to perform the validation against the `secondaryDecimals` state variable.

```diff
constructor(
    NotionalProxy notional_, 
    AuraVaultDeploymentParams memory params
) PoolMixin(notional_, params) {
    ..SNIP..
    // If the underlying is ETH, primaryBorrowToken will be rewritten as WETH
    uint256 primaryDecimals = IERC20(primaryAddress).decimals();
    // Do not allow decimal places greater than 18
    require(primaryDecimals <= 18);
    PRIMARY_DECIMALS = uint8(primaryDecimals);

    uint256 secondaryDecimals = address(SECONDARY_TOKEN) ==
        Deployments.ETH_ADDRESS
        ? 18
        : SECONDARY_TOKEN.decimals();
-   require(primaryDecimals <= 18);
+   require(secondaryDecimals <= 18);
    SECONDARY_DECIMALS = uint8(secondaryDecimals);
}
```