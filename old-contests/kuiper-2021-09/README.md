# Kuiper Findings

## High

### [H-01] Re-entrancy in settleAuction allow stealing all funds
Submitted by cmichel

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
Submitted by WatchPug

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Basket.sol#L102-L108

Given the auctionBurn() function will _burn() the auction bond without updating the ibRatio. Once the bond of a failed auction is burned, the proportional underlying tokens won’t be able to be withdrawn, in other words, being frozen in the contract.

#### GPT explaination

### 

#### GPT explaination

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

1. centralization risk

2. no check if the old pendingWeight.tokens and pendingWeight.weights is equal to the new ones

3. no implementation if 