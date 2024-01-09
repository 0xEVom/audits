## [L-01] Protocol is not ERC-1155 compliant
_This issue was downgraded from Medium to Low_

### Impact
The `Ocean` contract is not fully compliant with the ERC-1155 standard. Specifically, the `onERC1155Received` and `onERC1155BatchReceived` [functions](https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/Ocean.sol#L317-L366) do not revert when they reject a transfer. According to the [ERC-1155 standard](https://eips.ethereum.org/EIPS/eip-1155#erc-1155-token-receiver), these functions MUST revert if they reject a transfer.
This may have been a deliberate choice by the authors given the comments in the functions' documentation, but ERC-1155 adherence is explicitly listed as an invariant in the contest documentation and this behaviour represents strict non-conformity.

### Proof of Concept
This could lead to a potential loss of funds if an external IERC1155 contract is improperly implemented and doesn't check for the return value of `onERC1155Received`. In such a case, a user could unknowingly transfer tokens to the `Ocean` contract without a proper interaction. Since the `Ocean` contract doesn't revert the transaction, the tokens would be transferred successfully, but without the required interaction to credit them to the user in the ocean. This could lead to the tokens being stuck in the `Ocean` contract, resulting in a loss of funds for the user.

### Tools Used
Manual inspection

### Recommended Mitigation Steps
To ensure full compliance with the ERC-1155 standard, modify the `onERC1155Received` and `onERC1155BatchReceived` functions to revert when a transfer is rejected. This can be achieved by replacing the `return 0;` statement with a `revert` statement.

## Assessed type

Other

## [L-02] Improper external calculation of unwrap fee

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/OceanAdapter.sol#L70-L74

### Description

In the `computeOutputAmount` function of `OceanAdapter.sol`, the `unwrappedAmount` is calculated by subtracting the `unwrapFee` from the `inputAmount`. This `unwrappedAmount` is then passed as a parameter to the `primitiveOutputAmount` function. However, the actual unwrap fee may be higher than that calculated in this function due to truncation, in which case the value passed to the `primitiveOutputAmount` function is higher than the actual available amount. 

The code in question:

```solidity
uint256 unwrapFee = inputAmount / IOceanInteractions(ocean).unwrapFeeDivisor();
uint256 unwrappedAmount = inputAmount - unwrapFee;
outputAmount = primitiveOutputAmount(inputToken, outputToken, unwrappedAmount, metadata);
```

The code implicitly assumes that Adapters need to truncate the `unwrappedAmount` to the token's decimals. 

### Recommendation 

Instead of attempting to recalculate the fee outside of the Ocean, the unwrapped amount should be inferred from the balance change and passed to `primitiveOutputAmount`, ideally already in the token's precision. This would ensure that the correct amount is used in the calculations.


## [L-03] Incorrect event parameter in swap action

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/CurveTricryptoAdapter.sol#L230-L234
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L178-L182

### Description

In the `primitiveOutputAmount` function, the `inputAmount` parameter is used in the `Swap` event without being converted to `rawInputAmount`. This could potentially lead to incorrect event logs as the `inputAmount` might not reflect the actual amount used in the swap operation due to it being truncated in the decimal conversion. The `rawInputAmount` is the value that is actually used in the swap operation, and it is derived from `inputAmount` after adjusting for decimal differences.

Relevant code:
```solidity
if (action == ComputeType.Swap) {
    emit Swap(inputToken, inputAmount, outputAmount, minimumOutputAmount, primitive, true);
}
```

### Recommendation 

To ensure the event logs accurately reflect the operations performed, it is recommended to emit the `Swap` event with `rawInputAmount` instead of `inputAmount`. This will ensure that the logged amount is the actual amount used in the swap operation. 


## [L-04] Incorrect documentation regarding int256 maximum value

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/BalanceDelta.sol#L58-L67

### Description

The documentation for the `safeCast` modifier suggests that the maximum value of `int256` is one unit higher than the absolute value of the minimum value. However, this is incorrect. The maximum value of `int256` is actually one unit lower than the absolute value of the minimum value. This discrepancy could lead to confusion and potential errors in the future if the code is misunderstood. The relevant code is:

```solidity
if (uint256(type(int256).max) <= amount) revert CAST_AMOUNT_EXCEEDED();
```

In addition, given the above, it is also safe to use strict equality in the above comparison, since type(int256).max is safely castable to int256 (and also has a negative representation).

### Recommendation 

The documentation should be corrected to accurately reflect the properties of `int256`. The correct statement should be: "the maximum value of `int256` is one unit lower than the absolute value of the minimum value". This will ensure that the documentation is accurate and does not lead to potential misunderstandings or errors in the future.


## [L-05] Unnecessary fallback function

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/CurveTricryptoAdapter.sol#L30-L53

### Description

The contract `CurveTricryptoAdapter` includes a fallback function that is not necessary. The fallback function is an unnamed function that is executed whenever the contract is called with function data that does not match any of the existing functions. However, in this case, it is not required and can be replaced with a `receive()` function which is triggered when the call data is empty.

### Recommendation

Remove the fallback function and replace it with a `receive()` function. 


## [L-06] Missing sender address check

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/CurveTricryptoAdapter.sol#L291

### Description

The fallback function in the `CurveTricryptoAdapter` contract does not include a check for the sender's address. This could potentially lead to Ether being locked in the contract if it is sent from any address other than the Ocean.

### Recommendation 

Add a check for the sender's address in the fallback/receive function to ensure that Ether is only accepted from the Ocean.


## [N-01] slippageProtection type and naming

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L30-L53

### Description

In the `Swap`, `Deposit` and `Withdraw` event declarations, the `slippageProtection` parameter is currently of type `bytes32`. However, considering its usage and context, it would make more sense for it to be of type `uint256`.

Additionally, the name `slippageProtection` could be improved to better reflect its purpose. A more appropriate name could be `minOutput`, which clearly indicates that it represents the minimum output amount the user expects to receive, providing protection against price slippage.

```solidity
event Swap(
    uint256 inputToken,
    uint256 inputAmount,
    uint256 outputAmount,
    bytes32 slippageProtection,
    address user,
    bool computeOutput
);
```

### Recommendation 

Change the type of `slippageProtection` to `uint256` and rename it to `minOutput`.


## [N-02] Inconsistent accounting of ETH wrapping as interactions

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/Ocean.sol#L216
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/Ocean.sol#L243
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/Ocean.sol#L266
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/Ocean.sol#L297

### Description

In the Ocean contract, there is an inconsistency in how the wrapping of ETH is accounted for as interactions. When ETH is wrapped in a call to `doInteraction` or `forwardedDoInteraction`, this is counted as an interaction in the `numberOfInteractions` parameter of the emitted `OceanTransaction`/`ForwardedOceanTransaction` event. However, if ETH is wrapped as part of a call to `doMultipleInteractions` or `forwardedDoMultipleInteractions`, the emitted event does not account for that action as an interaction. This inconsistency could lead to confusion and potential misinterpretation of the events.

### Recommendation

It is recommended to check for `msg.value` in the `doMultipleInteractions` and `forwardedDoMultipleInteractions` functions and add 1 to the number of interactions if present. This would ensure that the wrapping of ETH is consistently accounted for as an interaction across all relevant functions.


## [N-03] Potential revert in _getNegativeBalanceDelta function

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/ocean/BalanceDelta.sol#L299

### Description

In the `_getNegativeBalanceDelta` function, there is a potential edge case where the function will revert if `amount` equals `type(int256).min`. This is due to the line of code `return uint256(-amount);`. In the case where `amount` equals `type(int256).min`, negating `amount` will result in a value that cannot be represented as an `int256`, causing an overflow.

```solidity
function _getNegativeBalanceDelta(BalanceDelta[] memory self, uint256 tokenId) private pure returns (uint256) {
    uint256 index = _findIndexOfTokenId(self, tokenId);
    int256 amount = self[index].delta;
    if (amount > 0) revert DELTA_AMOUNT_IS_POSITIVE();
    return uint256(-amount);
}
```

While this is an edge case, it could potentially disrupt the execution of transactions that hit this condition.

### Recommendation 

To handle this edge case, an option would be to check if a balance equals `type(int256).min` after it has been decreased and revert if that is the case. However, the current behaviour is benign and more gas-efficient.


## [N-04] Misleading function name in Curve2PoolAdapter contracts

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L201C21-L201C21
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L201

### Description

In the `Curve2PoolAdapter` and `CurveTricryptoAdapter` contracts, the function `_determineComputeType` not only determines the type of computation (Swap, Deposit, Withdraw) based on the input and output tokens, but also validates the tokens. The function name `_determineComputeType` does not accurately reflect this validation behavior. Misleading function names can lead to confusion for developers and auditors, potentially leading to overlooked issues or bugs.

Relevant code:
```solidity
if (((inputToken == xToken) && (outputToken == yToken)) || ((inputToken == yToken) && (outputToken == xToken)))
```

### Recommendation 

To improve code readability and maintainability, consider renaming the function to `_determineComputeTypeAndValidateTokens`. This name more accurately describes the function's behavior, which includes both determining the computation type and validating the tokens.


## [N-05] Misleading index variable names

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L159-L160
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/CurveTricryptoAdapter.sol#L195-L196

### Description

In the `primitiveOutputAmount` function, the variable `indexOfInputAmount` is used to store the index of the input token in the Curve pool. The current name of the variable suggests that it is storing the index of an amount, which is misleading. This could lead to confusion for developers reading or maintaining the code. The relevant code is:

```solidity
int128 indexOfInputAmount = indexOf[inputToken];
```

### Recommendation 

To improve code readability and maintainability, it is recommended to rename the variable `indexOfInputAmount` to `indexOfInputToken`. This name accurately reflects the data that the variable is storing.


## [N-06] Inconsistent variable naming for IDs

### Context

https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/Curve2PoolAdapter.sol#L56-L62
https://github.com/code-423n4/2023-11-shellprotocol/blob/main/src/adapters/CurveTricryptoAdapter.sol#L61-L70

### Description

In the `Curve2PoolAdapter` and `CurveTricryptoAdapter` contracts, there is an inconsistency in the naming of variables representing Ocean IDs. The variables `xToken` and `yToken` are named without the 'Id' suffix, while `lpTokenId` includes the 'Id' suffix. This inconsistency can lead to confusion and potential errors when reading or modifying the code.

Relevant code:
```solidity
    /// @notice x token Ocean ID.
    uint256 public immutable xToken;

    /// @notice y token Ocean ID.
    uint256 public immutable yToken;

    /// @notice lp token Ocean ID.
    uint256 public immutable lpTokenId;
```

### Recommendation 

To improve code readability and maintainability, it is recommended to follow a consistent naming convention for similar variables. In this case, either add 'Id' to `xToken` and `yToken` to become `xTokenId` and `yTokenId`, or remove 'Id' from `lpTokenId` to become `lpToken`.

