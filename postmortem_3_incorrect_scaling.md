Welcome to Postmortem 3. in this postmortem i will explain how a small incorrect scaling can cause exploit or bug in the protocol.

This issue was found in `Monolith Stablecoin Factory` in shrlock. In the protocol they have introduces interest model to calculate interest on borrowed stablecoin. However there was incorrect scaling was applied that leads to incorrect interest calculation.

Look following part of interest model

```solidity
interest = _totalPaidDebt * ((_lastRate - MIN_RATE) / _expRate + 
          MIN_RATE * (_timeElapsed - timeToMin)) / 365 days / 1e18;
```

let's breakdown terms

term 1: `(_lastRate - MIN_RATE) / _expRate` = `[rate * seconds / 1e18] (simplified)` (because `_expRate = wadLn(2*1e18) / halfLife` contains 1e18)
term 2: `MIN_RATE * (_timeElapsed - timeToMin)` = `[rate * seconds]`

When both terms are divided by `365 days / 1e18`, the term 1 is divided by `1e18` twice. however according to protcol each terms should be in correct scale. By doing twice division by `1e18`. term 1 is not contributing in interest calculation entirly. This leads to incorrect interest calculation. 

Due to this protocol is calculating less interest as compar to expected interest that should be calculated. 

here is the poc that demonstrate this incorrect calculation

```solidity
        function test_InterestCalculationBug() public view {
        // --- Setup ---
        // This PoC demonstrates a scaling bug in the interest calculation when the borrow rate
        // decays and hits the minimum rate floor. The interest accrued during the decay
        // period is effectively ignored, leading to loss of yield.

        uint256 totalPaidDebt = 1_000_000 * 1e18; // 1M tokens
        uint256 lastRate = 1e17; // 10% APR
        uint256 MIN_RATE = 5e15; // 0.5% APR

        // Setting a higher half-life amplifies the loss during the decay period
        uint256 halfLife = 12 hours;
        uint256 timeElapsed = 3 days;

        uint256 expRate = uint256(wadLn(2 * 1e18)) / halfLife;

        uint256 buggyInterest;
        {
            // Set a high free debt ratio to trigger the rate decay logic
            uint256 lastFreeDebtRatioBps = 8000; // 80%
            uint256 targetFreeDebtRatioStartBps = 2000; // 20%
            uint256 targetFreeDebtRatioEndBps = 4000; // 40%

            // --- Execution ---
            // Get the interest calculated by the buggy contract
            (, buggyInterest) = interestModel.calculateInterest(
                totalPaidDebt,
                lastRate,
                timeElapsed,
                expRate,
                lastFreeDebtRatioBps,
                targetFreeDebtRatioStartBps,
                targetFreeDebtRatioEndBps
            );
        }

        // --- Correct Calculation ---
        // Manually calculate the interest with the corrected logic.
        uint256 correctInterest;
        {
            uint256 timeToMin = uint256(-wadLn(int256(MIN_RATE * 1e18 / lastRate))) / expRate;

            // 1. Calculate the interest from the decay period (unscaled value scaled to WAD, notice the * 1e18)
            uint256 decayAprSecondsWAD = (lastRate - MIN_RATE) * 1e18 / expRate;

            // 2. Calculate the interest from the flat period at MIN_RATE
            uint256 flatAprSecondsWAD = MIN_RATE * (timeElapsed - timeToMin);

            // 3. Sum the two parts (both are now WAD-scaled)
            uint256 totalAprSecondsWAD = decayAprSecondsWAD + flatAprSecondsWAD;

            // 4. Calculate the final correct interest
            correctInterest = totalPaidDebt * totalAprSecondsWAD / 365 days / 1e18;
        }

        console.log("--- PoC: Interest Calculation Bug ---");
        console.log("Buggy Interest Calculated:  ", buggyInterest / 1e18);
        console.log("Correct Interest Expected:  ", correctInterest / 1e18);
        console.log("Loss of Yield:              ", (correctInterest - buggyInterest) / 1e18);

        // The buggy interest is significantly less than the correct interest,
        // demonstrating a loss of funds for the protocol.
        assertLt(buggyInterest, correctInterest, "Buggy interest should be less than correct interest");
        assertGt(correctInterest - buggyInterest, correctInterest / 2, "Yield loss is unexpectedly high");
    }
```

Output:- 

```bash
Ran 1 test for test/InterestModel.t.sol:InterestModelTest
[PASS] test_InterestCalculationBug() (gas: 16681)
Logs:
  --- PoC: Interest Calculation Bug ---
  Buggy Interest Calculated:   11
  Correct Interest Expected:   199
  Loss of Yield:               187
```

As you can see in the result, there is huge difference in expected interest and actual interest that calculated.


> I hope this explanation helps you. Keep in mind everytime while auditing always deep dive and think about interest calulation. Check if it is calculated correctly? if yes then think about edge case that might break interest calculation.
