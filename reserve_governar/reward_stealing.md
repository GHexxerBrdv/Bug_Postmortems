# [M] A malicious user can steal rewards from the other users

## Summary

The `StakingVault::_accrueRewards` does not accrue rewards for the users for the reward tokens which are not registered in the `RewardTokenRegistry`.

```solidity
if (!rewardTokenRegistry.isRegistered(rewardToken)) {
    rewardTrackers[rewardToken].payoutLastPaid = block.timestamp;
    continue;
}
```

However the global reward idex is also not updated for the same token. The governar role can remove the token from the reward token registry for some reasone and then they can re add the token back to the registry. Between this time if a user transfer their vault share to other address owned by them self, then last global index will start by zero for that address, since global reward index and user level reward index are skipped by above if branch for the unregistered token.

The means if token is again re added to the reward token registry then the new address will calculate the rewards with indexDelat = globalRewardIndex - 0. Which will allow the new address to accrue rewards from the global reward index. that will dilute other users rewards. and at the end user will end up with a huge reward balance for that token.

```solidity
// the new address index will be: rewardIndex - 0 = rewardIndex
uint256 deltaIndex = rewardInfo.rewardIndex - userRewardTracker.lastRewardIndex;

if (deltaIndex != 0) {
    uint256 supplierDelta = Math.mulDiv(balanceOf(_user), deltaIndex, uint256(10 ** decimals()) * SCALAR);
    userRewardTracker.accruedRewards += supplierDelta;
```

