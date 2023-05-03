# Kuiper Findings

## High

### [H-01] Re-entrancy in settleAuction allow stealing all funds

Note that the Basket contract approved the Auction contract with all tokens and the settleAuction function allows the auction bonder to transfer all funds out of the basket to themselves. The only limiting factor is the check afterwards that needs to be abided by. It checks if enough tokens are still in the basket after settlement:

```solidity
// this is the safety check if basket still has all the tokens after removing arbitrary amounts
for (uint256 i = 0; i < pendingWeights.length; i++) {
    uint256 tokensNeeded = basketAsERC20.totalSupply() * pendingWeights[i] * newRatio / BASE / BASE;
    require(IERC20(pendingTokens[i]).balanceOf(address(basket)) >= tokensNeeded);
}
```
The bonder can pass in any inputTokens, even malicious ones they created. This allows them to re-enter the settleAuction multiple times for the same auction.

Calling this function at the correct time (such that bondTimestamp - auctionStart makes newRatio < basket.ibRatio()), the attacker can drain more funds each time, eventually draining the entire basket.

#### GPT explaination


### [H-02] Basket.sol#auctionBurn() A failed auction will freeze part of the funds

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Basket.sol#L102-L108

Given the auctionBurn() function will _burn() the auction bond without updating the ibRatio. Once the bond of a failed auction is burned, the proportional underlying tokens won’t be able to be withdrawn, in other words, being frozen in the contract.

#### GPT explaination
This is a report on a code review of a smart contract called Basket.sol. The report is created by WatchPug, who has identified a potential issue with the auctionBurn() function in the contract. The report highlights that if an auction fails and the bond is burned, the proportional underlying tokens may become frozen in the contract and unable to be withdrawn.

To support this claim, the report includes a proof of concept that shows what happens with the current configuration of the contract, which is a basket token with an ibRatio of 1e18, a factory bondPercentDiv of 400, a total supply of 400, two tokens (BTC and ETH) with equal weights of 1, and a failed auction that burns one basket token. The report shows that after the failed auction, the ibRatio remains 1e18, but the total supply reduces to 399. Burning one basket token will only get back 1 BTC and 1 ETH, which means that 1 BTC and 1 ETH are frozen in the contract.

To address this issue, the report recommends a mitigation step to change the code of the auctionBurn() function. The recommended change involves updating the ibRatio when burning the auction bond, to prevent any proportional underlying tokens from becoming frozen in the contract.

Overall, this report highlights a potential issue with the Basket.sol contract and provides a recommended solution to mitigate the problem. The report is useful for developers and auditors who are reviewing the contract and looking for ways to improve its functionality and security.


### [H-03] Reentrancy in settleAuction(): malicious publisher can bypass index timelock mechanism, inject malicious index, and rug the basket

The `settleAuction()` function calls `withdrawBounty()` before setting `auctionOngoing = false`, thereby allowing reentrancy.

**A malicious publisher can bypass the index timelock mechanism and publish new index which the basket’s users won’t have time to respond to. At worst case, this means setting weights that allow the publisher to withdraw all the basket’s underlying funds for himself, under the guise of a valid new index.

#### GPT explaination
This document is a report on a potential vulnerability in a smart contract's `settleAuction()` function, which could be exploited by a malicious publisher to bypass the index timelock mechanism and inject a malicious index.

The issue lies in the fact that the `settleAuction()` function calls `withdrawBounty()` before setting `auctionOngoing = false`, which allows for reentrancy. This means that a malicious publisher could propose a new valid index and bond the auction. When settling the auction, the publisher could execute a series of steps to exploit this vulnerability, including adding a bounty of an ERC20 contract with a malicious `transfer()` function, settling the valid new weights correctly using `settleAuction()` with the correct parameters and passing the malicious bounty id, and calling `withdrawBounty()` which upon transfer will call the publisher’s malicious ERC20 contract.

The attacker can then call the `settleAuction()` function again, with empty parameters, and since the previous call's effects have already set all the requirements to be met, `settleAuction()` will finish correctly and call `setNewWeights()`, which will set the new valid weights and set `pendingWeights.pending = false`. Still inside the malicious ERC20 contract transfer function, the attacker will now call the basket's `publishNewIndex()`, with weights that will transfer all the funds to them upon their burning of shares. This call will succeed in setting new pending weights as the previous step set `pendingWeights.pending = false`.

Now that the malicious `withdrawBounty()` has ended, and the original `settleAuction()` is resuming, but now with malicious weights in `pendingWeights` (set in step 6). `settleAuction()` will now call `setNewWeights()`, which will set the basket's weights to be the malicious pending weights. Now `settleAuction` has finished, and the publisher (within the same transaction) will `burn()` all their shares of the basket, thereby transferring all the tokens to themselves.

The impact of this vulnerability is that a malicious publisher can bypass the index timelock mechanism and publish a new index which the basket's users won't have time to respond to. At worst case, this means setting weights that allow the publisher to withdraw all the basket's underlying funds for themselves, under the guise of a valid new index.

The document includes a proof of concept (POC) exploit, which consists of two files: AttackPublisher.sol, to be put under contracts/contracts/Exploit, and ExploitPublisher.test.js, to be put under contracts/test. The recommended mitigation steps to address this vulnerability are to move `basketAsERC20.transfer()` and `withdrawBounty()` to the end of the `settleAuction()` function, conforming with the Checks Effects Interactions pattern.


### 

#### GPT explaination

### 

#### GPT explaination

### 

#### GPT explaination

### 

#### GPT explaination

1. Minting can be done without any transfer of tokens in the factory contract.

2. Fake Erc20 tokens can be used to mint tokens

3. Accounting problem if licensefee is to high

4. Basket can be initialized again by anyone


## Medium

### [M-01] Use safeTransfer instead of transfer
Submitted by hack3r-0m, also found by itsmeSTYJ, JMukesh, leastwood, and shenwilly

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Auction.sol#L146

transfer() might return false instead of reverting, in this case, ignoring return value leads to considering it successful.

### [M-01]

1. centralization risk

2. no check if the old pendingWeight.tokens and pendingWeight.weights is equal to the new ones

3. no implementation if 