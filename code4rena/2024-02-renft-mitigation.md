## [H-01] ERC-1155 rentals can be stolen

### Lines of code

[Stop.sol#L293-L297](https://github.com/re-nft/smart-contracts/blob/97e5753e5398da65d3d26735e9d6439c757720f5/src/policies/Stop.sol#L293-L297)  
[Stop.sol#L352-L356](https://github.com/re-nft/smart-contracts/blob/97e5753e5398da65d3d26735e9d6439c757720f5/src/policies/Stop.sol#L352-L356)

### Impact
PR [#13](https://github.com/re-nft/smart-contracts/pull/13) prevents the flash stealing of rented assets by computing the order hash and verifying that the order exists before anything else in `stopRent()` and `stopRentBatch()`.

However, the PR also introduces a change unrelated to this fix by [switching](https://github.com/re-nft/smart-contracts/pull/13/files#diff-4287a9f9c360214754fdf221b6f686e5ecb726f0f1831d8741999861d1ee9ef3R352-L361) the order in which rental orders are removed from storage and rental items are returned to the lender. The removal of the order from storage now happens before the transfer, which introduces a new vulnerability that can be exploited to steal rented ERC-1155 tokens.

### Proof of Concept
1. Alice rents an ERC-1155 token `X` with ID `id` from Bob.
2. Alice obtains another ERC-1155 token `Y` (created by the same contract) with the same `id` in some other way.
3. Alice creates another order to rent token `Y` to herself, resulting in both tokens being deposited in her safe, but also includes another ERC-1155 `Z` in the `offer` array before token `Y`.
4. She then stops the rental from herself.
5. On receiving the `onERC1155Receive()` when token `Z` is sent back to her, she performs a call to the Safe to remove the ERC-1155 with ID `id`.
6. Since the Safe contains 2 tokens with ID `id`, but only one is listed as rented, the transaction will succeed.
7. The Safe then transfers the other token with ID `id` to Alice.

### Tools Used
Manual review

### Recommended Mitigation Steps
Revert this change.

## [M-01] M-10 Mitigation Error

### Vulnerability
The original vulnerability constituted the ability of rental safe owners to indefinitely use an outdated guard policy contract, even after a newer version has been deployed. This situation arises because the protocol lacked a mechanism to enforce the update of guard policies across all safes, potentially leaving some safes operating under less secure or outdated policies.

### Mitigation
The mitigation introduces a protocol-wide capability to deactivate guard policies, aiming to prevent the continued use of outdated or insecure policies. However, this solution inadvertently introduced a new vulnerability: the potential for "bricking" rental safes if their associated guard policy is deactivated without providing a viable path for these safes to update to a new guard policy.

The only way for a safe owner to update the guard, if the one they are using becomes inactive, would be via a whitelisted DelegateCall, in the same manner as they would upgrade the Stop policy as described in the contest [documentation](https://github.com/code-423n4/2024-01-renft/blob/main/docs/protocol-whitelists.md#delegate-call-whitelist). However, the current implementation triggers a revert when a Guard policy is not active, before they can potentially execute a DelegateCall to upgrade the guard.

### Suggestion
Allow inactive guards to call whitelisted delegates. A potential implementation could look as follows:
```diff
diff --git a/src/policies/Guard.sol b/src/policies/Guard.sol
index c7823ca..1e94943 100644
--- a/src/policies/Guard.sol
+++ b/src/policies/Guard.sol
@@ -395,17 +395,22 @@ contract Guard is Policy, BaseGuard {
         bytes memory,
         address
     ) external override {
+        // Disallow transactions that use delegate call, unless explicitly
+        // permitted by the protocol.
+        if (operation == Enum.Operation.DelegateCall){
+            if (!STORE.whitelistedDelegates(to)) {
+                revert Errors.GuardPolicy_UnauthorizedDelegateCall(to);
+            } else {
+                // no need to check the transaction if it is to a whitelisted delegate
+                return;
+            }
+        }
+
         // Check if this guard is active for the protocol.
         if (!isActive) {
             revert Errors.GuardPolicy_Deactivated();
         }
 
-        // Disallow transactions that use delegate call, unless explicitly
-        // permitted by the protocol.
-        if (operation == Enum.Operation.DelegateCall && !STORE.whitelistedDelegates(to)) {
-            revert Errors.GuardPolicy_UnauthorizedDelegateCall(to);
-        }
-
         // Fetch the hook to interact with for this transaction.
         address hook = STORE.contractToHook(to);
         bool hookIsActive = STORE.hookOnTransaction(hook);

```

If this issue is addressed as suggested, it is important to keep in mind that disabled guards will also be able to call whitelisted delegates. It is crucial to ensure that this capability does not introduce any vulnerability.

### Conclusion
While the mitigation strategy addresses the original vulnerability of outdated guard policies, it introduces a new risk of rendering rental safes inoperative.

