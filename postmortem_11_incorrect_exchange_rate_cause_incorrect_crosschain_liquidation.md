Welcome to another postmortem, in this deep dive i will explein how incorrect exchange rate can cause incorrect calculation of token to seize. in this perticuler bug there are multiple dependecies, like cross chain messaging, cross chain borrow, cross chain exchange rates etc.

This bug was found on sherlock contest `Lend finance`. which is cross chain lending and borrowing protocol enables user to supply/borrow cross chain.

While calculating `seizeTokens`, which is amount of token to seize on chain where the collateral exists while liquidation on borrow chain. Now in case of same chain there is no issue.

But the main problem comes when two different chain is used. While calculating `seizeToken` the protocol use the exchange rate for token. but the problem was that it was using the borrow chain exchange rate for calculation. 

But both collateral and borrow chain could have different exchange rate. That lead to seize incorrect amount of token from collateral chain from the user.

Consider following case

1. An attacker has a $500 BTC loan on the destination chain (Arbitrum), which was originated from the source chain (Ethereum), and is in a liquidatable state.
2. To liquidate the loan, $500 worth of the user’s collateral on the source chain (Ethereum) must be seized.
3. Assume 1 BTC = $1000, and the user has 500 USDC as collateral on Ethereum.
4. The exchange rate of the cUSDC market on Arbitrum (destination chain) is 0.5, while on Ethereum (source chain), it is 0.25.
5. Because the seizure amount is currently calculated on Arbitrum, the contract computes: 500 / 0.5 = 1000 cUSDC.
6. As a result, 1000 cUSDC will be seized on Ethereum. However, with an exchange rate of 0.25, this amounts to only 1000 * 0.25 = 250 USDC — which is insufficient. The correct amount to seize is 500 / 0.25 = 2000 cUSDC.
7. In this case, too little collateral is seized, resulting in a loss to the liquidator. In the opposite scenario, if the exchange rate on Ethereum is higher than that on Arbitrum, more collateral than necessary will be seized, causing a loss to the user.

due to this issue the  incorrect amount of user cTokens will be seized during cross-chain liquidation, leading to financial loss for either the liquidator or the user, depending on the direction of the exchange rate mismatch.

