# Biconomy Nexus — Execution Mode / Execution Helper Hunt

**Date:** 2026-11-07
**Target:** bcnmy/nexus @ 9007ba1 (main)
**Program:** security@biconomy.io — Critical up to $50,000 / High up to $10,000
**Issue class:** execution-mode decoding (ModeLib/ExecLib) + execution helpers (try/catch, batch, delegatecall)
**Result:** NO FINDING — codec verified empirically correct. Details below.

## The candidate that wasn't (important negative result)

`ModeLib.decode` reads the calltype with `_calltype := mode` (low byte of the stack word)
while `ModeLib.encode` packs the calltype as the FIRST byte of `abi.encodePacked`
(most-significant byte of the resulting bytes32). Read naively, `encodeSimpleBatch()`
(0x0100...00) would decode as CALLTYPE_SINGLE — a batch/single confusion that could
silently misroute execution.

**Empirically disproven** (forge test, solc 0.8.27, isolated project):
- `decode(encodeSimpleBatch()).calltype == CALLTYPE_BATCH` → true
- `decode(encodeTrySingle())` → calltype=SINGLE, exectype=TRY → both correct
- `decode(encodeCustom(DELEGATECALL))` → DELEGATECALL → correct

Root cause of the false alarm: Solidity cleans bytesN values by taking the HIGH bytes
(bytesN is left-aligned by convention), so `_calltype := mode` yields the most-significant
byte — exactly matching the encode layout. `uint8(unwrap(ct))` = 1 for mode 0x0100...00.
Lesson recorded: verify codec semantics by execution, not by reading assembly.

## What was audited (all sound)

### ModeLib (contracts/lib/ModeLib.sol)
- Layout: [calltype 1B][exectype 1B][unused 4B][modeSelector 4B][modePayload 22B], MSB-first.
- decode/decodeBasic extract fields correctly (verified by test).
- Unknown calltype → `revert UnsupportedCallType` (Nexus.sol:153); unknown exectype →
  `revert UnsupportedExecType` (ExecutionHelper). Fail-closed.

### ExecLib (contracts/lib/ExecLib.sol)
- `decodeBatch`: full bounds checks on offset/length (`shr(64, u)` dirty-upper-bits check +
  `gt(add(s, shl(5, len)), e)` range check + per-entry checks). Reverts DecodingError on
  malformed input. Copied from Solady LibERC7579 (audited upstream).
- `decodeSingle`: slices [0:20]/[20:52]/[52:] — reverts on short calldata (fail-closed).
- `decodeDelegateCall`: [0:20]/[20:] — same.

### ExecutionHelper (contracts/base/ExecutionHelper.sol)
- `_execute*`: `codesize()` as call out-size is a gas optimization; full returndata is
  re-copied via returndatacopy after the call. Memory allocation correct.
- TRY variants catch reverts and emit TryExecuteUnsuccessful — no state corruption.
- Batch DEFAULT reverts atomically on any failure (documented semantics); TRY continues.

### Hooks & registry (contracts/base/ModuleManager.sol, RegistryAdapter.sol)
- Single hook slot; install requires isModuleType + empty slot; uninstall uses
  `excessivelySafeCall` (bounded gas — no uninstall griefing).
- `withHook`: preCheck → body → postCheck; validateUserOp correctly NOT hooked
  (pre-validation hooks are a separate, explicit module type).
- ERC-7484 registry is opt-in: `_checkRegistry` skips when unset; `setRegistry` is
  `msg.sender == address(this)` only.
- `_uninstallValidator`: sentinel-list pop with caller-supplied `prev` (reverts on wrong
  prev), then bounded onUninstall call.

## Conclusion

No payout-ready finding in the execution-mode / execution-helper class for
bcnmy/nexus @ 9007ba1. Remaining unaudited in-repo surface: K1ValidatorFactory,
BiconomyMetaFactory, BootstrapLib parsing details, and the external
@biconomy/composability dependency (likely out of the program's "main branch" scope).
