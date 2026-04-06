Welcome to another postmortem. in this postmortem i will explain you the bug related to lending protocol where user can provide the collateral but can not borrow using that collateral.

This bug was found in Lend protocol on sherlock.

while borrowing the LToken contract is called to satisy the borrow amount. But before borrow it checks that wether the borrow is allowed ot not. 

```solidity
        // Enter the Compound market
        enterMarkets(_lToken);

        // Borrow tokens
@>      require(LErc20Interface(_lToken).borrow(_amount) == 0, "Borrow failed");



-------


        /* Fail if borrow not allowed */
        uint256 allowed = lendtroller.borrowAllowed(address(this), borrower, borrowAmount);
        if (allowed != 0) {
            revert BorrowLendtrollerRejection(allowed);
        }
```

in `borrowAllowed` it checks if `CoreRouter` provides sufficient collateral to Ltoken by looping throug all whitelisted token.

```solidity
        (Error err,, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(borrower, LToken(lToken), 0, borrowAmount);
        if (err != Error.NO_ERROR) {
            return uint256(err);
        }
```

Now here is the catch, in `getHypotheticalAccountLiquidityInternal` function read the `accountAssets` in `Lendtroller` contract to get collateral info. However the collateral is only added if the specific collateral LToken is entered in market. if not then it will revert.

Now the problem was that while supplying in the protocol, the ltoken was not entered in market. That means everytime the user wants to borrow from the protocol it will revert. 

That means the user was not able to borrow from the collateral they putted in the system. 

This is how it could be mitigated

```solidity
    function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ...

+       // Enter the Compound market
+       enterMarkets(_lToken);

        // Mint lTokens
        require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");

        ...

        emit SupplySuccess(msg.sender, _lToken, _amount, mintTokens);
    }
```

> Always remember while auditing smart contrat, how funds are managed and what are the validation in each function and operation. are those things are doing in right way or wrong? if anything is missing then there could be possibally a bug that could lead to potential exploit.
