xiaoming90

medium

# No Settlement Cooldown On `CrossCurrencyfCashVault`

## Summary

The settlement cooldown was not implemented on the CrossCurrencyfCashVault.

## Vulnerability Detail

The Boosted3TokenAuraVault and MetaStable2TokenAuraVault vaults implement a cooldown to give some time between settlements to give the market time to arbitrage back into position as per the comments in `SettlementUtils` contract.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/settlement/SettlementUtils.sol#L50

```solidity
File: SettlementUtils.sol
46:     /// @notice Validates that the settlement is past a specified cool down period.
47:     /// @param lastSettlementTimestamp the last time the vault was settled
48:     /// @param coolDownInMinutes configured length of time required between settlements to ensure that
49:     /// slippage thresholds are respected (gives the market time to arbitrage back into position)
50:     function _validateCoolDown(uint32 lastSettlementTimestamp, uint32 coolDownInMinutes) internal view {
51:         // Convert coolDown to seconds
52:         if (lastSettlementTimestamp + (coolDownInMinutes * 60) > block.timestamp)
53:             revert Errors.InSettlementCoolDown(lastSettlementTimestamp, coolDownInMinutes);
54:     }
```

The following shows that the settlement cooldown is implemented on Boosted3TokenAuraVault and MetaStable2TokenAuraVault vaults.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/Boosted3TokenAuraVault.sol#L100

```solidity
File: Boosted3TokenAuraVault.sol
100:     function settleVaultNormal(
101:         uint256 maturity,
102:         uint256 strategyTokensToRedeem,
103:         bytes calldata data
104:     ) external {
..SNIP..
111:         Boosted3TokenAuraStrategyContext memory context = _strategyContext();
112:         SettlementUtils._validateCoolDown(
113:             context.baseStrategy.vaultState.lastSettlementTimestamp,
114:             context.baseStrategy.vaultSettings.settlementCoolDownInMinutes
115:         );
```

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/MetaStable2TokenAuraVault.sol#L105

```solidity
File: MetaStable2TokenAuraVault.sol
105:     function settleVaultNormal(
106:         uint256 maturity,
107:         uint256 strategyTokensToRedeem,
108:         bytes calldata data
109:     ) external {
..SNIP..
116:         MetaStable2TokenAuraStrategyContext memory context = _strategyContext();
117:         SettlementUtils._validateCoolDown(
118:             context.baseStrategy.vaultState.lastSettlementTimestamp,
119:             context.baseStrategy.vaultSettings.settlementCoolDownInMinutes
120:         );
```

During the settlement of CrossCurrencyfCashVault vault (USDC/DAI), a large amount of DAI will be traded for USDC in the open market to repay the debt. However, the cooldown feature is not implemented on the CrossCurrencyfCashVault vault gives the market time to arbitrage back into position before the next settlement starts. It is expected that the leverage vault will be trading with a large amount of DAI as the amount of DAI to be traded is a combination of all the vault users' leveraged assets. Thus, it will likely cause a swing in the market price.

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyfCashVault.sol#L121

```solidity
File: CrossCurrencyfCashVault.sol
113:     /**
114:      * @notice During settlement all of the fCash balance in the lend currency will be redeemed to the
115:      * underlying token and traded back to the borrow currency. All of the borrow currency will be deposited
116:      * into the Notional contract as asset tokens and held for accounts to withdraw. Settlement can only
117:      * be called after maturity.
118:      * @param maturity the maturity to settle
119:      * @param settlementTrade details for the settlement trade
120:      */
121:     function settleVault(uint256 maturity, uint256 strategyTokens, bytes calldata settlementTrade) external {
122:         require(maturity <= block.timestamp, "Cannot Settle");
123:         VaultState memory vaultState = NOTIONAL.getVaultState(address(this), maturity);
124:         require(vaultState.isSettled == false);
125:         require(vaultState.totalStrategyTokens >= strategyTokens);
126: 
127:         RedeemParams memory params = abi.decode(settlementTrade, (RedeemParams));
128:     
129:         // The only way for underlying value to be negative would be if the vault has somehow ended up with a borrowing
130:         // position in the lend underlying currency. This is explicitly prevented during redemption.
131:         uint256 underlyingValue = convertStrategyToUnderlying(
132:             address(0), vaultState.totalStrategyTokens, maturity
133:         ).toUint();
134: 
135:         // Authenticate the minimum purchase amount, all tokens will be sold given this slippage limit.
136:         uint256 minAllowedPurchaseAmount = (underlyingValue * settlementSlippageLimit) / SETTLEMENT_SLIPPAGE_PRECISION;
137:         require(params.minPurchaseAmount >= minAllowedPurchaseAmount, "Purchase Limit");
138: 
139:         NOTIONAL.redeemStrategyTokensToCash(maturity, strategyTokens, settlementTrade);
140: 
141:         // If there are no more strategy tokens left, then mark the vault as settled
142:         vaultState = NOTIONAL.getVaultState(address(this), maturity);
143:         if (vaultState.totalStrategyTokens == 0) {
144:             NOTIONAL.settleVault(address(this), maturity);
145:         }
146:     }
```

## Impact

Time is required between settlements to give the market time to arbitrage back into position after a settlement has occurred. If not, the next settlement will be executed with significant slippage.

## Code Snippet

https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/settlement/SettlementUtils.sol#L50
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/Boosted3TokenAuraVault.sol#L100
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/MetaStable2TokenAuraVault.sol#L105
https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyfCashVault.sol#L121

## Tool used

Manual Review

## Recommendation

It is recommended to implement a settlement cooldown on the CrossCurrencyfCashVault vault, similar to what has been implemented on Boosted3TokenAuraVault and MetaStable2TokenAuraVault vaults.