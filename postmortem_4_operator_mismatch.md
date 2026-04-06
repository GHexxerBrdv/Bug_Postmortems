Welcome to postmortem 4. in this postmortem i will explain how a small mismatch in operator cause incorrect liquidation.

This bug was found in `Monolith Stablecoin Factory`. where if a user borrow stablecoin, then anyone can liquidate the borrower even if they have healthy position.

Look in these parts of function where health factor is calculated and compared.

```solidity
        (uint price, bool reduceOnly, ) = getCollateralPrice();
        require(!reduceOnly, "Reduce only");
        uint borrowingPower = price * _cachedCollateralBalances[account] * collateralFactor / 1e18 / 10000;
@>        require(debtBalance <= borrowingPower, "Solvency check failed");
    }


    function getLiquidatableDebt(uint collateralBalance, uint price, uint debt) internal view returns(uint liquidatableDebt){
        uint borrowingPower = price * collateralBalance * collateralFactor / 1e18 / 10000;
@>        if(borrowingPower > debt) return 0; // <<<--- can still liquidate if borrowingPower == debt
        // liquidate 25% of the total debt
        liquidatableDebt = debt / 4; // 25% of the debt
        // liquidate at least MIN_LIQUIDATION_DEBT (or the entire debt if it's less than MIN_LIQUIDATION_DEBT)
        if(liquidatableDebt < MIN_LIQUIDATION_DEBT) liquidatableDebt = debt < MIN_LIQUIDATION_DEBT ? debt : MIN_LIQUIDATION_DEBT;
    }
```

If you see in one place `>` is used and in other place `>=` is used for comparing borrowing power against actual borrowed stablecoin. 

Here the use of `>` is correct operator which should be used. If debt is stricly less than borrowing power then it is fine. this will return 0. but while if debt is less then or equal to borrowing power, then that part of code is allowing liquidation. that results in immidiate liquidation after borrow of stablecoin.

consider following case:

1. Assume that collateral factor is 50%
2. An user adds 100$ collateral and borrows 50$ debt -> borrowing power = 50%
3. The position can be liquidated immediately after creation


This was the major flaw in the protocol due to simple incorrect use of operator while comparing health factor.

> Keep in mind, everytime while auditing smart contracts that consist liquidation mechanism, do not afraid to go deep into rabbit hole. There can be always a bug and exploit possible.