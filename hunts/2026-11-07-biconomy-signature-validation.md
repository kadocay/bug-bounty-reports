# Biconomy Nexus — Signature Validation / Access Control Hunt

**Date:** 2026-11-07
**Target:** bcnmy/nexus (main branch, commit 9007ba1)
**Program:** https://github.com/bcnmy/nexus — security@biconomy.io, Critical up to $50,000 / High up to $10,000
**Issue class:** signature validation & access control (validateUserOp, ERC-1271, module auth, init flows)
**Result:** NO FINDING — all checked paths fail-closed. Details below.

## Why this target

Only program found with: public Solidity repo scope + email/GitHub submission + published cash
payout table + no account/KYC requirement. Smart account (ERC-4337/7579) — the money path is
`validateUserOp`, so signature validation is the highest-impact class here.

## What was audited (file:line references)

### 1. Validator selection — Nexus.sol:134, ModuleManager.sol:637
`_handleValidator` reverts `ValidatorNotInstalled` unless the validator is in the account's
sentinel list; nonce-validator = 0 falls back to `_DEFAULT_VALIDATOR`. An attacker cannot inject
an arbitrary validator contract via the nonce or via ERC-1271 `signature[0:20]`.

### 2. K1Validator ECDSA — modules/validators/K1Validator.sol:147, 258
- Uses Solady `tryRecoverCalldata` (malleability-safe; returns address(0) on invalid).
- Accepts raw hash and EIP-191 personal-sign hash — documented behavior; exploiting requires the
  owner to sign the exact userOpHash (phishing, not a contract defect).
- `getOwner` returns the account itself when no owner set → validation fails closed for
  un-initialized accounts.
- `transferOwnership` / `addSafeSender` are keyed by `msg.sender`; direct EOA calls only affect
  the caller's own mapping entry. No cross-account write.

### 3. Module Enable Mode — ModuleManager.sol:155-175
Enable-mode install requires an EIP-712 signature (`_checkEnableModeSignature`, line 515) over
(module, moduleType, userOpHash, moduleInitData) from an already-installed validator. Cannot
install a module without an existing valid signature.

### 4. Emergency uninstall — Nexus.sol:266, ModuleManager.sol:453
EIP-712 signature from installed validator + `nonces[data.nonce]` replay guard + 1-timelock
wait (`_EMERGENCY_TIMELOCK`), stale requests reset after 3x. Sound.

### 5. PREP (EIP-7702) init — Nexus.sol:125-133, 521-551, LibPREP.sol
`r = hash(keccak(initData), salt)`; check is `tryRecover(h(delegation), 27, r, hash(r)) ==
address(this)`. initData is cryptographically bound to the account address (rootless keypair
derived from the signature — confirmed via test `_mine`, TestPREP.t.sol:138-152). A relayer
submitting a PREP userOp cannot substitute different initData: that requires an ECDSA-recovery
preimage (~2^160 work). Post-init `require(!isInitialized())` blocks re-entry.

### 6. initializeAccount (7702 branch) — Nexus.sol:306-334
initData must be signed by `address(this)` itself; `erc7702InitHashes` prevents replay of the
same initData hash. Self-calls skip the check by design (already authorized via execute).

### 7. Initialization / factory — NexusProxy.sol:12-15, ProxyLib.sol:40-47, Initializable.sol
- Proxy address = CREATE2 over initcode that embeds initData → address bound to initData;
  front-running with different initData is impossible.
- Constructor sets transient flag and calls initializeAccount atomically; `requireInitializable`
  now CONSUMES the flag (Initializable.sol:27-38) — this is the fix for the published advisory
  GHSA-q47q-h6x2-f5qg (stale transient flag → same-tx re-init). Verified fixed in this repo.

### 8. EIP-712 domain — ModuleManager.sol:54 (Solady EIP712)
Domain includes chainId + address(this) → no cross-chain or cross-account replay of
enable-mode / emergency-uninstall signatures.

### 9. Misc access control
- `executeUserOp` / `execute` / `executeFromExecutor`: onlyEntryPoint / onlyExecutorModule
  (executors sentinel list, ModuleManager.sol:66).
- UUPS `_authorizeUpgrade`: onlyEntryPointOrSelf; 7702 accounts cannot upgrade this way.
- `_checkInitializedValidators` (Nexus.sol:554): cannot remove the last initialized validator.

## Notes / non-issues observed

- `isValidSignature` reverts (outside try/catch) for 0 < signature.length < 20 — fail-closed.
- PREP test signs the userOp before packing (TestPREP.t.sol:83-86) with a MockValidator; the
  real K1Validator path requires the owner to sign the final packed userOpHash. Functional
  detail, not a vulnerability.
- Solady `tryRecover` high-s malleability is irrelevant here: replay protection is the
  EntryPoint nonce, not signature bytes.

## Conclusion

No payout-ready finding in the signature-validation / access-control class for bcnmy/nexus
@ 9007ba1. The known bug class (initialization re-entry) is already fixed and published.
Suggested next classes if this target is revisited: hook module interactions (withHook
preCheck/postCheck ordering), execution mode decoding (ModeLib/ExecLib), or gas-griefing in
batch execution.
