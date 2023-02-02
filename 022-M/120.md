berndartmueller

medium

# Claiming accumulated rewards while the contract is underfunded can lead to a loss of rewards

## Summary

The claimable rewards for an NFT staker are capped at the Ajna token balance at the time of claiming. This can lead to a loss of rewards if the `RewardsManager` contract is underfunded with Ajna tokens.

## Vulnerability Detail

The `RewardsManager` contract keeps track of the rewards earned by an NFT staker. The accumulated rewards are claimed by calling the `RewardsManager.claimRewards` function. Internally, the `RewardsManager._claimRewards` function transfers the accumulated rewards to the staker.

However, the transferrable amount of Ajna token rewards are capped at the Ajna token balance at the time of claiming. If the accumulated rewards are higher than the Ajna token balance, the claimer will receive fewer rewards than expected. The remaining rewards cannot be claimed at a later time as the `RewardsManager` contract does not keep track of the rewards that were not transferred.

## Impact

If an NFT staker claims the accumulated rewards in a bad situation (when the `RewardsManager` contract is underfunded with Ajna tokens), the staker will receive fewer rewards than expected and is unable to claim the rest of the rewards at a later time.

## Code Snippet

[contracts/src/RewardsManager.sol#L479](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/RewardsManager.sol#L479)

```solidity
445: function _claimRewards(
446:     uint256 tokenId_,
447:     uint256 epochToClaim_
448: ) internal {
         // [...]

477:     uint256 ajnaBalance = IERC20(ajnaToken).balanceOf(address(this));
478:
479:     if (rewardsEarned > ajnaBalance) rewardsEarned = ajnaBalance;
480:
481:     // transfer rewards to sender
482:     IERC20(ajnaToken).safeTransfer(msg.sender, rewardsEarned);
483: }
```

## Tool used

Manual Review

## Recommendation

Consider reverting if insufficient Ajna tokens are available as rewards.
