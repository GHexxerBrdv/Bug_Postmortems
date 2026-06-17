# [L] passive users can not claim already accrued rewards for removed reward token

## Description

The `StakingVault` does not allow reward accrual after a reward token is removed from the `RewardTokenRegistry`.

```solidity
function _accrueRewards(address _caller, address _receiver) internal {
    address[] memory _rewardTokens = rewardTokens.values(); //>/ fetch all reward tokens
    uint256 _rewardTokensLength = _rewardTokens.length; //>/ total number of reward tokens

    for (uint256 i; i < _rewardTokensLength; i++) {
        address rewardToken = _rewardTokens[i];
        
        
        if (!rewardTokenRegistry.isRegistered(rewardToken)) {
            rewardTrackers[rewardToken].payoutLastPaid = block.timestamp;
            continue;
        }
```

as shown in the code above, the function skips reward accrual for reward tokens that are removed. due to this the reward accrual for the token will be skipped for passive users who has already accrued rewards for the token since last they have interacted. However the readme is stating that user can accrue their already accrued reward for the token but it is only true for the active user who has interacted with the vault since the token was removed. other users reward accrual will be skipped and they lose the reward.