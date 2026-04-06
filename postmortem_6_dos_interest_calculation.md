Welcome to another postmortem, where we will understand how the interest calculation could be DOS infinitely.

This bug was found on `monolith stablecoin factory` contest on sherlock. There was a interest model which was responsible for calculating interest for borrowed stablecoin from the protocol. However there was a case when the interest could DOSed and the protocol could never calculate interest in future.

Have a look in following function.

```solidity
function accrueInterest() public {
        // info -> time elapsed
        uint timeElapsed = block.timestamp - lastAccrue;
        if(timeElapsed == 0) return;

        uint256 gasBefore = gasleft(); // info -> gas calculation

        try interestModel.calculateInterest(
            totalPaidDebt,
            lastBorrowRateMantissa,
            timeElapsed,
            expRate,
            getFreeDebtRatio(),
            targetFreeDebtRatioStartBps,
            targetFreeDebtRatioEndBps
        ) returns (uint currBorrowRate, uint interest) {
            
        } catch {
            require(gasBefore >= INTEREST_CALCULATION_GAS_REQUIREMENT, "Not enough gas for accrueInterest");
        }
    }
```

as you can see in this function, the catch block is only checking if gas provided is enough if the `interestModel.calculateInterest` fails. You also can see if `calculateInterest` function fails then `lastAccrue` is not updated. 

Now let's see what are the cases when the `calculateInterest` could fail.

look closely in following snippet, which is part of the `` function.

```solidity
uint growthDecay = uint(wadExp(-int(_expRate * _timeElapsed))); // growth decay g
        
        if (_lastFreeDebtRatioBps < _targetFreeDebtRatioStartBps) {
            // r_new = r_old / g = r_old * exp(k * dt)
            currBorrowRate = _lastRate * 1e18 / growthDecay; // 18 decimal scale
            // @audit tricky math
            interest = _totalPaidDebt * (currBorrowRate - _lastRate) / _expRate / 365 days; // I = D * (r_new - r_old) / (k * 365 days)
        } else if (_lastFreeDebtRatioBps > _targetFreeDebtRatioEndBps) {
```

What if? the `growthDecay` becomes zero? if it becomes then `currBorrowRate = _lastRate * 1e18 / growthDecay;` will revert with division by zero error. Now there could be such case where `growthDecay` could be zero. According to function `wadExp`. it would return zero if `if (x <= -42139678854452767551) return 0;`.

that is possible only when timeelapsed is huge enough. that means for long time there is no interaction with protocol happens. if it happens then according to line `uint growthDecay = uint(wadExp(-int(_expRate * _timeElapsed)));` the `wadExp` function will be feeded by huge negative value. that could result in returning the function with zero value.

Now as we know returning zero value will revert the function and that is handled by the catch block where `lastAccrue` is not updated with block.timestamp. Now in future everytime any user interact with protocol the timeElapsed will become more and more huge. That will again result in revert. And this cycle will run forever, and interest calculation freez forever. 

According to numbers if no one interect with protocol for `200 days` then this exploit is possible. However that is not practical, but we can not ignore it because that can be happen in future.