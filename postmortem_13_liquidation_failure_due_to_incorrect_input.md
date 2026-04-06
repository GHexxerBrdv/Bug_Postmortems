welcome to another bug postmortem. in this postmortem i will explain how an incorrect token address can result in failure while liquidating a user.

This bug was found on Lend contest on sherlock. 

Let's understand the bug and get initial intution of it. 

the Lend is crosschain lending and borrowing protocol. consider it as initial info to get the idea of bug class. otherwise the bug is simple one but due to it system will misbehave.

Here the problem was in cross chain liquidation, Where borrower borrowes chain B token by calling borrowCrossChain function on chain A. And you should know that to borrow something you have to put in some collateral. Here the borrower put collateral on chain A and borrow on chain B using that collateral.

Now suppose there is such condition when collateral price drops unexpectedly and borrower position becomes underwater. in that case liquidator try to liquidate the borrower.

> the liquidation is not part of this postmortem, however you can research on your own about it.

first understand that how borrow & liquidation flow works in the protocol.

Borrow flow:

1. On Chain A (source chain), calculate the collateral value and send it.
2. On Chain B (destination chain), borrow the asset and register it to crossChainCollateral.
3. On Chain A, register the debt to crossChainBorrow.

During this process, the underlying token of the source chain and the lToken of the destination chain are used for `srcToken` and `dstlToken` of `crossChainCollateral` and `crossChainBorrow`.


Liquidation flow:

1. Initiation on Chain B.
2. Seizure on Chain A (the source chain).
3. Repayment on Chain B (the destination chain), or handle failure.
4. Matching(DestRepay) on Chain A.

> Here the main bug was thet in Cross-Chain Liquidation Flow Step 3, during the identification of crossChainCollaterals, the srcToken field incorrectly uses the destination chain's borrowed token instead of the source chain's borrowed token.

See following code to understanf the parameter

```solidity
    function _send(
        uint32 _dstEid,
        uint256 _amount,
        uint256 _borrowIndex,
        uint256 _collateral,
        address _sender,
        address _destlToken,
        address _liquidator,
        address _srcToken,
        ContractType ctype
    ) ...
```

now see following actual usage

```solidity
    function borrowCrossChain(uint256 _amount, address _borrowToken, uint32 _destEid) external payable {
        ...
        address destLToken = lendStorage.underlyingToDestlToken(_borrowToken, _destEid);
        ...
        _send(
            _destEid,
            _amount,
            0, // Initial borrowIndex, will be set on dest chain
            collateral,
            msg.sender,
            destLToken,
            address(0), // liquidator
151:        _borrowToken,
            ContractType.BorrowCrossChain
        );
    }
```

here you can see that `_borrowToken` is used as `srcToken` in parameter. 

but the actual token which was used as collateral on chain A was different then borrow token on chain B.

> However if both token are same then it is also an issue, because their corresponding addresses will be different in each chain.

Here borrowed token on chain B is incorreclty used as collateral token in chain A to seize. However this incorrect token address will result in incorrect value read and lead to failure in finding the borrow position and potentialy cause the liquidation failure.

> Always keep in mind while auditing smart contract, that in the protocol how tokens are moving through the functions and how they are managed. What are the validation on passed argument in each function. if there is something not done properly then there is an issue.