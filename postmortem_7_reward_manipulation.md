welcome to another postmoortem. in this exploit deep dive i will cover a simple yet mind opening attack vecotor that was found in sherlock contest `Super DCA Liquidity Network`. 

In this exploit there are multiple things are related. Like uniswap v4 integration, hooks, price manimulation etc. This exploit was combination of multiple attacks.

Let's see how this could steal rewards od the protocol.
welcome to another postmoortem. in this exploit deep dive i will cover a simple yet mind opening attack vecotor that was found in sherlock contest `Super DCA Liquidity Network`. 

In this exploit there are multiple things are related. Like uniswap v4 integration, hooks, price manimulation etc. This exploit was combination of multiple attacks.

Let's see how this could steal rewards od the protocol.

Look into following function

```solidity
    function _handleDistributionAndSettlement(PoolKey calldata key, bytes calldata hookData) internal {
        // Must sync the pool manager to the token before distributing tokens
        poolManager.sync(Currency.wrap(superDCAToken));

        // Derive the non-DCA token for accrual calculation
        // The staking contract uses this to determine reward amounts
        address otherToken = superDCAToken == Currency.unwrap(key.currency0)
            ? Currency.unwrap(key.currency1)
            : Currency.unwrap(key.currency0);

        // Calculate pending rewards from the external staking contract
        uint256 rewardAmount = staking.accrueReward(otherToken);
        if (rewardAmount == 0) return;

        // Check if pool has liquidity before proceeding with donation
        uint128 liquidity = IPoolManager(msg.sender).getLiquidity(key.toId());
        if (liquidity == 0) {
            // If no liquidity, try sending everything to developer (do not revert if mint fails)
            _tryMint(developerAddress, rewardAmount);
            return;
        }

        // Split the mint amount between developer and community (50/50)
        uint256 developerShare = rewardAmount / 2;
        uint256 communityShare = rewardAmount - developerShare;

        // Mint developer share (ignore failure)
        _tryMint(developerAddress, developerShare);

        // Mint community share and donate to pool only if mint succeeds
        // This prevents donation of tokens that don't exist
        if (_tryMint(address(poolManager), communityShare)) {
            // Donate community share to pool
            if (superDCAToken == Currency.unwrap(key.currency0)) {
                IPoolManager(msg.sender).donate(key, communityShare, 0, hookData);
            } else {
                IPoolManager(msg.sender).donate(key, 0, communityShare, hookData);
            }

            // Settle the donation to complete the transaction
            poolManager.settle();
        }

        /// @dev: At this point, there are DCA tokens left in the hook for the other pools.
    }
```

this function is called in hooks `_beforeAddLiquidity` and `_beforeRemoveLiquidity`. as name suggested these hooks are called when adding & removing liquidity in pool. the protocol was integrated with uniswap v4 pools. so when ever a user depositing/removing liquidity from the protocol it calls this two hooks accordingly. inside this hooks the above function called every time.

Now look into following lines from function

```solidity
            if (superDCAToken == Currency.unwrap(key.currency0)) {
                IPoolManager(msg.sender).donate(key, communityShare, 0, hookData);
            } else {
                IPoolManager(msg.sender).donate(key, 0, communityShare, hookData);
            }
```

here we are calling donate function of uniswap v4. And according to uniswap v4 docs this function donates to the current tick range which is `slot0.tick`.

Now the function was responcible to donate rewards everytime when someone add or remove liquidity in the protocol. 

However if you see the function is donating rewards only those whose position is in the current liquidity range. This is where the exploit born.

There is multiple precondition for this exploit.

1. the attcker must have LP position in the pool.
2. the contract has some rewards pending to distribute.


Now suppose attacker try to add liquidity in the protocol. but there is a catch. Here attacker can maipulate the current tick in uniswap v4 pool externally. 

Now if attacker could just frontrun their own transaction of adding/removing liquidity in the protocol, and do certain swaps to move current tick of uniswap v4 pool to their tick range. after this price manipulation, the tansaction just execute where rewards are distributed to the current tick. which is now attacker manipulated.

After stealing rewards they can move tick back. There is another catch. This attack is only profitable if the net reward are greter then the swap fees. otherwise attacker lose fees and exploit will not be profitable.


Here you can see multiple eploit patterns has been used something like front running, price manipulation, reward manipulation etc.

However to do swaps attacker could just take advatage of flash loan also. but that's the another case and way to exploit.

> Always keep in mind, while auditing do not just understand the code, try to visulize the system and think about different possible unexpected flows that a user could take in real life. thinking in that way will improve you as an security researcher.


I hope this postmortem helps you in your learning.

Thank you,
GB-53F8
Look into following function

```solidity
    function _handleDistributionAndSettlement(PoolKey calldata key, bytes calldata hookData) internal {
        // Must sync the pool manager to the token before distributing tokens
        poolManager.sync(Currency.wrap(superDCAToken));

        // Derive the non-DCA token for accrual calculation
        // The staking contract uses this to determine reward amounts
        address otherToken = superDCAToken == Currency.unwrap(key.currency0)
            ? Currency.unwrap(key.currency1)
            : Currency.unwrap(key.currency0);

        // Calculate pending rewards from the external staking contract
        uint256 rewardAmount = staking.accrueReward(otherToken);
        if (rewardAmount == 0) return;

        // Check if pool has liquidity before proceeding with donation
        uint128 liquidity = IPoolManager(msg.sender).getLiquidity(key.toId());
        if (liquidity == 0) {
            // If no liquidity, try sending everything to developer (do not revert if mint fails)
            _tryMint(developerAddress, rewardAmount);
            return;
        }

        // Split the mint amount between developer and community (50/50)
        uint256 developerShare = rewardAmount / 2;
        uint256 communityShare = rewardAmount - developerShare;

        // Mint developer share (ignore failure)
        _tryMint(developerAddress, developerShare);

        // Mint community share and donate to pool only if mint succeeds
        // This prevents donation of tokens that don't exist
        if (_tryMint(address(poolManager), communityShare)) {
            // Donate community share to pool
            if (superDCAToken == Currency.unwrap(key.currency0)) {
                IPoolManager(msg.sender).donate(key, communityShare, 0, hookData);
            } else {
                IPoolManager(msg.sender).donate(key, 0, communityShare, hookData);
            }

            // Settle the donation to complete the transaction
            poolManager.settle();
        }

        /// @dev: At this point, there are DCA tokens left in the hook for the other pools.
    }
```

this function is called in hooks `_beforeAddLiquidity` and `_beforeRemoveLiquidity`. as name suggested these hooks are called when adding & removing liquidity in pool. the protocol was integrated with uniswap v4 pools. so when ever a user depositing/removing liquidity from the protocol it calls this two hooks accordingly. inside this hooks the above function called every time.

Now look into following lines from function

```solidity
            if (superDCAToken == Currency.unwrap(key.currency0)) {
                IPoolManager(msg.sender).donate(key, communityShare, 0, hookData);
            } else {
                IPoolManager(msg.sender).donate(key, 0, communityShare, hookData);
            }
```

here we are calling donate function of uniswap v4. And according to uniswap v4 docs this function donates to the current tick range which is `slot0.tick`.

Now the function was responcible to donate rewards everytime when someone add or remove liquidity in the protocol. 

However if you see the function is donating rewards only those whose position is in the current liquidity range. This is where the exploit born.

There is multiple precondition for this exploit.

1. the attcker must have LP position in the pool.
2. the contract has some rewards pending to distribute.


Now suppose attacker try to add liquidity in the protocol. but there is a catch. Here attacker can maipulate the current tick in uniswap v4 pool externally. 

Now if attacker could just frontrun their own transaction of adding/removing liquidity in the protocol, and do certain swaps to move current tick of uniswap v4 pool to their tick range. after this price manipulation, the tansaction just execute where rewards are distributed to the current tick. which is now attacker manipulated.

After stealing rewards they can move tick back. There is another catch. This attack is only profitable if the net reward are greter then the swap fees. otherwise attacker lose fees and exploit will not be profitable.


Here you can see multiple eploit patterns has been used something like front running, price manipulation, reward manipulation etc.

However to do swaps attacker could just take advatage of flash loan also. but that's the another case and way to exploit.

> Always keep in mind, while auditing do not just understand the code, try to visulize the system and think about different possible unexpected flows that a user could take in real life. thinking in that way will improve you as an security researcher.