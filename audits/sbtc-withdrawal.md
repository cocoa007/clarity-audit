# Security Audit: sbtc-withdrawal.clar

**Contract**: `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-withdrawal`
**Protocol**: sBTC (Bitcoin-backed token on Stacks)
**Network**: Mainnet (deployed)
**Source**: https://api.hiro.so/v2/contracts/source/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4/sbtc-withdrawal
**Auditor**: cocoa007.btc
**Date**: 2026-03-03
**Clarity Version**: Pre-Clarity 4 (uses `.contract` relative references, no `as-contract?`)
**Confidence**: Medium (read-only audit of deployed mainnet contract; multi-contract system with off-chain signer components)

## Priority Score

| Metric | Score | Weight | Weighted |
|--------|-------|--------|----------|
| Financial risk | 3 | 3 | 9 |
| Deployment likelihood | 3 | 2 | 6 |
| Code complexity | 2 | 2 | 4 |
| User exposure | 3 | 1.5 | 4.5 |
| Novelty | 1 | 1.5 | 1.5 |
| **Total** | | **10** | **25.0 / 30 = 2.50** |

Clarity version penalty: -0.5 → **Final: 2.00** ✅ (above 1.8 threshold)

## Architecture Overview

The sBTC withdrawal flow:
1. User calls `initiate-withdrawal-request` — specifying amount, BTC recipient address, and max-fee
2. The contract locks `amount + max-fee` of the user's sBTC via `protocol-lock`
3. Off-chain signers observe the request and send BTC on Bitcoin, then call `accept-withdrawal-request` or `reject-withdrawal-request`
4. On accept: locked sBTC is burned, any unused fee (max-fee minus actual fee) is minted back to user
5. On reject: locked sBTC is unlocked back to user
6. Batch processing via `complete-withdrawals` handles up to 600 requests per tx

**Trust model**: Identical to sbtc-deposit — the signer set is fully trusted. The contract validates:
- Caller is current signer principal (for accept/reject)
- Request exists and is not already processed
- Actual fee ≤ max-fee
- Burn header hash matches (Bitcoin fork protection)
- Recipient address is well-formed (version + hashbytes validation)

**Key contracts**:
- `sbtc-withdrawal` — validates, locks/burns/unlocks sBTC (this audit)
- `sbtc-registry` — stores withdrawal requests, manages signer data
- `sbtc-token` — SIP-010 with protocol-gated lock/unlock/mint/burn

## Documented Limitations / Trust Assumptions

- Signers are trusted to provide correct bitcoin-txid, output-index, fee, burn-hash, sweep-txid
- No on-chain SPV proof of the Bitcoin withdrawal transaction
- Signers can reject any withdrawal without on-chain justification
- User must trust signers will eventually process their withdrawal

## Findings Summary

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 2 |
| Low | 3 |
| Informational | 3 |

---

### M-01: Batch Withdrawal Failure Aborts All Subsequent Withdrawals

**Location**: `complete-individual-withdrawal-helper`
**Description**: Identical to the deposit contract pattern (see sbtc-deposit M-01). The `complete-withdrawals` function uses `fold` with a helper that propagates errors. If any individual withdrawal in the batch fails, all subsequent withdrawals are skipped.

```clarity
(match helper-response
  index
    (let (...)
      (if (get status withdrawal)
        ;; accepted
        (begin
          (asserts! (and (is-some current-bitcoin-txid) ...) (err (+ ERR_WITHDRAWAL_INDEX_PREFIX (+ u10 index))))
          (unwrap! (accept-withdrawal-request ...) (err (+ ERR_WITHDRAWAL_INDEX_PREFIX (+ u10 index))))
        )
        ;; rejected
        (unwrap! (reject-withdrawal-request ...) (err (+ ERR_WITHDRAWAL_INDEX_PREFIX (+ u10 index))))
      )
      (ok (+ index u1))
    )
  err-response (err err-response)
)
```

**Impact**: A single bad request in a batch of 600 blocks processing of all subsequent requests. Users waiting for withdrawal completion experience delays. Signers must carefully filter or retry with the failing request removed.

**Recommendation**: Use a skip-on-error pattern that logs failed individual withdrawals but continues processing the remaining list.

---

### M-02: Dust Limit Check After Token Lock — User Loses Gas on Revert

**Location**: `initiate-withdrawal-request`
**Description**: The function calls `protocol-lock` (which transfers/locks the user's sBTC) *before* checking the dust limit:

```clarity
(begin
  (try! (contract-call? .sbtc-token protocol-lock (+ amount max-fee) tx-sender withdraw-role))
  (asserts! (> amount DUST_LIMIT) ERR_DUST_LIMIT)
  (try! (validate-recipient recipient))
  ...
)
```

If `amount <= DUST_LIMIT`, the lock succeeds (spending gas and modifying state) but then the `asserts!` fails. Because Clarity reverts all state changes on error, the lock is rolled back — but the user still pays the transaction fee.

**Impact**: Users who accidentally submit sub-dust withdrawals pay STX gas for a guaranteed-to-fail transaction. The ordering is suboptimal — cheap validation should precede expensive state changes.

**Recommendation**: Move the dust limit and recipient validation checks before the `protocol-lock` call:

```clarity
(begin
  (asserts! (> amount DUST_LIMIT) ERR_DUST_LIMIT)
  (try! (validate-recipient recipient))
  (try! (contract-call? .sbtc-token protocol-lock (+ amount max-fee) tx-sender withdraw-role))
  (ok (try! (contract-call? .sbtc-registry create-withdrawal-request ...)))
)
```

---

### L-01: No Maximum Amount Cap on Withdrawals

**Location**: `initiate-withdrawal-request`
**Description**: There is no upper bound on withdrawal amount. While the user is limited by their sBTC balance (the `protocol-lock` will fail if insufficient), a compromised signer could accept fabricated withdrawal requests or a bug could allow unexpectedly large withdrawals.

**Impact**: Low — user can only withdraw what they hold, and acceptance requires signer authorization. But a per-withdrawal sanity cap would limit blast radius of bugs.

**Recommendation**: Consider a configurable per-withdrawal cap as defense-in-depth.

---

### L-02: Error Code Collision Risk in Batch Withdrawals

**Location**: `complete-individual-withdrawal-helper`
**Description**: Error encoding uses `ERR_WITHDRAWAL_INDEX_PREFIX + 10 + index`. `ERR_WITHDRAWAL_INDEX_PREFIX` is `u507`. Errors range from `u517` to `u6107` (for 600 items). While unlikely to collide with protocol error codes (u500-u508), the encoding scheme is fragile.

**Impact**: Low — error decoding confusion, no functional impact.

**Recommendation**: Use a dedicated high-range prefix (e.g., `u10000 + index`) to guarantee no collisions.

---

### L-03: Dust Limit Check Uses Strict Greater-Than

**Location**: `initiate-withdrawal-request`
**Description**: The check is `(> amount DUST_LIMIT)` where `DUST_LIMIT = u546`. This means exactly 546 sats is rejected — only 547+ sats is accepted. The Bitcoin dust limit is typically 546 sats (inclusive).

```clarity
(asserts! (> amount DUST_LIMIT) ERR_DUST_LIMIT)
```

**Impact**: Users cannot withdraw exactly 546 sats. Minor UX inconsistency.

**Recommendation**: Use `(>= amount DUST_LIMIT)` for consistency with Bitcoin's dust limit semantics, or document that the minimum withdrawal is 547 sats.

---

### I-01: Pre-Clarity 4 — No Explicit Asset Allowances

**Description**: The contract does not use Clarity 4's `as-contract?` with explicit asset allowances. While this contract doesn't use `as-contract` at all (protocol operations go through `contract-call?` to sbtc-token), migrating to Clarity 4 would enable `contract-hash?` verification for deployment integrity.

**Recommendation**: Migrate to Clarity 4 on next upgrade for `contract-hash?` and future `as-contract?` benefits.

---

### I-02: Signer Can Reject Withdrawals Without Justification

**Description**: The `reject-withdrawal-request` function takes only `request-id` and `signer-bitmap` — no reason code or justification is required. While the user gets their sBTC back, they have no on-chain visibility into why the withdrawal was rejected.

**Impact**: Informational — signers could indefinitely reject withdrawals as a soft censorship mechanism. Users have no recourse other than retrying.

**Recommendation**: Consider adding an optional reason code to rejection events for transparency.

---

### I-03: accept-withdrawal-request Performs Double Caller Check in Batch Context

**Location**: `accept-withdrawal-request` and `complete-withdrawals`
**Description**: When called via `complete-withdrawals`, the signer check happens twice — once in `complete-withdrawals` and again inside `accept-withdrawal-request` (or `reject-withdrawal-request`). This is redundant but not harmful.

```clarity
;; In complete-withdrawals:
(asserts! (is-eq (get current-signer-principal current-signer-data) tx-sender) ERR_INVALID_CALLER)
;; Then in accept-withdrawal-request (called by the helper):
(asserts! (is-eq (get current-signer-principal current-signer-data) tx-sender) ERR_INVALID_CALLER)
```

**Impact**: Wasted gas on redundant reads, but provides defense-in-depth for direct calls to accept/reject.

**Recommendation**: Informational only — the redundancy is acceptable for safety.

---

## Cross-Contract Analysis

### Comparison with sbtc-deposit

| Aspect | sbtc-deposit | sbtc-withdrawal |
|--------|-------------|-----------------|
| User-initiated | No (signers call) | Yes (user calls initiate) |
| Token operation | Mint | Lock → Burn (accept) or Unlock (reject) |
| Batch size | 500 | 600 |
| Dust limit | ≥ 546 (>=) | > 546 (>) — inconsistent |
| Burn hash check | ✅ | ✅ |
| Batch error handling | Same fold pattern | Same fold pattern |
| Fee handling | N/A (no user fee) | max-fee with refund of unused portion |

**Notable**: The dust limit check is inconsistent between contracts — deposit uses `>=` while withdrawal uses `>`. This should be harmonized.

### sbtc-token Integration

- `protocol-lock`: Locks user's sBTC, gated by `sbtc-registry.is-protocol-caller` — correct
- `protocol-burn-locked`: Burns locked sBTC on accept — correct
- `protocol-unlock`: Returns locked sBTC on reject — correct
- `protocol-mint`: Mints fee refund back to user — correct
- Lock/burn/unlock cycle is sound — no path to double-spend or permanent lock (assuming signers eventually process)

### Reentrancy Analysis

Clarity prevents reentrancy by design — no callbacks or dynamic dispatch within a transaction. The contract-call chain is: withdrawal → sbtc-token (lock/burn/mint) → sbtc-registry (state update). All calls are to known contracts via static `.contract` references. **No reentrancy risk.**

### State Manipulation

- Request status uses `is-none` check — once set, cannot be re-processed (correct)
- `protocol-lock` moves tokens to locked state — user cannot transfer locked tokens
- Burn-hash verification prevents accepting withdrawals against a forked Bitcoin chain

## Overall Assessment

The sbtc-withdrawal contract is **well-designed and mirrors the deposit contract's patterns**. The withdrawal flow (lock → burn/unlock) is sound. Key observations:

1. **No critical vulnerabilities** — access control is correct, arithmetic is safe, state transitions are properly guarded
2. **Validation ordering** (M-02) is the most actionable fix — cheap checks should precede expensive state operations
3. **Batch error handling** (M-01) is a known pattern shared with sbtc-deposit — operational risk, not a security vulnerability
4. **Dust limit inconsistency** between deposit (>=) and withdrawal (>) is minor but should be harmonized
5. **No reentrancy risk** — Clarity's design eliminates this class of vulnerabilities
6. **Signer trust** is the dominant risk — same as all sBTC contracts

**Rating**: Solid contract for its scope. The withdrawal flow correctly handles the lock-burn-refund lifecycle. The findings are medium/low severity with no paths to fund loss (assuming honest signers).
