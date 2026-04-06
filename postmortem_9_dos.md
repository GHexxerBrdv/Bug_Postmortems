Welcome to another postmortem. in this deep dive i will explain how protocol assumption and use of native tokens can dos the fee collection. This is perticular case not general. but this is very crusial case where protocol assumption breaks.

This bug as found on `Super DCA Liquidity Network`. the protocol was using uniswap v4 pools to add/remove liquidity. However there was fee collection mechanism to collect fees.

The user has to register a uniswap v4 pool which must consist of SuperDCA token in pair. Also there must me liquidity in that pool. There was a function that collects fees for owner of the protocol from the positions in the pool. 


Have a look in following function

```solidity
    function collectFees(uint256 nfpId, address recipient) external override {
        _checkOwner();

        // Validate the NFT ID and recipient address
        if (nfpId == 0) revert SuperDCAListing__UniswapTokenNotSet();
        if (recipient == address(0)) revert SuperDCAListing__InvalidAddress();

        // Retrieve the position's pool information
        (PoolKey memory key,) = POSITION_MANAGER_V4.getPoolAndPositionInfo(nfpId);
        Currency token0 = key.currency0;
        Currency token1 = key.currency1;

        // Record token balances before fee collection
@>      uint256 balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient);
        ....
```

You can see that we caches the balance of recepient of one of the token of pool. In this case the token is other then DCA token. 

Now uniswap v4 can have any token pair pool. Suppose a user create pool of SuperDCA/ETH and register it in the protocol, Now there is a catch, uniswap v4 consider ETH as `address(0)` because ETH is not any ERC20 token. that means ETH does not have any balanceOf function. However as shown above in the function while caching the balance of recepient it is calling balanceOf function on ETH. But this will revert. results in making fees collection dosed forever. 

> Always keep in mind, while auditing smart contract consider the usage of native token. and ask what if i use native token? Then how does the protocol deal with it? What are the impact etc.

I hope this postmortem helped you in your learning.

Thank you,
GB-53F8.