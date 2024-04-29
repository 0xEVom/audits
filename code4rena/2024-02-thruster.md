## [M-01] Tickets can be entered after prizes for current round have partially been distributed

### Lines of code

[ThrusterTreasure.sol#L85](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L85)

### Impact
The `ThrusterTreasure` contract is designed to facilitate a lottery game where users can enter tickets to win prizes based on entropy. The contract includes mechanisms for entering tickets into rounds (`enterTickets()`), setting prizes for rounds (`setPrize()`), and claiming prizes (`claimPrizesForRound()`). A critical aspect of the game's integrity is ensuring each ticket has an equal chance to win every prize.

However, there is a significant flaw in `enterTickets()`. The function checks if winning tickets for the prize index 0 have been set by verifying that `winningTickets[currentRound_][0].length == 0`. This check is intended to prevent users from entering tickets after prizes have begun to be distributed, but it does not account for prizes with higher indices that may already have been distributed. As a result, users can still enter tickets after some prizes have been distributed, but these late-entered tickets will not have a chance to win the already distributed prizes:
[ThrusterTreasure.sol#L83-L96](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L83-L96)
```solidity
function enterTickets(uint256 _amount, bytes32[] calldata _proof) external {
    ...
    require(winningTickets[currentRound_][0].length == 0, "ET");
    ...
}
```

### Proof of Concept
1. The contract owner sets up a new round with multiple prizes.
2. User A enters tickets early in the round.
3. The contract owner distributes prizes for index 1.
4. User B enters tickets into the round.
5. Due to the flawed logic in `enterTickets()`, User B's tickets are accepted, even though the prizes for indices 1 and above have already been distributed. User B's tickets, therefore, have no chance of winning those prizes and are worth less than user A's, but the system incorrectly allows their participation for the undistributed prize at index 0.

### Tools Used
Manual review

### Recommended Mitigation Steps
Freeze ticket entry for the current round once any prize has been set.

## [M-02] `claimPrizesForRound` transfers the entire amount deposited for a prize regardless of the number of winners

### Lines of code

[ThrusterTreasure.sol#L163-L184](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L163-L184)  
[ThrusterTreasure.sol#L102-L134](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L134)

### Impact
`claimPrizesForRound()` transfers the entire amount of a prize to a winner without considering the total number of winners for that prize. 

The prize for a given round and prize index can be set by calling the `setPrize()` function, which pulls the amounts from the caller (the owner) and stores the prize data in the `prizes` array:

[ThrusterTreasure.sol#L163-L184](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L163-L184)
```solidity
function setPrize(uint256 _round, uint64 _prizeIndex, uint256 _amountWETH, uint256 _amountUSDB, uint64 _numWinners)
	external
	onlyOwner
{
	require(_round >= currentRound, "ICR");
	require(_prizeIndex < maxPrizeCount, "IPC");
	depositPrize(msg.sender, _amountWETH, _amountUSDB);
	prizes[_round][_prizeIndex] = Prize(_amountWETH, _amountUSDB, _numWinners, _prizeIndex, uint64(_round));
}

function depositPrize(address _from, uint256 _amountWETH, uint256 _amountUSDB) internal {
	WETH.transferFrom(_from, address(this), _amountWETH);
	USDB.transferFrom(_from, address(this), _amountUSDB);
	emit DepositedPrizes(_amountWETH, _amountUSDB);
}
```

However, the `claimPrizesForRound()` function always transfers the full prize amounts to the first caller, regardless of the number of winners for the prize. Once the prize for a specific index is claimed, other winners of that prize cannot claim their share (or winners of other prizes may end up not being able to claim theirs), effectively being denied their winnings:

[ThrusterTreasure.sol#L102-L134](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L134)
```solidity
function claimPrizesForRound(uint256 roundToClaim) external {
	...
	for (uint256 i = 0; i < maxPrizeCount_; i++) {
		Prize memory prize = prizes[roundToClaim][i];
		uint256[] memory winningTicketsRoundPrize = winningTickets[roundToClaim][i];
		for (uint256 j = 0; j < winningTicketsRoundPrize.length; j++) {
			uint256 winningTicket = winningTicketsRoundPrize[j];
			if (round.ticketStart <= winningTicket && round.ticketEnd > winningTicket) {
				_claimPrize(prize, msg.sender, winningTicket);
			}
		}
	}
	...
}

function _claimPrize(Prize memory _prize, address _receiver, uint256 _winningTicket) internal {
	uint256 amountETH = _prize.amountWETH;
	uint256 amountUSDB = _prize.amountUSDB;
	WETH.transfer(_receiver, amountETH);
	USDB.transfer(_receiver, amountUSDB);
	emit ClaimedPrize(_receiver, _prize.round, _prize.prizeIndex, amountETH, amountUSDB, _winningTicket);
}
```

This approach can lead to scenarios where the amount available to be distributed among prize winners is less than that represented by the prizes stored in the `prizes` array.

This is considered medium severity because:
- `claimPrizesForRound()` will revert if there aren't enough funds to transfer the prize to the winner, altering the user
- the owner can mitigate this by simply "refilling" the prize as many times as needed

### Proof of Concept
1. A prize is set with a certain amount of WETH and USDB for a specific round and prize index, intended for multiple winners.
2. Multiple users enter the round with tickets that end up winning this prize.
3. The first user to call `claimPrizesForRound()` for this round and prize index successfully claims the entire prize amount.
4. Subsequent winners attempting to claim their share of the prize for the same round and prize index find that they cannot, as the prize has already been fully distributed to the first caller.

### Tools Used
Manual review

### Recommended Mitigation Steps
It is unclear whether the amounts passed to `setPrize()` are meant to be distributed among all winners of the given prize or to be paid out to each winner, but the cleaner approach would be the latter. In that case, the amount pulled from the owner can simply be scaled by the number of winners:
```solidity
function depositPrize(address _from, uint64 _numWinners, uint256 _amountWETH, uint256 _amountUSDB) internal {
	WETH.transferFrom(_from, address(this), _amountWETH * _numWinners);
	USDB.transferFrom(_from, address(this), _amountUSDB * _numWinners);
	emit DepositedPrizes(_numWinners, _amountWETH, _amountUSDB);
}
```

## [M-03] Dynamic modification of `maxPrizeCount` affects prize claims

### Lines of code

[ThrusterTreasure.sol#L139-L142](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L139-L142)  
[ThrusterTreasure.sol#L102-L120](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L120)

### Impact
The `ThrusterTreasure` contract is designed to manage rounds of a lottery game, where participants can enter tickets and claim prizes based on random draws. The contract includes a variable `maxPrizeCount` which dictates the maximum number of prizes that can be set for any given round. This variable can be modified by the contract owner at any time through the `setMaxPrizeCount(uint256 _maxPrizeCount)` function:

[ThrusterTreasure.sol#L139-L142](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L139-L142)
```solidity
    function setMaxPrizeCount(uint256 _maxPrizeCount) external onlyOwner {
        maxPrizeCount = _maxPrizeCount;
        emit SetMaxPrizeCount(_maxPrizeCount);
    }
```

The issue arises when `maxPrizeCount` is decreased after prizes for a round have been set but before they have been claimed, which can be at any future time. Since the `claimPrizesForRound(uint256 roundToClaim)` function iterates over prize indices up to `maxPrizeCount`, reducing this count means that winners of prizes with indices higher than the new `maxPrizeCount` will be unable to claim their winnings:

[ThrusterTreasure.sol#L102-L120](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L120)
```solidity
    function claimPrizesForRound(uint256 roundToClaim) external {
        ...
        
        uint256 maxPrizeCount_ = maxPrizeCount;
        for (uint256 i = 0; i < maxPrizeCount_; i++) {
            [claim prize]
        }
        entered[msg.sender][roundToClaim] = Round(0, 0, roundToClaim); // Clear user's tickets for the round
        emit CheckedPrizesForRound(msg.sender, roundToClaim);
    }
```

This could lead to a scenario where legitimate winners are denied their prizes due to a change in contract state that is unrelated to the rules of the game or their actions. Moreover, since calling `claimPrizesForRound()` clears the user's entries for the round, reverting `maxPrizeCount` to its previous state does not allow them to claim the remaining tickets. This means they will effectively never be able to claim their prize.

Same reasoning as for "Risk of losing prizes for early claims in `ThrusterTreasure`", severity is high because this:
- leads to loss of (potentially matured) yield/rewards
- does not require an error on either the owner or the user's part
- can happen to the user without them ever becoming aware of it

### Proof of Concept
1. The contract owner sets `maxPrizeCount` to 5 and configures five prizes for a given round `n`.
2. Users participate in the round, and the round concludes with winners determined for all five prizes.
3. After a few rounds, the contract owner reduces `maxPrizeCount` to 3.
4. Winners of prizes 4 and 5 in round `n` attempt to claim their prizes but are unable to do so because the `claimPrizesForRound(uint256 roundToClaim)` function now iterates only up to the new `maxPrizeCount` of 3.

### Tools Used
Manual review

### Recommended Mitigation Steps
To address this issue, implementing a checkpoint pattern for the `maxPrizeCount` variable is suggested. This method involves tracking changes to `maxPrizeCount` with checkpoints that record the value and the round number when the change occurs.

A possible implementation could look like this:
```solidity
// Add a struct to store checkpoints for maxPrizeCount changes
struct MaxPrizeCountCheckpoint {
    uint256 round;
    uint256 maxPrizeCount;
}

// Use an array to keep track of all checkpoints
MaxPrizeCountCheckpoint[] public maxPrizeCountCheckpoints;

constructor(
	...
) Ownable(msg.sender) {
	maxPrizeCountCheckpoints.push(
		MaxPrizeCountCheckpoint(0, _maxPrizeCount)
	);
    ...
}

// Modify setMaxPrizeCount to push a new checkpoint to the array
function setMaxPrizeCount(uint256 _maxPrizeCount) external onlyOwner {
	require(_maxPrizeCount != getMaxPrizeCountForRound(currentRound), "same value")
    maxPrizeCountCheckpoints.push(
		MaxPrizeCountCheckpoint(currentRound, _maxPrizeCount)
    );
    emit SetMaxPrizeCount(_maxPrizeCount);
}

// Helper function to get the maxPrizeCount for a given round
// Assumes more recent rounds will be queried more often
function getMaxPrizeCountForRound(uint256 _round) public view returns (uint256) {
    uint256 length = maxPrizeCountCheckpoints.length;
    for (uint256 i = length; i > 0; i--) {
        MaxPrizeCountCheckpoint storage checkpoint = maxPrizeCountCheckpoints[i - 1];
        if (checkpoint.round <= _round) {
            return checkpoint.maxPrizeCount;
        }
    }
    return 0;
}

// Disallow setting prizes for future rounds since the maxPrizeCount could change
function setPrize(uint64 _prizeIndex, uint256 _amountWETH, uint256 _amountUSDB, uint64 _numWinners) external onlyOwner {
    uint256 maxPrizeCount = getMaxPrizeCountForRound(currentRound);
    require(_prizeIndex < maxPrizeCount, "IPC");
    ...
}

function claimPrizesForRound(uint256 roundToClaim) external {
    uint256 maxPrizeCount = getMaxPrizeCountForRound(roundToClaim);
    ...
}
```

This change ensures that each round's prize structure is fixed upon the round's creation, preventing post-hoc alterations that could negatively impact participants. Note that this implementation still requires attention is paid to not calling `setMaxPrizeCount()` for a given round if prizes have already been set for higher indices.

## [M-04] Incorrect gas claiming logic in `ThrusterPoolDeployer`

### Lines of code

[ThrusterPoolDeployer.sol#L45-L47](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/ThrusterPoolDeployer.sol#L45-L47)

### Impact

From the contest documentation:

> All contracts that use gas should comply with the Blast gas claim logic.

However, the `ThrusterPoolDeployer` contains a flaw in the implementation of `claimGas()` which will prevent it from ever claiming the gas it induces. The function attempts to claim gas for the zero address (`address(0)`) instead of the deployer's own address (`address(this)`):

```solidity
    function claimGas(address _recipient) external override onlyFactory returns (uint256 amount) {
        amount = IBlast(BLAST).claimMaxGas(address(0), _recipient);
    }
```

This misconfiguration prevents the `ThrusterPoolDeployer` from reclaiming any gas, as the `IBlast.claimMaxGas()` call will always fail when provided with the zero address.

### Proof of Concept
https://github.com/blast-io/blast/blob/master/blast-optimism/packages/contracts-bedrock/src/L2/Blast.sol#L274-L283
```solidity
/**
 * @notice Claims gas available to be claimed at max claim rate for a specific contract. Called by an authorized user
 * @param contractAddress The address of the contract for which maximum gas is to be claimed
 * @param recipientOfGas The address of the recipient of the gas
 * @return The amount of gas that was claimed
 */
function claimMaxGas(address contractAddress, address recipientOfGas) external returns (uint256) {
	require(isAuthorized(contractAddress), "Not allowed to claim max gas");
	return IGas(GAS_CONTRACT).claimMax(contractAddress, recipientOfGas);
}
```

### Tools Used
Manual review

### Recommended Mitigation Steps
Use `address(this)` rather than `0`.


## [L-01] Redundant Length Check in setWinningTickets

### Context
[ThrusterTreasure.sol#L291](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L291)

### Description
In the `setWinningTickets` function, there is a length check on the `_winningTickets` array to ensure it matches the expected number of winners (`numWinners`). This check is redundant and results in unnecessary gas consumption because `_winningTickets` is always initialized with a length equal to `numWinners`:

```solidity
uint256[] memory _winningTickets = new uint256[](numWinners);
for (uint256 i = 0; i < numWinners; i++) {
    _winningTickets[i] = revealRandomNumber(sequenceNumbers[i], userRandoms[i], providerRandoms[i]);
    emit SetWinningTicket(_round, _prizeIndex, _winningTickets[i], i);
}
winningTickets[_round][_prizeIndex] = _winningTickets;
require(_winningTickets.length == numWinners, "WTL");
```

### Recommendation 
It is recommended to remove the length check on the `_winningTickets` array to simplify the code and optimize gas usage. This change will not introduce any additional issues as the initialization of `_winningTickets` inherently guarantees its length matches `numWinners`.


## [L-02] Code Duplication in ThrusterYield Contract

### Context
[ThrusterGas.sol#L21-L37](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/ThrusterGas.sol#L21-L37)  
[ThrusterYield.sol#L44-L55](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterYield.sol#L44-L55)

### Description
The `ThrusterYield` contract contains code that is identical to what is implemented in the `ThrusterGas` contract. Specifically, the functionality for claiming gas is replicated in both contracts. This redundancy violates the DRY (Don't Repeat Yourself) principle, making the codebase harder to maintain and more susceptible to bugs if updates are required in the future.

### Recommendation 
The `ThrusterYield` contract could inherit from the `ThrusterGas` contract, eliminating the need to replicate the gas claiming functionality. This approach not only reduces the overall codebase size but also simplifies future maintenance and updates.

## [L-03] Risk of losing prizes for early claims in `ThrusterTreasure`
_This issue was downgraded from Medium to Low_

### Lines of code

[ThrusterTreasure.sol#L98-L104](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L98-L104)  
[ThrusterTreasure.sol#L261-L277](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L261-L277)

### Impact
The `ThrusterTreasure` contract facilitates a lottery game where users can enter tickets and claim prizes based on random draws. The contract uses a combination of user ticket entries, 
Merkle proofs for verification, and an entropy source for drawing winning tickets.

The `claimPrizesForRound()` function allows users to claim their prizes for a specific round. However, there's a significant issue where a user can claim their prize as soon as the winning tickets for the first prize index are set, without waiting for all prizes within the round to be determined. This premature claiming could lead to a scenario where users' tickets are cleared from the round after calling `claimPrizesForRound()` without receiving any prize, even if they had one or more winning tickets.

[ThrusterTreasure.sol#L98-L104](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L98-L104)
```solidity
    /**
     * Claim prizes for a round
     * @param roundToClaim - The round to claim prizes for
     */
    function claimPrizesForRound(uint256 roundToClaim) external {
        require(roundStart[roundToClaim] + MAX_ROUND_TIME >= block.timestamp, "ICT");
        require(winningTickets[roundToClaim][0].length > 0, "NWT");
        
        ...
        
        entered[msg.sender][roundToClaim] = Round(0, 0, roundToClaim); // Clear user's tickets for the round
        emit CheckedPrizesForRound(msg.sender, roundToClaim);
    }
```

[ThrusterTreasure.sol#L261-L277](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L261-L277)
```solidity
    /**
     *
     * @param _round - The round to claim the prize for
     * @param _prizeIndex - The index of the prize to claim
     * @param sequenceNumbers - The sequence numbers of the random number requests
     * @param userRandoms - The user random numbers
     * @param providerRandoms - The provider random numbers
     */
    function setWinningTickets(
        uint256 _round,
        uint256 _prizeIndex,
        uint64[] calldata sequenceNumbers,
        bytes32[] calldata userRandoms,
        bytes32[] calldata providerRandoms
    ) external onlyOwner {
        require(roundStart[_round] + MAX_ROUND_TIME >= block.timestamp, "ICT");
        require(winningTickets[_round][_prizeIndex].length == 0, "WTS");
        ...
```

This is considered high severity because it:
- leads to loss of (potentially matured) yield/rewards
- does not require an error on either the owner or the user's part
- can happen to the user without them ever becoming aware of it

### Proof of Concept
1. The owner sets the winning tickets for the prize with index 0 of a round using `setWinningTickets()`.
2. A user, who has entered tickets for the round and who has a winning ticket for the prize with index 1, calls `claimPrizesForRound()` and claims their prize.
3. The owner then sets the winning tickets for the remaining prize indexes of the round.
4. The user is now unable to claim their prize because their tickets have been cleared from the round after calling `claimPrizesForRound()`.

### Tools Used
Manual review

### Recommended Mitigation Steps
To address this issue, it is recommended to implement a mechanism that ensures all prizes for a round are set before any prize claims can be made. This could be achieved by:

1. Introducing a state variable that tracks whether all prizes for a round have been set.
2. Modifying the `claimPrizesForRound()` function to check this state variable before allowing any prize claims.

A possible implementation could look like this:

```solidity
// Add a state variable to track if all prizes for the round have been set
mapping(uint256 => bool) public allPrizesSetForRound;

// Modify the setPrize function to set allPrizesSetForRound to true when the last prize is set
// This requires setting prizes in sequential order
function setPrize(uint256 _round, uint64 _prizeIndex, uint256 _amountWETH, uint256 _amountUSDB, uint64 _numWinners) external onlyOwner {
    require(_round >= currentRound, "ICR");
    require(_prizeIndex < maxPrizeCount, "IPC");
    // Existing implementation
    // ...
    if (_prizeIndex == maxPrizeCount - 1) {
        allPrizesSetForRound[_round] = true;
    }
}

// Modify the claimPrizesForRound function to check if all prizes have been set
function claimPrizesForRound(uint256 roundToClaim) external {
    require(allPrizesSetForRound[roundToClaim], "Not all prizes set");
    // Existing implementation
    // ...
}
```

This solution ensures that users can only claim prizes once all prizes for the round have been determined, preserving the fairness and integrity of the lottery.

## [L-04] Tickets will be lost if they're entered after `MAX_ROUND_TIME`
_This issue was downgraded from Medium to Low_

### Lines of code

[ThrusterTreasure.sol#L83](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L83)  
[ThrusterTreasure.sol#L276](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L276)

### Impact
The `ThrusterTreasure` specifies a maximum round time (`MAX_ROUND_TIME`) intended to limit the period during which winners can be chosen and prizes claimed for a given round. However, the `enterTickets()` function does not enforce this time limit:
[ThrusterTreasure.sol#L83-L96](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L83-L96)
```solidity
    function enterTickets(uint256 _amount, bytes32[] calldata _proof) external {
        uint256 currentRound_ = currentRound;
        require(winningTickets[currentRound_][0].length == 0, "ET");
        bytes32 node = keccak256(abi.encodePacked(msg.sender, _amount));
        require(MerkleProof.verify(_proof, root, node), "IP");
        uint256 ticketsToEnter = _amount - cumulativeTickets[msg.sender];
        require(ticketsToEnter > 0, "NTE");
        uint256 currentTickets_ = currentTickets;
        Round memory round = Round(currentTickets_, currentTickets_ + ticketsToEnter, currentRound_);
        entered[msg.sender][currentRound_] = round;
        cumulativeTickets[msg.sender] = _amount; // Ensure user can only enter tickets once, no partials
        currentTickets += ticketsToEnter;
        emit EnteredTickets(msg.sender, currentTickets_, currentTickets_ + ticketsToEnter, currentRound_);
    }
```

This means users can continue to enter tickets even after the `MAX_ROUND_TIME` has elapsed, provided that the winning tickets for the round have not been set. This can lead to scenarios where tickets are entered into a round that cannot have winners determined, effectively rendering these tickets useless.

### Proof of Concept
1. `MAX_ROUND_TIME` passes for a lottery round in `ThrusterTreasure` without setting winning tickets, possibly due to no prizes.
2. A user (`Bob`) enters tickets after this period.
3. Since the round's prize setting period has expired, `Bob`'s tickets cannot win, effectively rendering them useless.

### Tools Used
Manual review

### Recommended Mitigation Steps
Add a check in the `enterTickets()` function to ensure that the current time is within the `MAX_ROUND_TIME` from the round's start time. Ideally this is combined with a check that none of the prizes have been set yet, as detailed in another finding.
