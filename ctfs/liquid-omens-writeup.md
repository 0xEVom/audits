# Liquid Omens CTF Writeup

`Liquid Omens` looked like a compact DeFi liquidation challenge: one unhealthy borrower in `BadDebtMarket`, a keeper-gated `LiquidationEngine`, and a `LiquidationVenue` already funded with enough `oUSD` to clear the solve threshold. The apparent task was to take a signed liquidation directive and turn it into a payout big enough to solve.

The real problem was narrower and stranger. Nothing had to be forged, and the liquidation math was not the point. The keeper signed one interpretation of a command directive, while the venue executed another, so the whole challenge was about keeping a real authorization valid while changing which command the venue eventually decoded.

```solidity
function isSolved() external view returns (bool) {
    return quoteToken.balanceOf(PLAYER) >= 900 ether;
}
```

## Recovering The Signature

The first useful clue was the owner-only `LiquidationEngine.recordDirective(...)` setup call. It did not emit a dedicated event, but once you found it in the engine's short setup history, that path filed a 65-byte keeper signature into custom storage slots derived from a fixed domain separator. The journal key was not supplied directly. Instead, the engine derived it internally from the per-instance player and the filing block number.

```solidity
uint256 filingKey = uint256(keccak256(abi.encode(DIRECTIVE_KEY_DOMAIN, player, filingBlock)));
uint256 directiveBase = uint256(keccak256(abi.encode(DIRECTIVE_JOURNAL_DOMAIN, filingKey)));
```

In the current implementation, the easiest recovery path still started from that setup transaction itself: once you identified the `recordDirective(...)` call, you could decode its lone `bytes` argument directly from calldata. The engine also journaled those same 65 bytes into storage, so an equivalent recovery path was to take that transaction's block number, derive `filingKey` from `player` and `filingBlock`, recompute `directiveBase` off-chain, and then read the directive length from `directiveBase` and the subsequent words from `directiveBase + 1`, `directiveBase + 2`, and so on.

Recovering those bytes from storage gave the only stored authorization artifact:

```solidity
abi.encodePacked(r, s, v)
```

So there was no need to forge anything. The challenge already gave you a valid keeper-approved signature. The missing piece was the canonical directive itself: test the small set of plausible zero-argument venue command surfaces offline until one preview digest recovered `keeperSigner`.

What that signature actually authorized was the canonical directive:

```solidity
abi.encode(
    DirectivePreviewLib.liquidationDirectiveKind(),
    abi.encodeCall(LiquidationVenue.highlyProfitableTradingStrategy, ())
)
```

That directive really did work. It just paid very little. The signed command was `highlyProfitableTradingStrategy()`, which sounded like the obvious jackpot path but in practice only transferred `(activeCollateral * 3) / 8`, which came out to `60e18` in the seeded setup, so it could not solve the challenge. The venue also exposed `netInventory()`, which looked like another plausible signed command surface but only paid `activeCollateral / 2`, or `80e18` in the same setup. Neither of those two plausible signed surfaces was enough. It was also one-shot: if you spent the canonical signed directive first, the borrower was liquidated and the instance was dead.

The public entrypoint stayed `LiquidationEngine.liquidate(bytes directive, bytes signature)`, so the outer call still looked like a standard liquidation flow. The liquidation context itself was derived from visible instance state: the engine took the active borrower from `market.currentBorrower()`, the debt from `market.positions(borrower).debt`, and the beneficiary from its per-instance `PLAYER`. Inside the venue, both `highlyProfitableTradingStrategy()` and `netInventory()` looked like plausible command surfaces but only paid bounded slices of quote. The function you actually wanted the venue to self-call was `settleLot()`. That one looked like the boring internal settlement path, emitted `LotSettled`, and burned the seized collateral, while quietly paying out the venue's staged quote inventory. It was guarded by `onlySelf`, so you could not call it directly. You had to make `LiquidationVenue.executeDirective()` decode it as the directive command and then self-call it internally.

## Two Decoders

The bug appeared because the same `directive` bytes were read twice, but not by the same rules. `DirectivePreviewLib.load()` reconstructed the preview the keeper supposedly signed by reading the directive head and then reading the command from fixed ABI positions. It did not honor the encoded offset word for the dynamic `bytes command`.

```solidity
assembly {
    let base := directive.offset

    mstore(preview, calldataload(base))

    commandDataOffset := add(base, 0x60)
    commandLength := calldataload(add(base, 0x40))
}
```

In practice that meant the verifier hashed whatever bytes were sitting in the first command tail, regardless of what offset the directive encoded. `LiquidationVenue.executeDirective()`, on the other hand, used normal ABI decoding:

```solidity
(uint256 directiveKind, bytes memory command) =
    abi.decode(directive, (uint256, bytes));
```

That path did honor the dynamic offset. So the verifier trusted fixed positions, while the executor trusted the offset word.

The directive layout was the familiar ABI layout:

```text
0x00  directive_kind
0x20  offset_to_command
0x40  command_length
0x60  command_bytes...
```

In the canonical signed directive, the offset at `0x20` was `0x40`, so the first command tail began where the verifier expected. The exploit worked by leaving that first tail untouched and only changing where the executor looked.

## Offset Smuggling

The smuggled directive was built by copying the original directive, rewriting the offset word, and appending a second command tail after the original directive body. The malicious command was:

```solidity
abi.encodeCall(LiquidationVenue.settleLot, ())
```

The new offset could simply be `directive.length`, because that was where the appended length word began in the extended blob:

```solidity
uint256 smuggledCommandOffset = directive.length;

smuggled = new bytes(directive.length + 0x20 + paddedCommandLength);

_copy(smuggled, 0, directive, 0, directive.length);
_storeWord(smuggled, 0x20, smuggledCommandOffset);
_storeWord(smuggled, smuggledCommandOffset, maliciousCommand.length);
_copy(smuggled, smuggledCommandOffset + 0x20, maliciousCommand, 0, maliciousCommand.length);
```

The verifier still hashed the original command tail at `0x40` and `0x60`, while the executor followed the rewritten offset and decoded the appended tail instead.

The signature continued to verify because the preview digest still depended on the same values: directive kind and the hash of the original canonical command bytes in the first tail. As long as the head and that first tail were preserved, the signed preview was unchanged. The appended malicious tail was invisible to the verifier because the verifier never followed the encoded offset.

At runtime the exploit was short. The call into `LiquidationEngine.liquidate(...)` first caused `DirectivePreviewLib.load()` to reconstruct the preview from fixed positions, then accepted the original keeper signature, derived the liquidation context from market state, and forwarded the same `smuggledDirective` into `LiquidationVenue.executeDirective()`. The venue then decoded the directive using `abi.decode`, followed the rewritten offset into the appended tail, decoded `settleLot()`, and transferred the staged quote inventory to `activeBeneficiary`. The protocol did not fail because the signature was wrong; it failed because it signed one meaning and executed another.

That was the whole lesson of the challenge. If a protocol signed or hashed dynamic data, the verifier and the executor needed to parse the same structure and honor the same offsets. As soon as those paths diverged, "validly signed" and "actually executed" stopped meaning the same thing.
