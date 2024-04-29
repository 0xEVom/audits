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


## [L-01] Incorrect constant used in deployment script
_This issue was downgraded from Medium to Low_

### Lines of code

[GIP_0.sol#L308](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/test/proposals/gips/GIP_0.sol#L308)

### Impact
In the deployment script, the constant `SDAI_CREDIT_HARDCAP` is intended to be used to set the hardcap for SDAI credit in the `LendingTermParams` struct. However, another constant `CREDIT_HARDCAP` is being used instead. While these two values are the same and hence this mistake has no effect, it could lead to incorrect behavior of the contract if either of the two values is modified in this or future deployments.

### Proof of Concept
In the `GIP_0.sol` file, the constant `SDAI_CREDIT_HARDCAP` is declared but not used. Instead, the constant `CREDIT_HARDCAP` is used in the `LendingTermParams` function to set the hardcap for SDAI credit.

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/test/proposals/gips/GIP_0.sol#L308

### Tools Used
Manual review

### Recommended Mitigation Steps
Replace the usage of `CREDIT_HARDCAP` with `SDAI_CREDIT_HARDCAP` in the `LendingTermParams` function to ensure the correct hardcap is set for SDAI credit. This will prevent any unintended consequences of modifying the `CREDIT_HARDCAP`  or `SDAI_CREDIT_HARDCAP` constants.


## [L-02] DoS in `LendingTermOnboarding` via proposal creation and cancellation
_This issue was downgraded from High to Low_

### Lines of code

[LendingTermOnboarding.sol#L181](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOnboarding.sol#L181)  
[Governor.sol#L348](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fd81a96f01cc42ef1c9a5399364968d0e07e9e90/contracts/governance/Governor.sol#L348)

### Impact
The `LendingTermOnboarding` contract is vulnerable to a Denial of Service (DoS) attack. This vulnerability arises from the ability for an attacker to repeatedly create and cancel proposals every time a term is created and after the `MIN_DELAY_BETWEEN_PROPOSALS` period.

This vulnerability is related to https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5h3x-9wvq-w4m2

The vulnerability referenced above was fixed in v4.9.1 of OZ's contracts by introducing opt-in frontrunning protection, but that protection isn't available in LendingTermOnboarding since the contract disallows custom proposals and enforces a deterministic description in `proposeOnboard()` based on the term address.

An attacker can exploit this vulnerability by observing when someone creates a term, then calling `proposeOnboard()` themselves. They can then cancel the proposal using the `cancel()` method provided by the `Governor` contract, from which `LendingTermOnboarding` inherits. By repeating this action every `MIN_DELAY_BETWEEN_PROPOSALS`, the attacker can effectively prevent the proposing of new terms.

```solidity
function proposeOnboard(
    address term
) external whenNotPaused returns (uint256 proposalId) {
    // Check that the term has been created by this factory
    require(created[term] != 0, "LendingTermOnboarding: invalid term");
    // Check that the term was not subject to an onboard vote recently
    require(
        lastProposal[term] + MIN_DELAY_BETWEEN_PROPOSALS < block.timestamp,
        "LendingTermOnboarding: recently proposed"
    );
    lastProposal[term] = block.timestamp;
    // ...
}
```

### Proof of Concept
Consider the following scenario:

1. Alice creates a term using the `createTerm()` function.
2. Bob  observes Alice's transaction and calls `proposeOnboard()` with the term Alice just created before she can do so herself
3. Bob then cancels the proposal using the `cancel()` method.
4. Bob repeats this process every `MIN_DELAY_BETWEEN_PROPOSALS`, effectively preventing Alice and other legitimate users from proposing new terms.

### Tools Used
Manual review

### Recommended Mitigation Steps
Allow users to provide a custom `description` string to attach to the one generated in `getOnboardProposeArgs()`.  This way, they would also be able to benefit from the [frontrunning protection](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/Governor.sol#L284) implemented in OZ's Governor.

## [L-03] Reorg & Frontrunning Attack Vulnerability in `createTerm()`
_This issue was downgraded from Medium to Low_

### Lines of code

[LendingTermOnboarding.sol#L153](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOnboarding.sol#L153)  
[Clones.sol#L33](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fd81a96f01cc42ef1c9a5399364968d0e07e9e90/contracts/proxy/Clones.sol#L33)

### Impact
The `createTerm()` function in the `LendingTermOnboarding.sol` contract is vulnerable to a reorg/frontrunning attack. This function deploys a new `LendingTerm` contract using OZ's `Clone.clone()`, which uses `create()` under the hood. Here, the address derivation depends solely on the `LendingTermOnboarding` nonce.

The worst-case scenario is an attacker exploiting the predictability of contract creation to manipulate votes, leading to a malicious contract being approved instead of the intended one.

This is a Medium severity issues previously reported here an in the referenced reports https://solodit.xyz/issues/m-09-create-methods-are-suspicious-of-the-reorg-attack-code4rena-pooltogether-pooltogether-git

### Proof of Concept
The following sequence of events illustrates the vulnerability:

1. Alice, a user with a significant amount of GUILD tokens, submits a series of transactions in which she:
   - Calls `createTerm()` with profitable parameters 
   - Calls `proposeOnboard()` to propose its onboarding
   - Calls `castVote()` to cast her vote in support of her proposal
2. Bob observes Alice's transactions and frontruns her by calling `createTerm()` with malicious parameters before Alice's transactions are executed.
3. Bob's malicious term gets the address Alice expected to create.
4. Since the proposal ID only depends on the term address, Alice's votes are cast in support of Bob's malicious term.

### Tools Used
Manual review

### Recommended Mitigation Steps
Use `Clones.cloneDeterministic()` passing a hash of the proposal parameters as the salt.


## [L-04] Failed transfers in `LendingTerm.onBid()` will lead to protocol loss
_This issue was downgraded from Medium to Low_

### Lines of code

[LendingTerm.sol#L804-L817](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L804-L817)

### Impact
The `LendingTerm.onBid()` function requires successful transfers to both the borrower and the bidder. This mandatory requirement can lead to a loss in the protocol under several circumstances. 

### Proof of Concept
Two scenarios can lead to this issue, not considering ERC-777 callbacks:

1. **Borrower Blacklisted:** If a borrower gets themselves blacklisted by the collateral token, they will not be able to receive tokens. In this scenario, only the edge case where `elapseTime == _startTime + midPoint` will succeed without a loss to the protocol.

2. **Transfers Paused on Collateral Token:** If a loan is called when transfers are paused on the collateral token (e.g., USDC), any bid will fail. Neither of the two transactions will succeed and the loan will have to be forgiven, leading to a loss in the protocol.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, a try-catch mechanism can be implemented around the transfer calls in the `onBid` function. If a transfer fails, the function should catch the error and increase the allowance accordingly so the recipient can later pull the tokens themselves. This would prevent the protocol from incurring a loss due to failed transfers.

## [L-05] `clock()` will not work properly for Arbitrum due to usage of `block.number`
_This issue was downgraded from Medium to Low_

### Lines of code

[GovernorVotes.sol#L25-L43](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fd81a96f01cc42ef1c9a5399364968d0e07e9e90/contracts/governance/extensions/GovernorVotes.sol#L25-L43)  
[Governor.sol#L180](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fd81a96f01cc42ef1c9a5399364968d0e07e9e90/contracts/governance/Governor.sol#L180)  
[Governor.sol#L277](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fd81a96f01cc42ef1c9a5399364968d0e07e9e90/contracts/governance/Governor.sol#L277)
^ the above functions are called internally through `super.` calls, either directly or through `GovernorTimelockControl`

### Impact
`clock()` and `CLOCK_MODE()` will not work correctly on Arbitrum.

This issue is similar to https://github.com/code-423n4/2023-06-lybra-findings/issues/114

### Proof of Concept
`GuildGovernor`, `GuildVetoGovernor` and `LendingTermOnboarding` rely internally on the `clock()` function provided by OZ's `GovernorVotes`, which returns the block number by default if the voting token doesn't provide a `clock()` function. Furthermore, `block.number` is employed in multiple instances in `ERC20MultiVotes` to determine voting power. However, on Arbitrum, `block.number` [returns](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/block-numbers-and-time) the most recently synced L1 block number, which is only updated once per minute. This can lead to inaccurate timing, as the block number does not accurately reflect the passage of time on these networks.

### Tools Used
Manual review

### Recommended Mitigation Steps
Inherit from [IERC6372](https://docs.openzeppelin.com/contracts/4.x/api/interfaces#IERC6372) in GuildToken and CreditToken and provide a `clock()` function based on the L2 block number. Additionally, update instances of `block.number` in `ERC20MultiVotes` to use the [L2 block number](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/block-numbers-and-time#arbitrum-block-numbers).

## [L-06] Constant `BLOCKS_PER_DAY` inaccurate for L2 chains
_This issue was downgraded from Medium to Low_

### Lines of code

[GIP_0.sol#L83](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/test/proposals/gips/GIP_0.sol#L83)

### Impact
The `BLOCKS_PER_DAY` constant in the `GIP_0.sol` deployment script is set to a fixed value that corresponds to the block time on Ethereum mainnet. However, this value will not be accurate if the contract is deployed on Arbitrum or other L2s, which have a different block time. This discrepancy could lead to unexpected behavior in the contract's functionality, as any calculations or operations relying on `BLOCKS_PER_DAY` would be based on incorrect assumptions about the frequency of block generation.

### Proof of Concept
The issue can be found in the `GIP_0.sol` file where `BLOCKS_PER_DAY` is defined.
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/test/proposals/gips/GIP_0.sol#L83

### Tools Used
Manual review

### Recommended Mitigation Steps
To resolve this issue, consider making `BLOCKS_PER_DAY` a dynamic variable that is defined based on the `chainId` during contract deployment. This would ensure that the value of `BLOCKS_PER_DAY` accurately reflects the block time of the specific chain where the contract is being deployed.

