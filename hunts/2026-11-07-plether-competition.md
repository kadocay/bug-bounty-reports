# Plether Audit Competition — Hunt (IN PROGRESS)

**Date:** 2026-11-07
**Program:** Immunefi Audit Competition | Plether — https://immunefi.com/audit-competition/plether/information/
**Status:** LIVE — ends 2026-11-16 10:00 UTC (9 days left at hunt start)
**Reward pool:** $80,000 USDC ($56k primary + $16k all-stars + $8k podium); pool unlocks fully if ANY valid bug found
**Why this target:** NO KYC required for payout (explicit on program page), NO reputation requirement, public repo, frozen code, small scope (21.4k LoC). Fits owner's constraints (new account, no KYC, no rep).
**Repo:** https://github.com/Plether-Fi/plether-core @ frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8
**Local clone:** ~/.hermes/cache/scratch/plether

## Program facts (verified from program page)

- Arbitrum, Solidity 0.8.35. Perps on a USD index (basket: EUR/JPY/GBP/CAD/SEK/CHF).
- Collateral USDG (6 dec). LP shares = ERC-4626 vaults implementing ERC-7540 + ERC-7575 (async requests).
- Two tranches: Senior (targeted coupon, absorbs losses last) / Junior (absorbs losses first).
- Orders priced by Pyth via delayed-order model; one global binding-order queue; frozen markets on weekends/holidays.
- Runnable PoC required for all severities. Submission fee applies per report (Immunefi).
- Known issues + prior audit findings (audits/ dir: pre-audit, collaborative 2026-05-12, AI audit 2026-10-26, Cantina report) are EXCLUDED — read before reporting to avoid duplicates.
- Code frozen at a82d4fe while live. Not deployed on mainnet; v2.0.0 on Arbitrum Sepolia for testing.
- Owner must register on Immunefi (free account) to submit — agent cannot create accounts.

## Audited so far (2026-11-07) — no findings

### CfdMath.sol (182 lines) — CLEAN
- `calculateExactPnl`: lots × price exact integer math, no division; price capped at capPrice; LONG/SHORT semantics consistent with basket-priced-in-USD (LONG USD profits when basket mark falls).
- `calculateExactMaxProfit`: LONG max = entryCost; SHORT max = capValue − entryCost. Correct.
- `tryGetSkewCost`: 0.5·k·S²/D with WAD scaling; overflow guards on every intermediate (skewWad, skewWad², factor multiply). Fail-closed via boolean.
- `tryCalculateVPI`: postCost − preCost as int256; both costs bounded so cast is safe.

### HousePoolRedemptionMathLib.sol (109 lines) — CLEAN
- `netAssetsForShares`: floor(shares·(principal+virtualAssets)/(supply+virtualShares)) then floor(·(BPS−fee)/BPS). Standard ERC-4626 + virtual offsets; fee rounded down (protocol-favoring, correct direction).
- `maxSharesForNetBudget`: exact constant-time inversion of both nested floors: grossCap = ceil((budget+1)·BPS/feeAdj)−1; shares = ceil((grossCap+1)·supply/assets)−1; clamped to maxShares; recomputed payout asserted ≤ budget and maximality asserted (fundedShares+1 would exceed budget). budget+1 overflow excluded by the partial-fill precondition (maxNet > budget ≥ ... ⇒ budget < uint256.max).

## Remaining audit plan (priority order)

1. TrancheVault.sol (1875) — share accounting, async request flow (ERC-7540), tranche loss waterfall
2. HousePool.sol (1886) — pool accounting, redemption entry points
3. MarginClearinghouse.sol (1833) — margin/liquidation math
4. CfdEngine.sol (1484) — position lifecycle, PnL settlement
5. PletherOracle.sol (1070) — Pyth integration, delayed-order model
6. OrderRouter + sidecars (OrderLifecycleBook, PositionProtectionBook, recovery/liquidation sidecars)
7. Spec-vs-code: ACCOUNTING_SPEC.md / WHITEPAPER.md vs implementation

## Rules reminder

Research/audit only — no live testing, no submission. Owner reviews and submits under his own Immunefi account.
