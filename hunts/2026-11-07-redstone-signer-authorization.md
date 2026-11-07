# Hunt Log — 2026-11-07 — RedStone Smart Contracts — issue class: signer authorization / signature validation

**Program:** RedStone Smart Contracts (HackenProof) — https://hackenproof.com/programs/redstone-smart-contracts
**Payouts:** Critical $50k–$250k / High $10k–$50k / Medium $1k–$10k / Low $50–$1k. $10 submission fee, 150 reputation required.
**Scope audited (from program page):** `redstone-oracles-monorepo/packages/evm-connector/contracts`, `rust-sdk/crates/redstone`, and the stellar/sui/solana/radix connectors.
**Repos cloned:** github.com/redstone-finance/redstone-oracles-monorepo (depth 1), github.com/redstone-finance/rust-sdk (depth 1).
**Result: NO payout-ready finding this session.** One candidate analyzed and dropped (unproven reachability). Details below.

## What was audited (signer authorization / signature validation class)

### EVM connector (contracts/core, contracts/libs, contracts/data-services)
- `libs/SignatureLib.sol` — `recoverSignerAddress`: rejects v ∉ {27,28}, rejects s > n/2 (malleability), rejects ecrecover → address(0). Sound.
- `core/RedstoneConsumerBase.sol` — `_extractDataPackage`: signature is recovered and `getAuthorisedSignerIndex` (reverts on unknown signer) is called **before** any value is recorded (lines 239–243); per-feed unique-signer bitmap + threshold enforced; `InsufficientNumberOfUniqueSigners` reverts if any requested feed is under threshold. Sound.
- `core/CalldataExtractor.sol` — RedStone marker check, metadata size bounds, package count bounds. Sound.
- `core/RedstoneConsumerBytesBase.sol` — tricky calldata pointers bounds-checked (`calldataOffset + valueByteSize > msg.data.length` → revert). Sound.
- `libs/NumericArrayLib.sol` — median; Solidity 0.8 checked arithmetic on the sum. Sound.
- `core/ProxyConnector.sol` — forwards payload + encoded call; return-value handling. No signature-logic surface.

### Rust SDK (rust-sdk/crates/redstone) — shared by all non-EVM connectors
- `crypto/mod.rs` — `recover_address`: length check, malleability (r≠0, s≠0, r<n, s≤n/2), recovery byte 0/1/27/28, address = keccak(pubkey)[12..]. Sound.
- `protocol/payload_decoder.rs` — line 89: `recover_address(...).ok()` — **invalid signatures do not error; the package is kept with `signer_address: None`.** Fail-closed downstream (see aggregator).
- `core/aggregator.rs` — `process_values`: packages with `signer_address: None` or unknown signer are skipped; per-feed threshold enforced; duplicate (signer, feed) → error; zero values skipped. Sound (fail-closed).
- `protocol/payload.rs` — `get_validated_timestamp` uses the **first package's timestamp regardless of signature validity**, then requires all packages to share it. Analysis: since values only come from validly-signed packages whose timestamps are inside the signed message, and all packages must share the first package's timestamp, the returned timestamp always equals a signature-protected timestamp — or the payload is rejected. Not exploitable for value manipulation.
- `core/validator.rs`, `core/config.rs`, `contract/verification.rs` — timestamp window checks; config validation (threshold ≤ signers, ≤255 signers, no zero/duplicate signers, feeds non-empty/unique/valid). Sound.

### Connector glue
- `radix-connector/.../common/src/{verify.rs, process_payload.rs}` — thin wrappers over the SDK. Sound.
- `solana-connector/.../write_price.rs` — requests a **single feed**, guards `values.is_empty()`, takes `values[0]`. Correct pattern.
- `stellar-connector/.../redstone-adapter/src/lib.rs` — iterates `FeedValue { value, feed }` (maps **by feed id**) and checks `prices.len() != feed_ids.len()` → error. Correct pattern.

## Dropped candidate (analyzed, NOT reported — unproven reachability)

**Feed/value misalignment in `radix-connector` price adapter** — `packages/radix-connector/deployments/radixMultiFeed/contracts/price_adapter/src/price_adapter.rs`, `process_payload_and_write`, line 225–251:

```rust
feed_ids.into_iter().map(|v| v.into()).zip(values.clone()).for_each(|(feed_id, value)| {
    ... self.prices.insert(feed_id, PriceDataRaw { price: value, timestamp, ... });
});
```

- The SDK's `aggregate_values` (rust-sdk `core/aggregator.rs:61–86`) **silently omits** requested feeds whose signer count is below threshold and returns a **compacted** `Vec<FeedValue>` (each entry carries its feed id).
- The Radix adapter zips the requested `feed_ids` with that compacted vector **positionally**, with no length check and no feed-id matching.
- If any requested feed at index *i* is below threshold while a later feed passes, every value from the first omitted index onward is written under the **wrong feed id** (stored price for feed A becomes feed B's value).
- Contrast: the EVM connector **reverts** (`InsufficientNumberOfUniqueSigners`) if any requested feed is under threshold; the Solana connector requests one feed + empty guard; the Stellar connector maps by feed id + length check. The Radix adapter is the only outlier.

**Why dropped:** triggering it requires a payload where per-feed signer coverage differs (feed A below threshold, feed B at/above). With the standard RedStone signer set every signer signs the full feed set, so coverage is identical across feeds — an attacker submitting any subset of genuine packages produces equal coverage for all requested feeds (all pass or all fail). I could not construct a reachable payload with the deployed configuration, and the program's rules exclude "theoretical vulnerabilities without any proof or demonstration." Per the hunt rules ("drop anything unproven"), this is documented, not reported.

**Trigger condition (for a future session):** a deployment whose configured signers sign different feed subsets (or a requested feed that no signer signs, placed before a signed feed in the request list). If the owner confirms such a configuration exists on a live deployment, this becomes reportable — the fix is to map by `FeedValue.feed` (like the Stellar connector) or to length-check and revert.

## Next-session options

1. Same program, different issue class: business logic in a specific connector (e.g., Radix `NibBugBounty`-style escrow flows are out of scope here; look at `price_adapter` update-verifier interactions: trusted vs untrusted updater paths, `MIN_TIME_BETWEEN_UPDATES_MS` bypass).
2. Same program, aggregation/median edge cases (even-count median, value-byte-size boundary 32 vs 33 bytes).
3. Different program with a smaller, newer surface: Ticko (200k USDT, direct email, testnet allowed) or Plether (contest ends Nov 16).

## Verification artifacts

- Foundry 1.9.0 installed (for EVM PoCs); Rust toolchain not installed (SDK tests not run this session — existing SDK test suite already covers the fail-closed paths cited above: `test_aggregate_values_invalid_signer_address`, `test_aggregate_values_unknown_signer`, `test_aggregate_values_insufficient_signers`).
- No code was modified in the audited repos; clones are read-only at `~/.hermes/cache/scratch/{redstone,rust-sdk}`.
