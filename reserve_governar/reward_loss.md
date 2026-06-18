# [M] user can loss their rewards by transferring zero amount of vault shares

## Description

In `StakingVault` allows zero share transfer by not reverting while passing zero amount in `_update` function.

```solidity
function _update(address from, address to, uint256 value)
    internal
    override(ERC20Upgradeable, ERC20VotesUpgradeable)
    accrueRewards(from, to)
{
    super._update(from, to, value);
    _moveOptimisticDelegateVotes(optimisticDelegatees[from], optimisticDelegatees[to], value);
}
```

The above function is overridden by the `StakingVault` contract which does not restrict zero share transfer. Also before performing transfer the rewards are accrued for the sender and receiver. And according to rewards math, the rewards are rounded down and end up with loosing 1 wei of rewards every time the accrue rewards function is called.

```solidity
Math.mulDiv(balanceOf(_user), deltaIndex, uint256(10 ** decimals()) * SCALAR);
```

However any user can call `transferFrom(user1, user2, 0)` to transfer zero amount of shares from `user1` to `user2` and lose their rewards. Since there is no need of approval for zero amount transfer, the function will be executed successfully. However in cheap chains like layer 2, the repitative transfer zero amount shares will be cheaper then the rewards which would be lost every time.

However the 1 wei loss seems not impact, but consider tokens with big value and small decimals, even 1 wei can effect with significant loss in value from the rewards. And repitative call in small duration can might drain the rewards from the users.

## Root Cause

> [!NOTE]
> protocol does not restrict zero amount share transfer, this could not be an issue, but every transfer was effecting the rewards of sender and recipient. That lead this zero amount share transfer to lose rewards.