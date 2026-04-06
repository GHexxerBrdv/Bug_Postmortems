Welcome to another postmortem. this time i will explain yout how a variable updated incorrectly can cause incorrect reward calculation.

This bug was found on sherlock contest `SuperDCA Liquidity Network`. 

To get some context see the following constructor and function

```solidity
    constructor(address _superDCAToken, uint256 _mintRate, address _owner) Ownable(_owner) {
        if (_superDCAToken == address(0)) revert SuperDCAStaking__ZeroAddress();
        DCA_TOKEN = _superDCAToken;
        mintRate = _mintRate;
@>      lastMinted = block.timestamp;
    }

```

see that while deploying the contract, `lastMinted` is updated to block.timestamp.

```solidity
    function _updateRewardIndex() internal {
        if (totalStakedAmount == 0) return; 
        uint256 elapsed = block.timestamp - lastMinted;
        if (elapsed == 0) return;
        uint256 mintAmount = elapsed * mintRate;
        rewardIndex += Math.mulDiv(mintAmount, 1e18, totalStakedAmount);
        lastMinted = block.timestamp;
        emit RewardIndexUpdated(rewardIndex);
    }
```

see this function calculated the elapsed time and calculate the `rewardIndex` according to `mintAmount` and `totalStakedAmount`. and update the `lastMinted`.

```solidity
    function stake(address token, uint256 amount) external override {
        if (amount == 0) revert SuperDCAStaking__ZeroAmount();
        if (gauge == address(0)) revert SuperDCAStaking__ZeroAddress();
        if (!ISuperDCAGauge(gauge).isTokenListed(token)) revert SuperDCAStaking__TokenNotListed();
        _updateRewardIndex();
        IERC20(DCA_TOKEN).transferFrom(msg.sender, address(this), amount);
        TokenRewardInfo storage info = tokenRewardInfoOf[token];
        info.stakedAmount += amount;
        info.lastRewardIndex = rewardIndex; 

        totalStakedAmount += amount;
        userStakes[msg.sender][token] += amount;
        userTokenSet[msg.sender].add(token);

        emit Staked(token, msg.sender, amount);
    }
```

Now see in this function. it is calling `_updateRewardIndex()` before increasing `totalStakedAmount`. 

Now there is a case where this protocol can act abnormal. 

Suppose the contract is just deployed. And there is only one user who stakes in protocol. 

In that case if you see the `_updateRewardIndex()` is called before increasing `totalStakedAmount`. however `_updateRewardIndex` function will return becasue in first deposit `totalStakedAmount` is zero withought updating `lastMinted`.

After the first deposit `totalStakedAmount` is updated with user stakes. Now cusider there is second deposit happened. then wrong `lastMinted` will be used to mint the rewards. 

Let's consider the following example to complately understand the bug.

- Contract deployed on day 1. block.timestamp is 0. Mint rate is 100.

- UserA deposit 100e18 tokens at day 2. Since totalStakedAmount is 0 when calling _updateRewardIndex(), the function just returns and does not update the lastMinted variable. The block.timestamp is now 86400 since oen day passed.

- So At t1, `totalStaked = 100e18`, `lastMinted = 0`

- UserB deposit 100e18 tokens on day 3. block.timestamp is now 172800. (two days have passed since deployment, one day has passed since first stake)
The _updateRewardIndex() calculates the rewards as,

```bash
elasped = 172800 - 0
mintAmount = 172800 * 100 = 1728000.
```

Even though tokens are only staked from day 2 to day 3 (one day), two days worth of tokens are minted instead.

So this was the bug that was overminitng reward token. However this was medium finding as no major fund theft was there. it can be only possible dor first deposit.

> Always keep in mind, while auditing always check how rewards are calculated, what things are there on which reward calculation is dependent, and when they updates.
