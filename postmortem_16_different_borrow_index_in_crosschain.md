welcome to another bug postmortem. here we will understand how borrow index could lead to incorrect debt calculation while borrowing cross chain. 

This bus was found on Lend contest on sherlock.

for intution you all might know that we use idexes to keep accounting precise and true while managing linier interest accrual in lending and borrowing. However this works fine if the protocol is just deployed only one chain or in different chain but the chains are not conneted and cummunicating to each other. 

While the Lend protocol is cross chain lending and borrowing protocol. While communicating cross chain there could be huge class of bugs that might result in unexpected behaviour in system. 

one of the flaw is different borrow index in cross chain borrow. this is the main reason that could result in incorrect calculation in debt cross chain.

See the following function

```solidity
    function borrowWithInterest(address borrower, address _lToken) public view returns (uint256) {
        address _token = lTokenToUnderlying[_lToken];
        uint256 borrowedAmount;

        Borrow[] memory borrows = crossChainBorrows[borrower][_token];
        Borrow[] memory collaterals = crossChainCollaterals[borrower][_token];

        require(borrows.length == 0 || collaterals.length == 0, "Invariant violated: both mappings populated");
        // Only one mapping should be populated:
        if (borrows.length > 0) {
            for (uint256 i = 0; i < borrows.length; i++) {
                if (borrows[i].srcEid == currentEid) {
                    borrowedAmount +=
                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } 
```

The root of the issue lies in the use of the local chain's borrowIndex when computing the debt for cross-chain borrows. Specifically, the debt is calculated using the formula:
`(borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;`

This formula assumes that the index on the source chain (where the borrow occurred) is the same as on the destination chain (where the debt is being calculated), which is not necessarily true. Borrow index could be different for two different chains based on market conditions.

`borrowWithInterest` is used inside `getHypotheticalAccountLiquidityCollateral` to calculate users total borrow amount. If borrow amount is calculated less than real, attackers can use this to redeem or borrow more tokens for profit.

> Always keep in mind while auditing smart contract, what values has been used to calculate the funds, ask does this value used is up to date? is it right value to use while calculating funds? etc. If something is wrong there then there is a bug.