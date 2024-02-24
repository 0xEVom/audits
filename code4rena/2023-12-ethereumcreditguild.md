
## [M-01] Inability to withdraw funds for certain users due to `whenNotPaused` modifier in `RateLimitedMinter`

### Lines of code

[RateLimitedMinter.sol#L52](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/rate-limits/RateLimitedMinter.sol#L52)  
[SurplusGuildMinter.sol#L158-L163](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L158-L163)  
[SurplusGuildMinter.sol#L259](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L259)

### Impact
From the contest documentation:
> \[The GUARDIAN role\] is expected to be able to freeze new usage of the protocol, but not prevent users from withdrawing their funds.

However, due to the `whenNotPaused` modifier in the `RateLimitedMinter.mint()` function, users who have `CREDIT` tokens staked on a gauge with a pending `guildReward` will not be able to withdraw their funds. This is because the `SurplusGuildMinter.unstake()` function attempts to mint rewards through the `RateLimitedMinter` in the `getRewards()` function prior to unstaking. If the protocol is paused, this step will fail, causing the transaction to revert and preventing users from withdrawing their funds.

### Proof of Concept
Consider a scenario where Alice has `CREDIT` tokens staked on a gauge with a pending `guildReward`. Then, the protocol is paused and Alice attempts to unstake her tokens by calling the `SurplusGuildMinter.unstake()` function, which in turn calls the `getRewards()` function. This function attempts to mint rewards through the `RateLimitedMinter.mint()` function. However, because the protocol is paused, this function cannot be executed, causing the transaction to revert and preventing Alice from unstaking and withdrawing her funds.

### Tools Used
Manual review

### Recommended Mitigation Steps
Provide an `emergencyWithdraw` method allowing users to withdraw their funds while foregoing rewards when the protocol is paused. This change should be carefully reviewed and tested to ensure it does not introduce other security risks.


## [M-02] No check for sequencer uptime can lead to dutch auctions failing or executing at bad prices

### Lines of code

N/A

### Impact
The `AuctionHouse` contract implements a Dutch auction mechanism to recover debt from collateral. However, there is no check for sequencer uptime, which could lead to auctions failing or executing at unfavorable prices. 

The current deployment parameters allow auctions to succeed without a loss to the protocol for a duration of 10m 50s. If there's no bid on the auction after this period, the protocol has no other option but to take a loss or forgive the loan. This could have serious consequences in the event of a network outage, as any loss results in the slashing of all users with weight on the term. 

Network outages and large reorgs happen with relative frequency. For instance, Arbitrum suffered an hour-long outage just two weeks ago ([source](https://github.com/ArbitrumFoundation/docs/blob/50ee88b406e6e5f3866b32d147d05a6adb0ab50e/postmortems/15_Dec_2023.md)).

### Proof of Concept
Consider the following scenario:

1. A loan is called and an auction is initiated.
2. The network experiences an outage, causing the sequencer to go offline.
3. The auction fails to receive any bids within the 10m 50s window due to the outage.
4. The protocol is forced to take a loss (if there's still a bid after the `midPoint` and before the auction ends) or forgive the loan, both leading to the complete slashing of all users with weight on the term.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, consider integrating an external uptime feed such as [Chainlink's L2 Sequencer Feeds](https://docs.chain.link/data-feeds/l2-sequencer-feeds). This would allow the contract to invalidate an auction if the sequencer was ever offline during its duration. Alternatively, implement a mechanism to restart an auction if it has received no bids.


## [M-03] Inability to unstake if the credit minter buffer is low

### Lines of code

[LendingTerm.sol#L290-L291](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L290-L291)  
[LendingTerm.sol#L302-L303](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L302-L303)  
[LendingTerm.sol#L309-L311](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L309-L311)  
[LendingTerm.sol#L316-L317](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L316-L317)  
[LendingTerm.sol#L323-L330](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L323-L330)  
[GuildToken.sol#L226-L230](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L226-L230)

### Impact
The `LendingTerm.debtCeiling()` function, only called from `GuildToken._decrementGaugeWeight()`, which is itself internally called when unstaking/decreasing gauge weight, contains a mistake in calculation that will prevent users from unstaking from any term if the `creditMinterBuffer` is low. This is because it will always return the `creditMinterBuffer` if it is lower than the `_hardCap` or `_debtCeiling`. The only exception is the case where `_issuance >= debtCeilingBefore`, in which case `debtCeilingBefore` will be returned, which will then fail the `require(issuance <= debtCeilingAfterDecrement)` statement in `_decrementGaugeWeight()` unless in case of strict equality.

It seems that this function was written with another purpose in mind (to limit minting) and then repurposed to fulfil another purpose (to determine how much can be unstaked).

This could lead to a situation where, under heavy protocol usage or simply after a large mint, users are unable to unstake or decrease their gauge weight, regardless of the issuance on the term they're supporting.

### Proof of Concept
Consider the following scenario:

1. The protocol experiences high usage, reducing the `creditMinterBuffer`.
2. A user tries to unstake or decrease their gauge weight from a term they're supporting, which is far from reaching its debt ceiling.
3. The `GuildToken._decrementGaugeWeight()` function is called, which internally calls the `debtCeiling()` function.
4. The `debtCeiling()` function returns the `creditMinterBuffer` as it is lower than the `_hardCap` or `_debtCeiling`.
5. The user is unable to unstake or decrease their gauge weight as the `creditMinterBuffer` is lower than the term's issuance.

Additionally, note that this also opens up a DoS vector (albeit probably beneficial to the protocol) exploitable by continually minting CREDIT and depleting the minting buffer, which will prevent anybody from unstaking or reducing their weight.

### Tools Used
Manual review

### Recommended Mitigation Steps
The  `debtCeiling()` function  needs to be re-architected to ensure it only prevents unstaking based on a term's issuance.


## [M-04] A large enough loss will brick the protocol 

### Lines of code

[ProfitManager.sol#L172-L176](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L172-L176)  
[SimplePSM.sol#L87-L99](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol#L87-L99)

### Impact
The `totalBorrowedCredit` function in the `ProfitManager` contract may revert if the `creditMultiplier` is low. This is because `redeemableCredit` is scaled by the `creditMultiplier`, but `targetTotalSupply` isn't. This can potentially brick core protocol functionality as no one would be able to unstake or borrow.

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L172-L176
```solidity
    /// @notice returns the sum of all borrowed CREDIT, not including unpaid interests
    /// and creditMultiplier changes that could make debt amounts higher than the initial
    /// borrowed CREDIT amounts.
    function totalBorrowedCredit() external view returns (uint256) {
        return
            CreditToken(credit).targetTotalSupply() -
            SimplePSM(psm).redeemableCredit();
    }
```

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol
```solidity
   function redeemableCredit() public view returns (uint256) {
        return getMintAmountOut(pegTokenBalance);
    }

    function getMintAmountOut(uint256 amountIn) public view returns (uint256) {
        uint256 creditMultiplier = ProfitManager(profitManager)
            .creditMultiplier();
        return (amountIn * decimalCorrection * 1e18) / creditMultiplier;
    }
```

The `totalBorrowedCredit` function is reachable from several entry points including `borrow()`, `unstake()`, `updateMintRatio()`, and `applyGaugeLoss()`. If a large enough loss occurs, all these functions will revert, effectively bricking the protocol.

The root cause of this issue is the way `totalBorrowedCredit` is calculated. It subtracts `redeemableCredit` from `targetTotalSupply`. However, `redeemableCredit` is calculated using the `creditMultiplier`, which can decrease in the event of a loss. This is incorrect and was not the intention of the developers, as can be understood from `totalBorrowedCredit()`'s documentation. If the `creditMultiplier` is low enough, `redeemableCredit` can exceed `targetTotalSupply`, causing the subtraction to underflow and the function to revert.

### Proof of Concept
Consider the following scenario:

1. The protocol has 100 CREDIT minted on deployment.
2. Alice mints 200 CREDIT through the PSM, resulting in a `targetTotalSupply` of 300 CREDIT and a `pegTokenBalance` of 200.
3. A loss of 150 CREDIT is applied, causing the `creditMultiplier` to decrease to 0.5.
4. Now, `redeemableCredit` is 400 (≈`pegTokenBalance / creditMultiplier`), which is greater than `targetTotalSupply`.
5. Calling `totalBorrowedCredit` now causes an underflow and reverts.

This scenario is illustrated in the `testTotalBorrowedCredit_breaks` test below:
```solidity
        function testTotalBorrowedCredit_breaks() public {
        assertEq(profitManager.totalBorrowedCredit(), 100e18);

        // psm mint 200 CREDIT
        pegToken.mint(address(this), 200e6);
        pegToken.approve(address(psm), 200e6);
        psm.mint(address(this), 200e6);

        assertEq(pegToken.balanceOf(address(this)), 0);
        assertEq(pegToken.balanceOf(address(psm)), 200e6);
        assertEq(credit.balanceOf(address(this)), 300e18);
        assertEq(profitManager.totalBorrowedCredit(), 100e18);

        vm.startPrank(governor);
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
        vm.stopPrank();

        // apply a loss
        // 150 CREDIT of loans completely default (~150 USD loss)
        profitManager.notifyPnL(address(this), -150e18);
        assertEq(profitManager.creditMultiplier(), 0.5e18); // 50% discounted

        // totalBorrowedCredit() reverts since targetTotalSupply() is still 300
        // but redeemableCredit() is 400 (= pegTokenBalance / creditMultiplier)
        vm.expectRevert();
        profitManager.totalBorrowedCredit();
    }
```

### Tools Used
Foundry

### Recommended Mitigation Steps
Do not use the value returned by `redeemableCredit` to calculate the `totalBorrowedCredit`. Instead, use the amount that has been minted through the PSM without scaling.


## [M-05] Reward dilution in `ERC20RebaseDistributor` through minimal token distribution

### Lines of code

[ERC20RebaseDistributor.sol#L364](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20RebaseDistributor.sol#L364)

### Impact
In the `ERC20RebaseDistributor` contract, the `distribute` function is designed to distribute tokens proportionately to all rebasing accounts. However, there is a potential issue where anyone can dilute rewards by distributing a minimal amount of tokens (1 wei).

The root cause of this issue lies in the `distribute` function. When a distribution is made, the `endTimestamp` is simply extended up to `block.timestamp + DISTRIBUTION_PERIOD` , which means the current distribution is extended. This allows for a scenario where an attacker could repeatedly distribute a minimal amount of tokens (1 wei), diluting the rewards for other users.

### Proof of Concept
Consider the following scenario:

1. A large distribution is ongoing (as an example, halfway through `DISTRIBUTION_PERIOD`), with many users set to receive a significant amount of tokens.
2. An attacker, who has a minimal amount of tokens (1 wei), calls the `distribute` function.
3. Because the `endTimestamp` is simply extended up to `DISTRIBUTION_PERIOD` in each `distribute` call, the attacker's distribution dilutes the rewards that were intended for the other users (in this case, halving them in all future blocks).
4. The attacker can carry out this attack repeatedly, such that the rewards are never fully distributed.

### Tools Used
Manual review

### Recommended Mitigation Steps
A more comprehensive solution could involve interpolating rewards individually for each distribution. This would mean that each distribution, which should be of `DISTRIBUTION_PERIOD` duration, would have its rewards calculated separately. This would prevent the dilution of rewards when a new distribution is made before the end of the current distribution period. However, the gas costs imposed by such an implementation could be prohibitively high, making this an unrealistic solution in practice.

Additionally, the `distribute` function could be restricted to being only callable by the `ProfitManager`. This would not directly fix the issue of reward dilution, but it would prevent a direct attack as per the example scenario. This would add an additional layer of security, as only the `ProfitManager` would be able to call the `distribute` function and potentially dilute rewards.


## [M-06] Rewards to GUILD token holders are sandwichable 

### Lines of code

[ProfitManager.sol#L396-L399](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L396-L399)  
[ProfitManager.sol#L427-L435](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L427-L435)

### Impact
The `ProfitManager` contract has a potential vulnerability where an attacker can perform a sandwich attack. This vulnerability arises from the way the `notifyPnL` function updates the `gaugeProfitIndex` for the reporting gauge immediately when a positive PnL is reported.

Entry points that call `notifyPnL` with a positive PnL are `LendingTerm.repay()`, `LendingTerm.PartialRepay()`, and in some cases `AuctionHouse.bid()`. Unlike rewards to CREDIT holders, rewards to GUILD holders aren't distributed gradually. This means an attacker can sandwich any of these calls, increasing their weight in this gauge, immediately call `ProfitManager.claimGaugeRewards()` or `SurplusGuildMinter.getRewards()` afterwards to reap the rewards, and then unstake/decrease their weight again.

### Proof of Concept
An attacker can follow these steps to exploit the vulnerability:

1. Monitor the blockchain for transactions that call `LendingTerm.repay()`, `LendingTerm.PartialRepay()`, or `AuctionHouse.bid()` during the first phase of an auction.
2. When such a transaction is found, the attacker sends a transaction with a higher gas price to increase their weight in the gauge.
3. The attacker then immediately calls `claimGaugeRewards()` or `getRewards()` to claim the rewards.
4. Finally, the attacker unstakes or decreases their weight in the gauge.

This sequence of actions allows the attacker to unfairly claim more rewards than they should be entitled to.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this vulnerability, consider implementing a mechanism to distribute GUILD rewards gradually, similar to how CREDIT rewards are distributed. This could prevent an attacker from being able to immediately claim rewards after increasing their weight in the gauge. Additionally, consider implementing measures to prevent rapid changes in gauge weight, such as rate limiting or cooldown periods.


## [M-07] Potential inconsistent state in `LendingTermOffboarding` can lead to redemptions remaining paused forever

### Lines of code

[LendingTermOffboarding.sol#L154](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L154)  
[LendingTermOffboarding.sol#L191-L195](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L191-L195)

### Impact
The `LendingTermOffboarding` contract has a potential issue that could lead to an inconsistent state in the system. This inconsistency could brick redemptions and disrupt the normal functioning of the lending term offboarding process.

The issue arises when a lending term is offboarded and then immediately re-onboarded without the `cleanup()` function being called. This would allow anyone to immediately offboard the term again, leading to an incorrect value in `nOffboardingsInProgress`. This would in turn block the unpausing of redemptions in the `PSM` as the `nOffboardingsInProgress` variable could not be decreased down to 0 again.

### Proof of Concept
Consider the following sequence of events:

1. A `LendingTerm` is offboarded with the intention of calling all loans and immediately re-onboarding it (due to e.g. some loans being so old that the interest accrued brings them close to being underwater) 
2. The `LendingTerm` is immediately re-onboarded.
3. No one calls the `cleanup()` function during the time it is being offboarded.
4. Since `canOffboard[term]` is still `true`, anyone can call the `offboard()` function again.
5. This adds the term to the `_deprecatedGauges` set again and increases `nOffboardingsInProgress` to 2.
6. Now, `cleanup()` can only be called once as `canOffboard[term]` will be `false` on subsequent calls, which makes it impossible to unpause redemptions in the `PSM`.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, consider adding a check in the `proposeOnboard()` function to ensure that a term cannot be re-onboarded if it hasn't been cleaned up. This could be done by checking if `LendingTermOffboarding.canOffboard[term]` is `false` before allowing the term to be onboarded. 