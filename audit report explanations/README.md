# Audit Report Explanations

This file contains explanations of different vulnerabilities in different audit reports. 

## High

### 1. Using chainlink oracle with Optimistic rollups can be troublesome if valid checks are not implemented
The vulnerability reported in the audit report (Issue M-3) is related to the _validateAndGetPrice() function in the BondChainlinkOracle smart contract. The issue is that the function does not check if the Arbitrum L2 sequencer is down in Chainlink feeds. This vulnerability could potentially be exploited by malicious actors to gain an unfair advantage by manipulating the prices perceived as fresh when the sequencer is down Source.

To address this issue, you can follow the recommendations provided in the audit report, which refers to the Chainlink documentation on L2 sequencer feeds. The example code provided in the Chainlink documentation shows how to handle the situation when the sequencer is down.

You can modify the _validateAndGetPrice() function to include an additional check for the sequencer status using the isSequencerStatusOk() function from the Chainlink example code. The updated function would look like this:

```solidity
function _validateAndGetPrice(AggregatorV2V3Interface feed_, uint48 updateThreshold_)
    internal
    view
    returns (uint256)
{
    // Get latest round data from feed
    (uint80 roundId, int256 priceInt, , uint256 updatedAt, uint80 answeredInRound) = feed_
        .latestRoundData();

    // Check if Arbitrum L2 sequencer is down in Chainlink feeds
    require(isSequencerStatusOk(feed_), "BondOracle: Sequencer down");

    // Validate chainlink price feed data
    // 1. Answer should be greater than zero
    // 2. Updated at timestamp should be within the update threshold
    // 3. Answered in round ID should be the same as the round ID
    if (
        priceInt <= 0 ||
        updatedAt < block.timestamp - uint256(updateThreshold_) ||
        answeredInRound != roundId
    ) revert BondOracle_BadFeed(address(feed_));
    return uint256(priceInt);
}
```

This additional check will help ensure that the price data used by the smart contract is not falsely perceived as fresh when the Arbitrum L2 sequencer is down.

To further mitigate the risk of smart contract vulnerabilities, it's crucial to conduct regular audits, perform internal security checks, and leverage security audit tools pixelplex.io. You can also consider following best practices and guidelines provided in various resources, such as 0xpredator.medium.com, for auditing smart contracts and understanding common vulnerabilities.

### 2. If for loop check if atleast one possibility of revert

when a function uses an array, if there is a possibility of a revert in atleast one of the iteration the entire for loop fails and may lead to DOS.

Example: https://github.com/code-423n4/2021-05-nftx-findings/issues/46

## Medium

### 1. Using checks inside for loops that accepts arrays

The bug revolves around contracts which perform validations in `for` loops.

rom initial looks the contract seems to be fine. 

It simply validates some signatures and if those signatures are valid the contract transfers funds to the caller

If any one of those signatures is invalid then the txn is reverted

However the exploit occurs when an empty signature array is passed to the contract

In that case the 'for' loop is not executed (length = 0) and the execution flow goes directly to the fund transfer statement

Essentially the contract gets rekt

While the non-execution of `for` loop when the length of array it is iterating over is zero seems obvious, somehow a few similar bugs went passed the development cycle and got caught during the audit

### 2. returnData for call not checked

function should check returnData.length == 1 before decoding(if decoding is implemented). Otherwise, if it returns no return data, the abi.decode call will revert

### 3. If ERC721 and ERC1155 are handled in the same function check if they are properly implemented

improper implementation may cause many issue and may even lead to loss of funds

## Low