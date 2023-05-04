# Findings

## High

### 1. RewardThrottle.checkRewardUnderflow() might track the cumulative APRs wrongly.

**Underlying Problem** : _Calculation Error_

The Malt contract has a function called RewardThrottle.checkRewardUnderflow() which might not calculate APRs correctly. This could cause the calculation of cashflowAverageApr to be wrong and result in unexpected changes to targetAPR. This happens because another function called _sendToDistributor() is not updating APRs when it should. To fix this, the recommended solution is to update APRs even when there is no reward to send.

### 2. RewardThrottle: If an epoch does not have any profit, then there may not be rewards for that epoch at the start of the next epoch.

**Underlying Problem** : _2 function that call the same function but one function has additional calculations or function calls_

This describes an issue in the smart contract code for RewardThrottle. If an epoch does not generate any profit, there may not be rewards for that epoch at the start of the next epoch. The code includes two functions, checkRewardUnderflow and fillInEpochGaps, that both call the _fillInEpochGaps function to fill in the state of the previous epoch without profit. However, checkRewardUnderflow requests rewards from the overflowPool and distributes them, while fillInEpochGaps does not. As a result, if an epoch does not generate any profit, the next epoch will have a reward if checkRewardUnderflow is called, but no reward if fillInEpochGaps is called. Calling the populateFromPreviousThrottle function can also cause epochs without profit to lose their rewards. The recommended mitigation steps include removing the fillInEpochGaps function or only allowing it to be called when the contract is not active.

### 3. Manipulation of livePrice to receive defaultIncentive in 2 consecutive blocks

**Underlying Problem** : _Manipulation of price to receive more incentives_

An attacker can manipulate the StabilizerNode contract in the Malt project to receive default incentives in two consecutive blocks by manipulating the _validateSwingTraderTrigger() function, which is used to validate and start the priming using livePrice. The attacker can use a flash loan to manipulate the livePrice to be larger than the entryPrice and call stabilize() to receive the incentive again, then repay the flash loan. The recommended mitigation steps include not giving incentives for the caller or resetting the primedBlock at least after primedWindow blocks.

### 4. SwingTraderManager.addSwingTrader will push traderId with active = false to activeTraders

**Underlying Problem** : _adding the same thing twice to mess up calculations because of ignoring a condition_

The code describes a vulnerability in the SwingTraderManager contract, where if the active parameter of addSwingTrader is set to false, the traderId is still pushed to activeTraders. This causes the data associated with the trader to be calculated twice in the getTokenBalances() and calculateSwingTraderMaltRatio() functions, resulting in incorrect data and potential issues with syncGlobalCollateral(), MaltDataLab.getRealBurnBudget(), and getSwingTraderEntryPrice(). Additionally, if toggleTraderActive is called on the traderId, only one of the two identical traderIds in activeTraders will be removed, causing the inactive trader to still participate in the calculations. It is recommended to modify the code so that the traderId is only pushed to activeTraders if active is true.

### 5. _distributeProfit will use the stale globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio(), which will result in incorrect profit distribution

**Underlying Problem** : _stale prices because of not calling a function before calling another function_

The _distributeProfit() function in the contract uses stale data from globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio() when distributing profits, which can result in incorrect profit distribution. The latest globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio() needs to be used to ensure that profits are distributed correctly. The two calls to handleProfit() in the contract do not call syncGlobalCollateral() to synchronize the data in globalIC, which uses the data in getCollateralizedMalt(), including the collateralToken balance in overflowPool/swingTraderManager/liquidityExtension and the malt balance in swingTraderManager.
Before handleProfit() is called by StabilizerNode.stabilize, checkAuctionFinalization() is called to liquidityExtension.allocateBurnBudget(), which transfers the collateralToken from liquidityExtension to swingTrader, making the data in globalIC stale. Similarly, swingTraderManager.sellMalt() will exchange malt for collateralToken and also make the data in globalIC stale. The same goes for dexHandler.sellMalt(), which exchanges malt for collateralToken. This stale data will result in larger lpCut and smaller swingTraderCut values.
To mitigate this issue, it is recommended to call syncGlobalCollateral() to synchronize the data in globalIC before calling handleProfit().

### 6.StabilizerNode.stabilize uses stale GlobalImpliedCollateralService data, which will make stabilize incorrect

**Underlying Problem** : _stale prices because of not calling a function before calling another function_

The review identifies a vulnerability in the StabilizerNode.stabilize function of the Malt protocol. The function uses stale data from the GlobalImpliedCollateralService to calculate the collateralRatio, which can result in incorrect stabilization of the Malt token. The review recommends calling impliedCollateralService.syncGlobalCollateral() before calling maltDataLab.getActualPriceTarget to ensure that the data used for stabilization is up-to-date. The review suggests that this vulnerability is high risk, as stabilizing with incorrect data can cause the Malt token to be depegged from its target value.

## Medium

### 1. priceTarget is inconsistent in StabilizerNode.stabilize

**Underlying Problem** : _Wrong calculation because of not imposing a condition_

This is an issue related to the StabilizerNode contract in the MALT project. The priceTarget variable is inconsistent in the stabilize function, meaning that the function may either sell malt or start an auction instead of doing what it is intended to do. This can lead to incorrect decisions and potentially cause issues with the project. The recommended mitigation steps involve using the same logic as the _shouldAdjustSupply function for priceTarget in stabilize.

### 2. The latest malt price can be less than the actual price target and StabilizerNode.stabilize will revert

- Here, the function should revert but it should be a custom error instead of subtraction overflow

**Underlying Problem** : _revert of function if some price is less than a certain threshold_

This is a technical issue that was identified in the Ethereum smart contract StabilizerNode.sol. The issue relates to the function StabilizerNode.stabilize. The function is designed to stabilize the price of a particular asset when it deviates from a target price. The target price is calculated using the maltDataLab.getActualPriceTarget() function, which fetches the latest price target from the Malt Data Lab.
The issue occurs when the latest malt price is less than the actual price target. In this scenario, the StabilizerNode.stabilize function will revert, resulting in a failed transaction. This issue was reported by two users, and a proof of concept was provided to demonstrate the problem.
The root cause of the issue is the incorrect assumption that latestSample >= priceTarget. While exchangeRate is the malt average price during priceAveragePeriod, latestSample is one of the malt prices, and it is possible for it to be less than both exchangeRate and priceTarget. This results in the StabilizerNode.stabilize function reverting.
To mitigate this issue, the recommended solution is to use a different formula for minThreshold when priceTarget > latestSample. Specifically, the formula minThreshold = latestSample + (((priceTarget - latestSample) * sampleSlippageBps) / 10000) should be used. This will ensure that the StabilizerNode.stabilize function does not revert in scenarios where the latest malt price is less than the actual price target.
Overall, this issue highlights the importance of careful consideration and testing of Ethereum smart contracts to ensure that they function correctly and securely. It also demonstrates the value of community-driven bug reporting and collaboration to identify and resolve issues in a timely and effective manner.

### 3. LinearDistributor.declareReward can revert due to dependency of balance

**Underlying Problem** : _DOS due to dependency of balance in the contract for a particular token_

The issue was raised by hansfriese and identifies a vulnerability in the contract's declareReward function that could allow a denial-of-service (DOS) attack.
The issue explains that if an attacker sends enough collateral tokens to the contract to increase the balance above a certain level, the declareReward function will revert and cause a temporary DOS. This is because the function contains a check that will forfeit any excess balance beyond a certain buffer requirement. If the function finds that the balance is greater than the buffer requirement, it will call another function called _forfeit, which requires that the amount forfeited is less than or equal to the value of declaredBalance. If an attacker is able to send enough tokens to cause a balance increase that exceeds the value of declaredBalance, then the _forfeit function will revert and cause a temporary DOS.
However, the issue goes on to state that if the attacker is able to increase the balance enough to cover all reward amounts in vesting, then the declareReward function will always revert and cause a permanent DOS. This is because the function sends vested rewards to the liquidity mine before calling the _forfeit function. Since the vested amount will increase over time, if the attacker can increase the balance enough to exceed the total vested amount, then the declareReward function will always revert and cause a permanent DOS.
The recommended mitigation step is to track the collateral token balance and add sweep logic for unused collateral tokens in the contract. This would help prevent the balance from exceeding the value of declaredBalance and reduce the risk of a DOS attack.
Overall, this issue highlights the importance of thoroughly testing smart contracts for vulnerabilities and implementing appropriate mitigation steps to address any issues found.

### 4. SwingTraderManager.swingTraders() shoudn’t contain duplicate traderContracts.

**Underlying Problem** : _multiple accounts/contract can exist for a single person and this can cause miscalculations of rewards or price as duplicates are not expected_

the issue is that the swingTraders() function should not contain duplicate traderContract items. The problem is that several functions, including buyMalt() and sellMalt(), work according to the balance of each trader. If swingTraders() includes duplicate traderContract items, these functions would not work as expected.
The text provides a code snippet that shows that the program does not validate each trader to have a unique traderContract. This means that the same traderContract might have two or more traderIds, which could result in one trader contract being counted twice and receiving more shares than it can manage.
To address this issue, the recommended mitigation steps include adding a new mapping called activeTraderContracts to check if the contract has already been added and ensuring that each trader contract is added only once. By doing so, the program can make sure that there are no duplicate traderContract items in the swingTraders() function, and other functions such as buyMalt() and sellMalt() can work properly.

### 5. StabilizerNode.stabilize() should update lastTracking as well to avoid an unnecessary incentive.

**Underlying Problem** : _unnecessary minting of incentives due to not updating/tracking the last time the function was called_

The submitter of the issue, hansfriese, suggests that this function should update a variable called lastTracking, to avoid paying unnecessary incentives to track a pool.
The trackPool() function within the codebase pays an incentive per trackingBackoff to ensure pool consistency. However, if stabilize() was recently called, paying this incentive would be unnecessary. The reason for this is that stabilize() function also tracks the pool, meaning that the pool would already be up to date. This redundancy could be avoided by updating the lastTracking variable within the stabilize() function. This would ensure that the trackPool() function only pays incentives when necessary.
The recommended mitigation step, proposed by the submitter of the issue, is to update lastTracking within the stabilize() function. This would ensure that incentives are only paid when tracking the pool is actually necessary.

### 6. Average APRs might be calculated wrongly after calling populateFromPreviousThrottle().

**Underlying Problem** : _APR calculation problem if an admin function is called with params that influence the APR rate_

Calling a particular function, populateFromPreviousThrottle(), may cause the calculation of APRs to be incorrect, which can lead to the target APR being changed unexpectedly.
The report provides a proof of concept in the form of an excerpt from the software code, which shows that the epoch state struct contains a cumulativeCashflowApr element, and that cashflowAverageApr is used to adjust the targetAPR in the updateDesiredAPR() function. The populateFromPreviousThrottle() function is an administrative function that changes the activeEpoch and the relevant epoch state using the previous throttle. The report highlights that the activeEpoch is likely to be increased inside this function.
The problem occurs when epoch < _activeEpoch + smoothingPeriod because state[epoch].cumulativeCashflowApr and state[epoch - smoothingPeriod].cumulativeCashflowApr will be used for cashflowAverageApr calculation. As a result, the cumulativeCashflowApr of the original epoch and the newly added epoch will be used together and cashflowAverageApr might be calculated incorrectly. This can lead to the target APR being changed unexpectedly, which can have serious implications.
The report suggests a mitigation step of adding a check to the populateFromPreviousThrottle() function to ensure that epoch - _activeEpoch > smoothingPeriod. This will prevent the problem from occurring and ensure that the calculation of APRs is accurate.

### 7. RewardThrottle._sendToDistributor() reverts if one distributor is inactive.

**Underlying Problem** : _array of people are send rewards but because of a condition, some people won't get the reward, meaning the function will revert_

This is a submission regarding a finding in the code of a project called MALT. The function RewardThrottle._sendToDistributor() reverts if one distributor is inactive. The function distributes the rewards to several distributors according to their allocation ratios. LinearDistributor.declareReward() has an onlyActive modifier and it will revert in case of inactive. As a result, RewardThrottle._sendToDistributor() will revert if one distributor is inactive rather than working with active distributors only. The recommended mitigation step is to continue to work with active distributors in _sendToDistributor().

### 8. LinearDistributor.declareReward() might revert after changing vestingDistributor.

**Underlying Problem** : _underflow error_

The issue is that the declareReward() function may fail due to an underflow error when changing the vestingDistributor, which could cause the function to revert. The problem is that the function calculates the netVest and netTime by subtracting the previous amount and time, but there is no guarantee that the vested amount of the new vestingDistributor is greater than the previously saved amount after changing the distributor. The recommended mitigation step is to also change the previous amounts while updating the distributor.

### 9. Repository._removeContract() removes the contract wrongly.

**Underlying Problem** : _removing array object wrongly due to miscalculation of index_

 The report identifies an issue with the _removeContract() function in the Repository contract. This function is responsible for removing a contract from the repository, but after removing the contract, the function updates the contracts array incorrectly. As a result, the array contains the wrong contract names, which can cause problems for other parts of the application that rely on the contracts array.
To provide evidence of the issue, the report includes a proof of concept that shows how the function updates the array incorrectly. Specifically, the function uses the already changed index (which is set to 0) and replaces the last name with the 0 index all the time. This leads to the contracts array still containing the removed name and removing the valid name at index 0, causing confusion and potential errors for users.
To mitigate this issue, the report provides a recommended mitigation step. The mitigation step involves using the original index instead of the already changed index to update the contracts array. This will ensure that the correct contract names are removed from the array and that the array remains consistent with the repository.


### 10. StabilizerNode.stabilize may use undistributed rewards in the overflowPool as collateral

**Underlying Problem** : _calculating when you already have undistributed rewards. calling a function before calculating/calling another function recommended_

in the StabilizerNode.stabilize function of the Malt protocol. The stabilize function is a core function of the protocol, and it is responsible for stabilizing Malt's price to the target price. The function calculates the SwingTraderEntryPrice and ActualPriceTarget, which are critical for maintaining the peg of Malt.
The vulnerability arises due to the function using the global collateral to calculate the SwingTraderEntryPrice and ActualPriceTarget, including the balance of collateral tokens in the overflowPool. The issue occurs when there are some gap epochs in the RewardThrottle, and their rewards are not distributed from the overflowPool. When the stabilize function is called, it treats the undistributed rewards in the overflowPool as collateral, thus making the global collateral ratio larger than it should be. As a result, the results of maltDataLab.getActualPriceTarget/getSwingTraderEntryPrice will be incorrect, and this can cause the stabilize function to be incorrect, which could lead to Malt being depegged.
The recommended mitigation for this vulnerability is to call RewardThrottle.checkRewardUnderflow at the beginning of StabilizerNode.stabilize to distribute the rewards in the overflowPool, then call impliedCollateralService.syncGlobalCollateral() to synchronize the latest data.


### 11. RewardThrottle.setTimekeeper: If changing the timekeeper causes the epoch to change, it will mess up the system

**Underlying Problem** : _reward miscalculation due to change of a variable by a priviledged account_

The text is describing a vulnerability in a smart contract function called setTimekeeper. If the epoch of the new timekeeper is changed, it can cause issues with the rewards system. If the epoch increases, large rewards can be distributed immediately. If the epoch decreases, the current epoch may not be able to receive rewards. The recommendation is to only allow setTimekeeper to be called when the rewards system is not active. However, the client eventually removed the setTimekeeper methods as it was unnecessary.

### 12. Value of totalProfit might be wrong because of wrong logic in function sellMalt()

**Underlying Problem** : _variable updated incorrectly because it is calculated after a return statement_

This text is discussing an issue with the SwingTraderManager contract's totalProfit variable. The variable is meant to keep track of the total profit made during the sellMalt() function, but the logic for accounting is incorrect. This can result in the totalProfit variable having an incorrect value, which can in turn affect other contracts that use it. The recommended mitigation step is to update the logic so that totalProfit is updated before the dust check in the sellMalt() function.


### 13. Function stabilize() might always revert because of overflow since Malt contract use solidity 0.8

**Underlying Problem** : _a combination of never overflow + desired overflow because of using two different versions of solidity_

This document is a software development report that highlights an issue with the Malt contract. it discusses the problem with the stabilize() function in the contract. The issue is related to the overflow that can occur when the stabilize() function calls the MaltDataLab contract to get state.
The problem arises because the MaltDataLab contract uses Solidity 0.8, which will revert when overflow happens. On the other hand, the Uniswap V2 pool uses Solidity 0.5.16, which does not revert when overflow happens. The report includes a proof of concept to demonstrate how the problem occurs when the priceCumulative value overflows in the Uniswap pool, causing the stabilize() function to always revert.
The report recommends the use of the unchecked block to handle overflow calculations in Uniswap V2. The recommended mitigation steps also include matching the overflow calculation handling in the Uniswap V2 codebase with the Malt contract codebase. This will ensure that the contract can handle the overflow correctly, without causing the stabilize() function to always revert.

### 14. RewardThrottle.populateFromPreviousThrottle may be exposed to front-run attack

**Underlying Problem** : _two function does the same thing but one is access controlled while the other is not. the function is called at first but an attacker frontrun by using the second function to cause harm_

 The vulnerability concerns the "populateFromPreviousThrottle" function, which allows an administrator to use epoch data from a previous version of the contract to populate the state from the active epoch to the current epoch in the current version of the contract. However, the function may be vulnerable to front-run attacks, which means that a malicious user could exploit this function to update the state for future epochs, potentially causing harm.
The report explains that another function in the "RewardThrottle" contract, "_fillInEpochGaps", has essentially the same function as "populateFromPreviousThrottle", and thus a malicious user could call this function to front-run "populateFromPreviousThrottle". The only difference between the two functions is that "populateFromPreviousThrottle" can make the epoch and activeEpoch greater than the current epoch, whereas "_fillInEpochGaps" sets activeEpoch to the current epoch, thereby invalidating "populateFromPreviousThrottle" for future updates.
The report recommends that if "populateFromPreviousThrottle" is used to initialize the state in the current RewardThrottle, it should be called only once during contract setup. This would prevent any future updates through this function and reduce the potential risk of a front-run attack.


### 15. LinearDistributor.declareReward: previouslyVested may update incorrectly, which will cause some rewards to be lost

**Underlying Problem** : _miscalculation and wrong updation_

This text is a software bug report related to a function called LinearDistributor.declareReward. The bug was found by someone named cccz, who explains that the previouslyVested variable can be updated incorrectly, causing some rewards to be lost. This is because the rewards to distribute (calculated using netVest) cannot exceed the balance, and any part of the rewards in netVest that exceeds the balance will be lost. At the end of the function, previouslyVested is directly assigned to currentlyVested instead of using the Vested adjusted according to distributed, which means that the previously lost rewards will also be skipped in the next distribution. The report suggests a mitigation step, which is to adapt previouslyVested based on distributed.

### 16. MaltRepository._revokeRole may not work correctly

**Underlying Problem** : _overriding oz implementation can cause problems. be careful when doing something like that_

The report highlights a potential issue with the _revokeRole function, which may not work correctly due to a validation check in the hasRole function.
The MaltRepository contract inherits from AccessControl and adds validation of validRoles to the hasRole function. This means that even if super.hasRole(role, account) == true, if validRoles[role] == false, hasRole will return false. This can cause issues with the _revokeRole function because, if a role is removed and its corresponding validRoles flag is set to false, the hasRole function will return false for that role, even if the user still has that role. As a result, _revokeRole function fails to revoke the user's role.
The report also highlights that the renounceRole and _transferRole functions may also be affected due to the same issue. In particular, the _transferRole function may cause both Alice and Bob to have the role in case of a transfer, if validRoles[role] == false.
The report recommends overriding the renounceRole and removeRole functions in the MaltRepository contract and modifying them to include a check for the role's validRoles flag. The suggested modifications will add a require statement for validRoles[role] to ensure that the role is still valid before revoking it.
