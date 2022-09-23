Arbitrary-Execution
# `getRouterImplementation` is susceptible to function selector collisions

## Summary
`getRouterImplementation` is susceptible to function selector collisions

## Vulnerability Detail
`getRouterImplementation` in `Router.sol` determines which contract address to forward a function call to by comparing the called function selector to several function selectors per contract:
```
if (
    sig == NotionalProxy.batchBalanceAction.selector ||
    sig == NotionalProxy.batchBalanceAndTradeAction.selector ||
    sig == NotionalProxy.batchBalanceAndTradeActionWithCallback.selector ||
    sig == NotionalProxy.batchLend.selector
) {
    return BATCH_ACTION;
} else if (
  ...
```
There appears to be no checks in the contract or external tests to ensure that the current function selectors in `getRouterImplementation` are unique, and that any future additions to `getRouterImplementation` are unique. While function selector collisions are uncommon, the low number of bytes used in selectors means collisions are still possible. 

## Impact
Should a function selector collision occur in `getRouterImplementation` at a minimum users would only be able to call the function that occurs first in the block of `if/else if/else` statements. In the worst case, this means a users may call a function that they were not intending to call.

## Code Snippet
N/A
## Tool used

Manual Review

## Recommendation
Consider implementing the [diamond proxy-pattern](https://eips.ethereum.org/EIPS/eip-2535) as `Router.sol` is a good candidate for this proxy-pattern. Otherwise, consider implementing a test to ensure that there are no function selector collisions in any of the compiled contracts.
