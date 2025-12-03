## Across: Theft of in-flight user funds via nonce manipulation and signature replay

### Introduction

Across is a cross-chain optimistic bridge that coordinates L2/L3 SpokePools with an L1 HubPool: independent relayers front liquidity to recipients on destination chains and are later refunded from the HubPool after an optimistic challenge window. In April 2025, Radiant Labs identified a vulnerability that would have allowed an attacker to steal in‑flight user funds by combining: (1) a missing field in the EIP‑712 witness used for gasless ERC‑7683 orders, and (2) insufficient binding of “speed up” update signatures to a unique deposit. Together these issues let a relayer front‑run and redirect a later (larger) deposit to settle under parameters authorized for an earlier (smaller) deposit, while remaining indistinguishable to off‑chain validators.

### Relevant mechanics

- Gasless orders (first‑party meta‑tx): users can authorize cross‑chain transfers without paying origin‑chain gas themselves. A relayer submits a signed ERC‑7683 order via Permit2; the order’s integrity depends on the EIP‑712 witness for the embedded `AcrossOrderData`, computed by [`hashOrderData()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/ERC7683Permit2Lib.sol#L97-L113).
- Deterministic deposit IDs for nonzero `depositNonce`: the flow routes to [`depositV3()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L589-L617) or [`unsafeDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L651) based on `depositNonce`, selected by [`_callDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/AcrossOriginSettler.sol#L63-L94). With a nonzero nonce, the `depositId` is derived deterministically via [`getUnsafeDepositId()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L1290-L1296) from `(settler, depositor, depositNonce)`.
- Update (“speed up”) signatures: depositors can adjust terms after submission (for example, lower `updatedOutputAmount` or change `updatedRecipient`), and relayers can honor those updates using [`fillRelayWithUpdatedDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L1026-L1062). The authorization is verified by [`_verifyUpdateV3DepositMessage()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L1560-L1587) against `(depositId, originChainId, updatedOutputAmount, updatedRecipient, updatedMessage)`.

These mechanics together explain how deposits are identified and how updates are authorized.

### Root cause

1. Missing field in the EIP‑712 witness for gasless orders: `AcrossOrderData`'s EIP‑712 type includes `depositNonce` (see type at [`ERC7683Permit2Lib.sol`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/ERC7683Permit2Lib.sol#L29-L41)), but [`hashOrderData()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/ERC7683Permit2Lib.sol#L97-L113) omitted it from the struct hashing (highlighted below). This allowed a relayer to freely change `depositNonce` in the on‑chain call while still satisfying the user’s signature.

```solidity
// ERC7683Permit2Lib.hashOrderData() - depositNonce omitted
function hashOrderData(AcrossOrderData memory orderData) internal pure
returns (bytes32) {
    return keccak256(
        abi.encode(
            ACROSS_ORDER_DATA_TYPE_HASH,
            orderData.inputToken,
            orderData.inputAmount,
            orderData.outputToken,
            orderData.outputAmount,
            orderData.destinationChainId,
            orderData.recipient,
            orderData.exclusiveRelayer,
            orderData.exclusivityPeriod,
            keccak256(orderData.message)
        )
    );
}
```

2. Update‑signature scope too narrow: [`_verifyUpdateV3DepositMessage()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L1560-L1587) bound the update signature to `depositId` and `originChainId` (plus updated fields), but not to the full relay uniqueness (e.g., the `relayHash`). With [`unsafeDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L651), a relayer could cause a later deposit to reuse a prior `depositId` by choosing the same `depositNonce`, which made earlier update signatures valid for the new deposit.

```solidity
// SpokePool._verifyUpdateV3DepositMessage() - bound to depositId + originChainId
_verifyUpdateV3DepositMessage(
    relayData.depositor.toAddress(),
    relayData.depositId,
    relayData.originChainId,
    updatedOutputAmount,
    updatedRecipient,
    updatedMessage,
    depositorSignature,
    UPDATE_BYTES32_DEPOSIT_DETAILS_HASH
);
```

### Exploit flow

1) The victim signs a small gasless order via [`openFor()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/ERC7683OrderDepositor.sol#L55-L87) using a nonzero `depositNonce = N`. The relayer submits it; the deposit emits with `depositId = keccak256(address(OriginSettler), victim, N)`.
2) The victim “speeds up” that small deposit by signing an update allowing a reduced `updatedOutputAmount` and/or a different `updatedRecipient` via [`speedUpV3Deposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L881-L909) / [`speedUpDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L827-L856). The signature is now public on‑chain.
3) Later, the victim creates a much larger gasless order (e.g., 100 ETH). Because [`hashOrderData()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/erc7683/ERC7683Permit2Lib.sol#L97-L113) omits `depositNonce`, a malicious relayer can front‑run and set `depositNonce = N` in `openFor()`, forcing the new deposit’s `depositId` to equal the small deposit’s `depositId`.
4) The attacker calls [`fillRelayWithUpdatedDeposit()`](https://github.com/across-protocol/contracts/blob/69cc1dbdeea1a724a81ef423a0c69b6393275ecc/contracts/SpokePool.sol#L1026-L1062) for the new (large) deposit, supplying the previously emitted update signature (valid for `(depositId = N, originChainId, ...)`). Signature verification passes, and the relay settles under the attacker‑favorable updated parameters (e.g., smaller `updatedOutputAmount`).
5) Off‑chain validation and refunds proceed as if this were a legitimate updated fill; the attacker captures the spread between the large input and the smaller updated output.

Effectively, the attacker rebinds the victim’s earlier update authorization to a later, larger deposit by reusing the `depositId` via a malleable `depositNonce`.

The Across team assessed the severity as Medium because the `openFor()` flow was not supported through the frontend and exploitation required a non‑EIP‑712‑compliant signature to be constructed and signed by the user, which significantly reduced the attack surface versus standard deposit flows.

### Proof of concept

A Foundry PoC for the full end‑to‑end attack is available in [this gist](https://gist.github.com/0xEVom/8b95545e2367c9fd4260f2dc6f131751).

### Remediation

Upon receiving our report, the Across team implemented the following fixes:

- Included `depositNonce` in the `AcrossOrderData` EIP‑712 struct hashing computed by [`ERC7683Permit2Lib.hashOrderData()`](https://github.com/across-protocol/contracts/commit/644239d408e7d132da4a47632567c052f03b0058#diff-7cb8e9d0d8784e1523d6435c7b786c9be27dced64165e7c9f7c15a73e2e97ab1R112), restoring the uniqueness of signed gasless orders with respect to `depositNonce`.
- Documented the replay risk when users reused a `depositNonce` with [`SpokePool.unsafeDeposit()`](https://github.com/across-protocol/contracts/commit/644239d408e7d132da4a47632567c052f03b0058#diff-c828c513da79ba8aaef90514612391ab755989974a14d8e480ae02ebd9cfb4beR623-R625), clarifying the implications for `depositId` determinism and update‑signature scope.

### Conclusion

This issue illustrates how a single omitted field in an EIP‑712 witness, combined with narrowly scoped signature checks, can enable high‑impact fund redirection in cross‑chain systems. Across addressed the core cause promptly. When designing meta‑transaction and cross‑chain authorization flows, ensure all user‑controlled inputs are covered by the signed payload and that update authorizations bind to a unique on‑chain identity that cannot be recreated by an adversary.

