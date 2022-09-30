Waze
# Latest RoundData from Chainlink May Return Outdated Results

## Summary
Out of date price from chainlink can lead some fund lost.
## Vulnerability Detail
Across these contracts, you are using Chainlink's latestRoundData API, but there is only a check on baseAnswer.
## Impact
The result of latestRoundData API will be used across various functions, therefore, an outdated price from Chainlink can lead to loss of funds to end-users.
## Code Snippet
https://github.com/None/blob/None/leveraged-vaults/contracts/trading/oracles/wstETHChainlinkOracle.sol#L26-L43
## Tool used

Manual Review

## Recommendation
Consider adding the missing checks for stale data.

For example:
require(answeredInRound >= roundId, "Out of date price");
require(updateAt != 0, "Round not complete");