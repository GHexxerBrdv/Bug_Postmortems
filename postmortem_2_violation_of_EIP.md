# EIP4626 violation

Welcome in postmortem 2. This postmortem is based on EIP4626 violation.

Before diving into the bug. first we need to understand the EIP which is used in the protocol. We should consider its specs while auditing. there can be such case where EIP specs are not following by such function in the protocol. This can be low/medium issue according to the impact and usage of the function.

This bug was present in recent public contest `monolith stablecoin factory` in Sherlock.

there was a vault contract implemented which was extending ERC4626. There was a function called `totalAssets()`. According to ERC4626 this function must not revert. 

see [ERC4626](https://eips.ethereum.org/EIPS/eip-4626)

Following was the function which is not following the EIP standerd

```solidity
    function totalAssets() public view override returns (uint256) {
        return asset.balanceOf(address(this)) + lender.getPendingInterest();
    }
```

as you can see this function returns the addition of the asset held by vault contract and pending interest.

the issue was in `getPendingInterest` function. 

this function is using `try catch` inside it. in solidity if any error occurs and we do not handle it properly then issues can be arise. 

Look the following function

```solidity
    function getPendingInterest() external view returns (uint256 pendingVaultInterest) {
        uint timeElapsed = block.timestamp - lastAccrue;
        if(timeElapsed == 0) return 0;

        uint256 gasBefore = gasleft();
        
        try interestModel.calculateInterest(
            totalPaidDebt,
            lastBorrowRateMantissa,
            timeElapsed,
            expRate,
            getFreeDebtRatio(),
            targetFreeDebtRatioStartBps,
            targetFreeDebtRatioEndBps
        ) returns (uint, uint interest) {
            uint120 localReserveFee = uint120(interest * feeBps / 10000);
            uint120 globalReserveFee = uint120(interest * cachedGlobalFeeBps / 10000);
            // we remove reserve fees from interest before calculating how much to give to stakers
            uint interestAfterFees = interest - localReserveFee - globalReserveFee;
            uint totalStaked = coin.balanceOf(address(vault));
            if(totalStaked < totalPaidDebt) { // this also implies totalPaidDebt > 0 and guards the division below
                // if total staked is less than paid debt, giving all interest to stakers would
                // result in higher supply rate than borrow rate which is undesirable.
                // we cap the supply rate at the borrow rate and give the rest to local reserves.
                uint stakedInterest = interestAfterFees * totalStaked / totalPaidDebt;
                return stakedInterest;
            } else {
                // if total staked is greater than paid debt, we give all interest to stakers
                return interestAfterFees;
            }
        } catch {
            require(gasBefore >= INTEREST_CALCULATION_GAS_REQUIREMENT, "Not enough gas for accrueInterest");
        }
    }
```

this function is calling another function called `calculateInterest` function, which have same usge of try catch in there. However if the function call failes and not enough gas is not provided then this function call will revert the whole transaction. 

However according to EIP4626 specs, `totalAssets()` must not revert. the protocol implemented `totalAssets()` function is reverting, which is opposite of EIP4626. 

This issue was validated as medium, as the protocol readme specified explicitly that any EIP violation is medium issue.


> I hope this will help you to understand how functional dependency can create bug. While auditing any protocol always keep in mind that which functions are dependent on which function. And how it effected by small unexpected cases. This could be your next finding. 