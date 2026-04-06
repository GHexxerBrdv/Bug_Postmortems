welcome to another postmortem. Here we will understand how incorrect liquidate amount could break the liquidation.

As we all know all lending protocols follows the same kinf of principal that if position becomes underwater then it should be liquidate to prevent user and protocol form bad debt. However the protocol always put limit that how much liquidator can liquidate any position in one go.

The bug as found in Lend audit contest on sherlock. 

The Lend protocol is crosschain lending and borrowing protocol that facilitate user to deposit and borrow cross chain live.

However while liquidate user position the protcol is calculating max liquidate amount. which is max repay amount that any liquidator can repay in one go to liquidate any user position.

See the following function

```solidity
function getMaxLiquidationRepayAmount(address borrower, address lToken, bool isSameChain)
    external
    view
    returns (uint256)
{
    uint256 currentBorrow = 0;

    currentBorrow += isSameChain
        ? borrowWithInterestSame(borrower, lToken)
        : borrowWithInterest(borrower, lToken);

    uint256 closeFactorMantissa = LendtrollerInterfaceV2(lendtroller).closeFactorMantissa();
    uint256 maxRepay = (currentBorrow * closeFactorMantissa) / 1e18;

    return maxRepay;
}
```

Here if the borrow and deposit was made on the same chain then there is no problem anywhere. but the main problem arise with cross chain borrow. where user initiated borrow from chain A and actual borrow was on chain B. 

in this case the above function uses `borrowWithInterest` function to calculate borrow amount. later the max repay amount is calculated further. 

However there is a problem in `borrowWithInterest` function. this function was calculating borrow amount if the borrow was initiated in chain B(which is destination chain). However it will return `zero` if the borrow was initiated in chain A. 

In this perticular case where the borrow was initiated in source chain the max repay amount could be zero. That does not satisfy the liquidate condition as shown below

```solidity
require(params.repayAmount <= maxLiquidationAmount, "Exceeds max liquidation");
```

Here the check will always fail in such case. that prevent any user to liquidate position.

> Always keep in mind while auditing smart contract, that how the perticular value is calculated, check if it is calculated in right way or wrong. If wrong then there could be bug.