# Understand how redeemption can steal user deposits from protocol

Welcome to another postmortem, then i will explain how unfair redeemption can steal user funds from the protocol. This bug was found in `monolith stablecoin factory` contest on sherlock.

This exploit has multiple pre condition to be happened. but if it happens any one can steal deposits from users and make them unable to withdraw their funds from the protocol. 

Look into the following function.

```solidity
function redeem(uint amountIn, uint minAmountOut) external returns (uint amountOut) {
        accrueInterest();
        // calculate amountOut in internal 18 decimals
        uint internalAmountOut = getRedeemAmountOut(amountIn); // 18 decimal collateral amount after excluding fees
        require(internalAmountOut > 0, "amount out is zero");
        // Convert to token decimals (rounds down)
        amountOut = internalToCollateral(internalAmountOut); // actual amount of collateral that being out
        require(amountOut >= minAmountOut, "insufficient amount out"); // check for minimum out token for slippage control
        
        // Check redeemable collateral in internal representation
        uint256 totalInternalCollateral = collateralToInternal(collateral.balanceOf(address(this))); // get collateral amount hold by this contract in internal representation
        // this check ensures the redeemable collateral is not redeemed
        require(totalInternalCollateral - internalAmountOut >= nonRedeemableCollateral, "Insufficient redeemable collateral");
        
        // repay on behalf of free debtors
        totalFreeDebt -= amountIn; // update the amount of free debt
        coin.transferFrom(msg.sender, address(this), amountIn);
        coin.burn(amountIn);

        // distribute collateral redemption per free debt share (in internal representation)
        // @audit did not get this
        epochRedeemedCollateral[epoch] += internalAmountOut.mulDivUp(1e36, totalFreeDebtShares); // update the epochRedeemedCollateral

        collateral.safeTransfer(msg.sender, amountOut);

        // Intentional division by zero and revert if totalFreeDebt is 0
        if( totalFreeDebtShares / totalFreeDebt > 1e9) {
            epoch++;
            totalFreeDebtShares = totalFreeDebtShares.mulDivUp(1e18,1e36); // @audit why this has done? -> 100e18 * 1e18 / 1e36
            emit NewEpoch(epoch);
        }

        emit Redeemed(msg.sender, amountIn, amountOut);
        return amountOut;
    }
```

any one can call this function and repay part of everyone's debt. and in return they get propostional amount of their collateral. 

However there is an exploit. consider the following case.

if there is two free debt user.

1. deposited 1000$ and has 500$ debt.
2. deposited 1000$ but no debt.

This means 100% shares are owned by the user who has debt. Now suppose the collateral price goes down and make user position underwater. That means if someone redeem to pay user debt. Then they will get more collateral as the price of collateral just droped. in result the redeemer will just seize the whole principal of the user but due to price drop other free debt user will also lose their collateral even if they have no any debt yet. Now if they want to withdraw they will not able to get their collateral back. it will stuck forever in the contract. 

For more understanding consider following POC:

suppose there is two free debt user

- alice has put in 10000e18 of collateral and has free debt of 5000e18 of coin
- bob has put in 10000e18 of collateral and does not have any debt
- this time 100% of shares are owned by alice alone
- now suppose collateral price drops which makes the collateral cheap. result in more collateral exchange with coin
- now someone comes and redeem 5000e18 - 1 coin and get collateral back from all the free debt user according to their shares
- but here all shares are owned by alice and price has been dropped, the redeemer will get more amount of collateral then it will get in normal price
- this make alice collateral 0 because she has not that much amount of collateral. result in whole collateral consumption of her,
- the redeemer will get their remaining collateral from the bob.
- that makes the bob unable to get his collateral back.

This exploit has nother POV also. Have a look in following case.

- there are two free debt user and both have same debt -> which means both have same amount of shares -> 50 / 50
- but one user has less collateral put in then another
- now price drops
- after that if someone redeems from the protocol then according to shares both user should deduct same amount of collateral from their account
- due to less amount of collateral of one user, and price drop. that user will lose their whole collateral. which is less then 50 % of shares
- now that additional share collateral will deducted from the second user who have more collateral.
- in result instead of loosing 50 % of collateral, second user deducted with more then 50 % of shares collateral. which breaks the invariant


Following is with value example

Assume that the following setup:

- `collateralFactor` = 85%
- `redeemFeeBps = 1000` (10%)
- Collateral oracle price is \$1,000 per collateral token
- Two borrowers (Alice and Bob) are in free‑debt mode

Consider the following scenario.

1.  Alice and Bob borrow the same amount, so they end up with the same share of total free debt.

Alice deposits 1.2 collateral tokens (value \$1,200) and borrows 1,000 Coin (free debt). Bob deposits 3.0 collateral tokens (value \$3,000 at \$1,000/token) and borrows 1,000 Coin (free debt). So, at this point:

- `totalFreeDebt` = 2,000 Coin
- `totalFreeDebtShares` is split 50/50 (Alice and Bob have equal shares)
- Total collateral = 1.2 + 3.0 = 4.2

1.  Collateral price drops from \$1000 to \$400
2.  Charles (the redeemer) executes `redeem(amountIn = 1,400 Coin)`. Collateral value paid out = 1,400 × (1 − 0.10) = \$1,260. At \$400/token: Collateral tokens transferred to redeemer = 1,260 / 400 = 3.15 tokens

Since both Alice and Bob have equal `freeDebtShares`, each should pay 50% of the seized collateral: 1.575 tokens.

However, within the `updateBorrower()` function, it caps Alice's loss to 0, so Alice only loses 1.2 tokens, not 1.575 tokens. Meanwhile, Bob loses 1.575 tokens. Technically, Bob is the only borrower with remaining collateral since Alice's position is "wiped" off.

Let's see how much collateral is left in the system and belongs to Bob:

- 4.2 - 3.15 (Withdraw by Charles) = 1.05 (collateral left)

Bob deposited 3.0 collateral, if 1.575 (his fair share of deductions) is deducted from his account due to redemption, he should have 1.425 collateral left. However, in this case, only 1.05 collateral remains, which means Bob overpaid or lost 0.375 collateral. Meanwhile, Alice underpaid by 0.375 collateral. If Bob tries to withdraw the full 1.425, `adjust()` will revert due to insufficient actual collateral balance.