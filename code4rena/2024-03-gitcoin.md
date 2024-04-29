## [L-01] UUPS implementation contract should not be initializable

### Context
[IdentityStaking.sol#L163](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L163)

### Description
The `IdentityStaking` contract is an implementation contract that should not allow its initialization, but lacks a constructor with a call to `_disableInitializers()`.

While newer versions of UUPSUpgradeable such as the one `IdentityStaking` inherits from can no longer be upgraded directly via `upgradeToAndCall()`, it is safer and considered better practice to disallow the initialization of implementation contracts.

### Recommendation 
Add a constructor to the `IdentityStaking` contract that calls `_disableInitializers()` to prevent the implementation contract from being initialized.

## [L-02] Missing address validation in `initialize()`

### Context
[IdentityStaking.sol#L179](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L179)

### Description
The `initialize()` checks that the `tokenAddress` is not 0, but does not perform any checks on the `_burnAddress` and `initialAdmin` parameters. However, if `_burnAddress` is set to the zero address, the `lockAndBurn()` function will always fail because the GTC token contract does not allow transfers to the zero address. Similarly, if `initialAdmin` is set to the zero address, administrative privileges will be irrecoverably lost, as no address will have the `DEFAULT_ADMIN_ROLE`, making it impossible to manage roles or upgrade the contract.

### Recommendation 
Add similar validation checks for `_burnAddress` and `initialAdmin`, ensuring neither can be set to the zero address.

## [L-03] Missing call to `__UUPSUpgradeable_init()` in initializer

### Context
[IdentityStaking.sol#L183](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L183)

### Description
The `initialize()` function is missing a call to `__UUPSUpgradeable_init()`. While this particular initializer is empty, the `__AccessControl_init()` initializer is called, which is also empty. Future versions of the OpenZeppelin contracts might include logic in these initializers, and failing to call them could result in incomplete or incorrect initialization.

### Recommendation 
Include a call to `__UUPSUpgradeable_init()` within the `initialize()` function.

## [L-04] `slash()` can be frontrun to prevent slashing if a stake is past its unlock time

### Context
[IdentityStaking.sol#L424](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L424)
[IdentityStaking.sol#L288](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L288)

### Description
As per the provided documentation, all stakes are [liable](https://github.com/gitcoinco/id-staking-v2?tab=readme-ov-file#slash) to be slashed, even if they're past their unlock time. However, this means that stakes past their unlock time are not guaranteed to be slashed when they're provided as an argument in a call to `slash()`, since the staker could frontrun the transaction to withdraw their stake.

Given that stakes past their unlock time do not affect a user's score, as specified by the sponsor privately, this should have no further consequences.

### Recommendation
The decision to allow stakes past their unlock time could be reconsidered. This not only leads to the issue just described, but is unfair towards those who do not withdraw and means the incentive is always to remove one's stake ASAP as soon as it's unlocked, just in case.

## [L-05] Loss of precision and lack of minimum stake can enable sybil attack

### Context
[IdentityStaking.sol#L443](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L443)  
[IdentityStaking.sol#L473](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L473)

### Description
The staking mechanism does not enforce a minimum stake amount for participants. This opens up a vulnerability where an attacker could avoid being slashed by distributing their stake across many accounts with amounts that would result in a slash of less than the smallest token unit. For example, every stake of 10 wei or less will not suffer any losses for slashes of under 10%.

Furthermore, the slasher may need to pay a substantial amount of gas in order to slash such a large number of accounts, which is strictly speaking a separate attack since it also applies to larger amounts that are slashable.

### Recommendation
Fortunately, the calculation of an account's score happens off-chain, which means the protocol could choose to simply ignore such accounts. In order to prevent such a scenario from happening in the first place, consider introducing a minimum staking amount.

## [L-06] Unprotected admin role represents a single point of failure

### Context
[IdentityStaking.sol#L186](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L186)

### Description
While the contract exercises recommended security practices by employing OpenZeppelin's `AccessControl` and separating responsibilities into different roles (none of which have the ability to cause irreversible loss of funds), the unrenounced `DEFAULT_ADMIN_ROLE` holds the ability to grant and revoke other roles unrestrictedly, as well as to upgrade the implementation to any other contract.

This represents a single point of failure if the private key of the admin role is compromised. In such a scenario, an attacker could e.g. upgrade the implementation to irreversibly brick all functionality in the contract.

### Recommendation
Consider granting the admin role to a [timelock](https://docs.openzeppelin.com/contracts/3.x/access-control#delayed_operation).


## [L-07] Community staking can be exploited as a honeypot
_This issue was downgraded from Medium to Low_

### Lines of code

[IdentityStaking.sol#L238-L239](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L238-L239)  
[IdentityStaking.sol#L329-L330](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L329-L330)  
[IdentityStaking.sol#L270-L271](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L270-L271)

### Impact
Both the `selfStake()` and `communityStake()` functions enforce the same restrictions on the duration passed as parameter. However, this can lead to community stakers being required to stake for longer than an existing self stake.

For instance, _any_ community stakes created on an identity with a self stake with the minimum duration will be required to end later than the self stake.

An attacker can exploit this by staking tokens on their own identity for the minimum duration and then convincing other users to stake on their identity. Once the attacker's self stake unlock time is reached, they can withdraw their stake and subsequently commit an offense, leading to the slashing of the community stakes placed on their identity with no loss to them.

### Proof of Concept
1. Alice stakes an amount of tokens on her own identity using `selfStake()` for the minimum duration (12 weeks).
2. Alice convinces Bob to stake tokens on her identity through `communityStake()`.
3. Bob has no other choice but to stake his tokens with an `unlockTime` later than Alice's.
4. Once the minimum duration expires, Alice withdraws her self stake.
5. Alice then commits an offense, and Bob is slashed for having staked on Alice's identity, but Alice suffers no loss.

### Tools Used
Manual review

### Recommended Mitigation Steps
Consider allowing community stakers to stake for a duration dependent on an existing self stake's duration as a default.

Alternatively, add a `bool` to the community staking functions (`communityStake()` and `extendCommunityStake()`) requiring the caller to be explicit in allowing their stake's duration to surpass an existing self stake's duration.


## [N-01] Constants declared as mutable variables

### Context
[IdentityStaking.sol#L104](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L104)

### Description
The variable `burnRoundMinimumDuration` is defined as a mutable state variable, but it is assigned a fixed value in the initializer and cannot be modified afterwards. Similarly, the `token` and `burnAddress` can be known at compile time, but are assigned a value passed as argument to the initializer. 

### Recommendation
Constants will be stored in the bytecode, which means they can be safely used in implementation contracts.

A declaration of the aforementioned variables as constants would more closely reflect their purpose and ensure its immutability regardless of potential code changes. Additionally, using a constant value can result in minor gas savings for contract deployments and interactions that read this value.

## [N-02] Lack of zero amount check in `withdrawSelfStake()`

### Context
[IdentityStaking.sol#L285](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L285)

### Description
The `withdrawSelfStake()` function lacks a check for a zero withdrawal amount, unlike its counterpart `withdrawCommunityStake()`. This inconsistency allows the emission of a `SelfStakeWithdrawn` event with a zero amount, which could potentially disrupt off-chain processing or monitoring systems that rely on these events.

### Recommendation 
Check for a zero amount, or abstract the functionality in `withdrawSelfStake()` and `withdrawCommunityStake()` to ensure they perform equivalent logic.

## [N-03] Superfluous checks for `address(0)`

### Context
[IdentityStaking.sol#L354](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L354)  
[IdentityStaking.sol#L385](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L385)

### Description
The checks against the zero address in `extendCommunityStake()`, `withdrawCommunityStake()` and `release()` can be removed. Given that the `communityStake()` function prevents initiating a stake on the zero address, these checks are redundant. 

### Recommendation 
Remove the checks against `address(0)` in these functions to simplify the code and reduce gas costs.

## [N-04] Avoid duplicate logic by using private helper functions and modifiers

### Context
[IdentityStaking.sol#L230-L254](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L230-L254)  
[IdentityStaking.sol#L321-L345](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L321-L345)  

[IdentityStaking.sol#L262-L280](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L262-L280)  
[IdentityStaking.sol#L360-L378](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L360-L378)  

[IdentityStaking.sol#L286-L303](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L286-L303)  
[IdentityStaking.sol#L393-L410](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L393-L410)  

[IdentityStaking.sol#L443-L467](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L443-L467)  
[IdentityStaking.sol#L473-L498](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L473-L498)  

[IdentityStaking.sol#L554-L563](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L554-L563)  
[IdentityStaking.sol#L565-L574](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L565-L574)

### Description
The contract contains a considerable amount of duplicate logic across different functions. This redundancy not only increases the deployment costs due to the extra bytecode but also makes the contract harder to maintain. For instance, any changes to the slashing logic would need to be applied in multiple places, increasing the risk of errors.

### Recommendation
Refactor the duplicate logic into private helper functions and/or modifiers. [Here](https://gist.github.com/0xEVom/6661350219d057c40d027979a618b9c8)'s one possible implementation that reduces the number of nSLOC by 23% (the zero address checks recommended in [L-02](#l-02) have been added and superfluous ones pointed out in [N-03](#n-03) have been removed).

## [N-05] Consider emitting an event at the end of the initializer

### Context
[IdentityStaking.sol#L202](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L202)  

### Description
The contract emits no event on initialization.

### Recommendation
As a best practice, consider emitting an event when the contract is initialized. This way, it's easy for the user to track the exact point in time when the contract was initialized, by filtering the emitted events.

## [N-05] Initializer function can be made external

### Context
[IdentityStaking.sol#L178](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L178)  

### Description
The `initialize()` function has `public` visibility, but is not called internally. Furthermore, it should never be called in the constructor, as the contract will be proxied.

### Recommendation
Restrict visibility to `external` .

## [N-06] Use of floating pragma

### Context
[IdentityStaking.sol#L2](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L2)

### Description
Floating pragma is meant to be used for libraries and contracts that have external users and need broader compatibility.

### Recommendation
Lock the pragma to a specific version. This helps avoid deploys with compiler versions that have not beed tested locally and ensures consistency across local setups during development.

## [N-07] The latest Solidity versions may not be the safest

### Context
[IdentityStaking.sol#L2](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L2)

### Description
The contract uses Solidity version 0.8.23 or higher, which was [released](https://github.com/ethereum/solidity/releases/tag/v0.8.23) 4 months ago. Newer versions have a higher risk of yet-undiscovered bugs in the compiler or in new language features.

### Recommendation
Consider using a slightly older and trusted Solidity version, e.g. the one Slither [recommends](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity).

## [N-08] Multiple`address` mappings can be combined into a single mapping

### Context
[IdentityStaking.sol#L87-L94](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L87-L94)

### Description
The `userTotalStaked`, `selfStakes` and `communityStakes` mappings all map from an address to different staking-related data associated with that address. Combining them into one mapping may improve readability and lead to gas savings, since `userTotalStaked` is always modified when either `selfStakes` or `communityStakes` is.

### Recommendation
Combine these mappings into a single mapping from an address to a struct (e.g. `StakingInfo`). 

## [N-09] Layout order does not comply with best practices

### Context
[IdentityStaking.sol#L25-L162](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L25-L162)

### Description
The order of declaration of contract elements in the `IdentityStaking` contract does not follow [best practices](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-layout) .

### Recommendation
Use the following recommended order:
1. Type declarations
2. State variables
3. Events
4. Errors
5. Modifiers
6. Functions
