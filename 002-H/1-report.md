Bnke0x0
# Overpayment of native ETH is not refunded to the buyer


## Vulnerability Detail

## Impact
`VaultConfiguration` accepts payments in native ETH, but does not return overpayments to the buyer. Overpayments are likely in the case of auction orders priced in native ETH.

The end user is likely to send more ETH than the final calculated price in order to ensure their transaction succeeds, since price is a function of `block.timestamp`, and the user cannot predict the timestamp at which their transaction will be included.
 
## Code Snippet

https://github.com/None/blob/None/contracts-v2/contracts/internal/vaults/VaultConfiguration.sol#L449-L452

                  '// Tokens with transfer fees are not allowed in vaults
        require(!underlyingToken.hasTransferFee);
        if (underlyingToken.tokenType == TokenType.Ether) {
            require(msg.value == depositAmountExternal, "Invalid ETH");'

## Tool used

Manual Review

## Recommendation
Calculate and refund overpayment amounts to callers.
