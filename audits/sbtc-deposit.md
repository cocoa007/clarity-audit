# Security Audit: sbtc-deposit.clar

**Contract**: `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-deposit`
**Protocol**: sBTC (Bitcoin-backed token on Stacks)
**Network**: Mainnet (deployed)
**Source**: https://raw.githubusercontent.com/boomcrypto/clarity-deployed-contracts/main/contracts/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4/sbtc-deposit.clar
**Auditor**: cocoa007.btc
**Date**: 2026-02-27
**Clarity Version**: Pre-Clarity 4 (no `as-contract?` usage observed; uses `.contract` relative references)
**Confidence**: Medium (read-only audit of deployed mainnet contract; no compilation/testing; multi-contract system with off-chain signer components)

## Priority Score

| Metric | Score | Weight | Weighted |
|--------|-------|--------|----------|
| Financial risk | 3 | 3 | 9 |
| Deployment likelihood | 3 | 2 | 6 |
| Code complexity | 2 | 2 | 4 |
| User exposure | 3 | 1.5 | 4.5 |
| Novelty | 2 | 1.5 | 3 |
| **Total** | | **10** | **26.5 / 30 = 2.65** |

Clarity version penalty: -0.5 → **Final: 2.15** ✅ (above 1.8 threshold)

## Architecture Overview

The sBTC deposit flow:
1. User sends BTC to the sBTC peg address on Bitcoin
2. sBTC signers (off-chain) observe the deposit and call `complete-deposit-wrapper` or `complete-deposits-wrapper` on Stacks
3. The contract validates the deposit, mints sBTC to the recipient, and records it in `sbtc-registry`

**Trust model**: The signer set (multisig) is fully trusted to submit correct deposit data. The contract validates:
- Caller is current signer principal
- Amount ≥ dust limit (546 sats)
- TXID length = 32 bytes
- No replay (deposit not already completed)
- Burn header hash matches (Bitcoin fork protection)

**Key contracts**:
- `sbtc-deposit` — validates and mints (this audit)
- `sbtc-registry` — stores state, manages protocol roles, key rotation
- `sbtc-token` — SIP-010 fungible token with protocol-gated mint/burn

## Documented Limitations / Trust Assumptions

- Signers are trusted to provide correct `amount`, `recipient`, `burn-hash`, `burn-height`, `sweep-txid`
- No on-chain verification of actual Bitcoin transaction content (SPV proof) — relies entirely on signer honesty
- The contract cannot enforce that signers actually process all valid deposits

## Findings Summary

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 1 |
| Medium | 2 |
| Low | 2 |
| Informational | 3 |

---

### H-01: No Guard Against maxFee ≥ Amount — Deposits Become Unprocessable

**Location**: `complete-deposit-wrapper` — missing validation
**Description**: The contract accepts any `amount` ≥ 546 (dust limit) but has no concept of fees or minimum net amount. In the sBTC protocol, Bitcoin deposits include a `max_fee` parameter set by the user. If `max_fee >= amount`, the sBTC signers will skip the deposit because there would be zero or negative sBTC to mint after fees. However, nothing in the contract or the Bitcoin-side deposit script prevents a user from setting `max_fee >= amount`.

This results in **permanently stuck BTC**: the Bitcoin is sent to the peg address, but signers will never call `complete-deposit-wrapper` for it because the economics don't work. The BTC cannot be recovered.

**Impact**: Users lose funds. We personally experienced this: 5,000 sats stuck in a deposit where maxFee ≥ amount. The signers simply never process it. There is no reclaim mechanism.

**Recommendation**:
1. The deposit request script on Bitcoin should enforce `max_fee < amount - dust_limit` at the script level
2. Alternatively, signers should process these deposits by minting `amount - max_fee` (even if tiny) rather than silently skipping them
3. A reclaim/refund mechanism should exist for deposits that signers cannot or will not process
4. At minimum, front-end UIs should validate `max_fee < amount` before broadcasting

**Note**: This is partially an off-chain/signer issue, but the lack of any on-chain reclaim path makes it a High severity finding for the protocol as a whole.

---

### M-01: Batch Deposit Failure Aborts All Subsequent Deposits

**Location**: `complete-individual-deposits-helper`
**Description**: The `complete-deposits-wrapper` uses `fold` with `complete-individual-deposits-helper`. If any individual deposit fails (e.g., replay, bad burn hash), the error is propagated and **all subsequent deposits in the batch are skipped**. The fold returns the error and stops processing.

```clarity
(match helper-response
  index
    (begin
      (unwrap!
        (complete-deposit-wrapper ...)
      (err (+ ERR_DEPOSIT_INDEX_PREFIX (+ u10 index))))
      (ok (+ index u1))
    )
  err-response
    (err err-response)
)
```

The `err-response` match arm short-circuits — once one deposit fails, all remaining deposits in the list are never processed.

**Impact**: A single invalid deposit in a batch of 500 blocks all deposits after it. Signers must carefully order/filter deposits, or retry with the failing deposit removed. This is a DoS vector on deposit throughput.

**Recommendation**: Use a "skip on error" pattern instead of aborting. Return an `(ok)` with an error log for failed individual deposits, allowing the fold to continue processing remaining deposits.

---

### M-02: No Maximum Amount Cap on Deposits

**Location**: `complete-deposit-wrapper` — amount validation
**Description**: The contract only enforces `amount >= dust-limit` (546 sats) with no upper bound. A compromised or buggy signer could mint an arbitrary amount of sBTC in a single transaction.

**Impact**: If the signer key is compromised, an attacker can mint unlimited sBTC. While this is inherent to the trust model (signers are trusted), a sanity cap (e.g., no single deposit > 21M BTC worth of sats) would limit blast radius.

**Recommendation**: Add a reasonable per-deposit cap as a safety rail, even if set very high (e.g., 2,100,000,000,000,000 sats = 21M BTC). This limits damage from key compromise or bugs.

---

### L-01: Error Code Collision in Batch Deposits

**Location**: `complete-individual-deposits-helper`
**Description**: The error encoding is `ERR_DEPOSIT_INDEX_PREFIX + 10 + index`. `ERR_DEPOSIT_INDEX_PREFIX` is `u303`. So errors are `u313`, `u314`, etc. For large batches (index > ~696), these could theoretically collide with other protocol error codes, though in practice the list is capped at 500 items (max error = u813).

**Impact**: Low — error codes could be confusing but functional impact is minimal.

**Recommendation**: Use a dedicated error range (e.g., `u10000 + index`) to avoid any possible collision.

---

### L-02: No Event Emission in Deposit Contract

**Location**: `complete-deposit-wrapper`
**Description**: The deposit contract itself emits no `print` events. Events are only emitted by `sbtc-registry.complete-deposit`. If the registry call fails after minting, the mint event exists but no deposit-completion event is emitted, making it harder to track partial failures.

**Impact**: Low — observability gap. The registry does emit events, but a failed registry call after successful mint would leave an inconsistent state with no deposit event.

**Recommendation**: Emit a print event in the deposit contract itself for better observability.

---

### I-01: Pre-Clarity 4 — Uses Legacy Contract Call Patterns

**Description**: The contract uses `.sbtc-registry` and `.sbtc-token` relative references and does not use Clarity 4's `as-contract?` with explicit asset allowances. While this contract doesn't use `as-contract` at all (minting is done via `contract-call?` to the token), future upgrades should adopt Clarity 4 patterns.

**Recommendation**: When upgrading, migrate to Clarity 4 for `as-contract?` with explicit asset allowances and `contract-hash?` for verifying contract integrity.

---

### I-02: Signer Principal is Single Point of Trust

**Description**: The `current-signer-principal` is a single address checked via `is-eq tx-sender`. This represents the multisig output of the signer set, but on-chain it appears as a single principal. Key rotation is handled by `sbtc-registry.rotate-keys` (governance-only).

**Impact**: Standard for the sBTC design. The signer set is the root of trust. If compromised, unlimited minting is possible.

**Recommendation**: Informational — this is the intended design. Consider time-locked minting caps as defense-in-depth.

---

### I-03: No Recipient Validation

**Description**: The `recipient` parameter is an unchecked `principal`. Signers could (accidentally or maliciously) mint sBTC to any address, including the contract itself or burn addresses.

**Impact**: Low likelihood given trusted signers, but no guardrails exist.

**Recommendation**: Consider basic validation (e.g., recipient is a standard principal, not a contract principal) if the protocol intends deposits only for user wallets.

---

## Cross-Contract Analysis

### sbtc-registry
- Role-based access control via `active-protocol-contracts` map — well-designed
- `is-protocol-caller` performs bidirectional check (flag→contract AND contract→flag) — good
- `map-insert` for deposit-status means a deposit can only be recorded once — correct replay protection
- `rotate-keys` has `aggregate-pubkeys` replay protection — good
- **Note**: `update-protocol-contract` uses `map-set` not `map-insert` for roles, meaning old contract→role mapping persists. This could leave stale role entries, though they'd point to deactivated contracts.

### sbtc-token
- Protocol functions gated by `sbtc-registry.is-protocol-caller` — correct
- `protocol-mint` has no supply cap — by design (sBTC supply = pegged BTC)
- SIP-010 `transfer` correctly checks `tx-sender` or `contract-caller` = sender
- `get-balance` includes locked tokens — documented behavior, not a bug

## Overall Assessment

The sbtc-deposit contract is **well-structured and minimal** — it does one thing (validate and mint deposits) with appropriate access control. The main risks are:

1. **Off-chain/protocol-level**: The maxFee ≥ amount issue (H-01) is the most impactful real-world problem, as it leads to permanent fund loss with no reclaim path
2. **Batch processing**: The all-or-nothing batch semantics (M-01) could cause operational issues
3. **Trust concentration**: The entire system trusts the signer set — standard for sBTC's design but worth noting

The contract is lean (~90 lines of logic) which limits attack surface. No reentrancy risks (Clarity prevents this by design). No `as-contract` usage means no asset authority concerns in this specific contract.

**Rating**: The contract itself is solid for its scope. The critical gap is at the protocol level — the lack of protection against unprocessable deposits and the absence of a reclaim mechanism.
