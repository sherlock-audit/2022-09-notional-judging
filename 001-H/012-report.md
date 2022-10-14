Arbitrary-Execution

medium

# `ALLOW_REENTRANCY` logic is dangerous and should be carefully considered

## Summary
`ALLOW_REENTRANCY` logic is dangerous and should be carefully considered

## Vulnerability Detail
Strategy vaults are designed to earn profit by interacting with other market makers/protocols. Notional also allows vaults to interact with other components of the Notional protocol. To accomplish this, Notional added an `ALLOW_REENTRANCY` flag and logic that resets the `reentrancyStatus` mutex lock to allow vaults to call back into the Notional protocol. If `reentrancyStatus` was not cleared, Notional protocol-based strategy vaults would not be possible as the `reentrancyStatus` lock is shared across all functions within the Notional protocol as a result of Notional's router-based proxy (i.e. `Router.sol`). However, because `reentrancyStatus` is the lock for the entire Notional protocol, clearing it allows full access to all subsystems within Notional and not just the AMM. Therefore, if a user was granted execution via a reentrant token (not `ether` as Notional specifically uses `transfer` to avoid reentrancy) or if a vault was granted execution to interact with the Notional protocol, subsystems within Notional that are susceptible to reentrancy would no longer be protected. This is especially relevant to `VaultAccountAction.sol` as the functions in that contract do not follow the checks-effects-interactions pattern and thus are extremely susceptible to reentrancy attacks.

## Impact
Should a user or a vault be allowed to reenter into the Notional protocol, subsystems that were once protected by the `reentrancyStatus` storage variable may be susceptible to reentrancy which could critically impact the protocol.

## Code Snippet
Example of logic that allows reentrancy when the `ALLOW_REENTRANCY` flag is set:
https://github.com/notional-finance/contracts-v2/blob/cf05d8e3e4e4feb0b0cef2c3f188c91cdaac38e0/contracts/external/actions/VaultAccountAction.sol#L48-L51

## Tool used

Manual Review

## Recommendation
Notional should carefully consider the risks of allowing reentrant calls back into the protocol.
