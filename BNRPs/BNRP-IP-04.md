# BNRP-IP-04: Reverse Resolution & Anti-Spoofing

| Field | Value |
|-------|-------|
| **BNRP-IP** | 04 |
| **Title** | Reverse Resolution & Anti-Spoofing |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | BNRP-IP-01, BNRP-IP-02, BNRP-IP-03 |
| **Replaces** | BtcName `"op": "reverse"` inscription (extends, does not break) |

---

## Abstract

This proposal defines the reverse resolution process for BNRP — how a Bitcoin wallet address maps back to a `.btc` primary name — and the mandatory anti-spoofing verification that prevents an attacker from falsely claiming any name as their reverse record.

---

## Motivation

Without anti-spoofing guarantees, a malicious actor could inscribe a reverse record claiming their wallet is `satoshi.btc`, even though they don't own that name. Wallets displaying this name would mislead users. BNRP mandates a bidirectional verification step: reverse resolution only succeeds when the forward resolution of the claimed name confirms the wallet address.

---

## Specification

### 1. Reverse Resolution Definition

Reverse resolution is the process of mapping a Bitcoin wallet address to a `.btc` primary name for display purposes. The result is called the **primary name** of the address.

### 2. Resolution Sources

BNRP supports two input paths for reverse resolution:

**Path 1 — Primary Name Table (Solution B):** The indexer maintains a `primary_name` table derived from self-transfer events (BNRP-IP-03 Section 2). This is the default and authoritative source.

**Path 2 — Explicit Reverse Record (Legacy):** An inscription with `"op": "reverse"` explicitly declares the reverse mapping. This is supported for backward compatibility with existing BtcName inscriptions but is secondary to Path 1.

### 3. Mandatory Anti-Spoofing Verification

Regardless of how the candidate primary name is found (Path 1 or Path 2), the following verification MUST be performed before returning the name:

```
GIVEN:  address A claims primary name N
VERIFY: forward_resolve(N) includes address A

IF forward_resolve(N).btc_taproot == A
   OR forward_resolve(N).btc_segwit == A
   OR forward_resolve(N).btc_legacy == A
   OR forward_resolve(N).btc_p2sh == A:
   → return N (verified)
ELSE:
   → return null (anti-spoofing rejection)
```

This verification MUST be performed at resolution time. Cached results older than 10 minutes MUST be re-verified.

### 4. Full Reverse Resolution Algorithm

```pseudocode
function reverseResolve(address: string) -> ReverseResult | null:

  // Step 1: Normalize address
  addr = normalizeAddress(address)
  // For BTC: lowercase bech32 canonical form
  // For EVM addresses: EIP-55 checksum form

  // Step 2: Check primary_name table (Solution B / Solution A)
  candidate = db.getPrimaryName(addr)
  // candidate = { name, mechanism, block_height, ... } or null

  // Step 3: If no primary name, check for explicit reverse inscription
  if candidate == null:
    candidate = db.getExplicitReverseRecord(addr)
    // Returns name from most recent valid "op":"reverse" inscription
    // where reverse_address == addr and inscription owner == addr

  // Step 4: If still null, return null
  if candidate == null:
    return null

  // Step 5: Anti-spoofing — forward resolve the candidate name
  forwardRecord = resolve(candidate.name)
  if forwardRecord == null:
    return null  // Name has no routing record

  // Step 6: Check if addr appears in forward record's address set
  btcAddresses = [
    forwardRecord.btc_taproot,
    forwardRecord.btc_segwit,
    forwardRecord.btc_legacy,
    forwardRecord.btc_p2sh
  ].filter(notNull)

  if addr in btcAddresses:
    return ReverseResult {
      name:     candidate.name,
      verified: true,
      mechanism: candidate.mechanism
    }
  else:
    // Anti-spoofing rejection
    log("Reverse spoofing attempt: " + addr + " claims " + candidate.name)
    return null
```

### 5. Explicit Reverse Record Format (Legacy BtcName)

The explicit reverse record inscription format is inherited from BtcName:

```json
{
  "p": "bnrp",
  "op": "reverse",
  "reverse_address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "name": "alice.btc"
}
```

Validity requirements:
1. The inscription MUST be owned by the address in `reverse_address` at time of resolution.
2. `reverse_address` MUST be the owner of the `alice.btc` name inscription.
3. The anti-spoofing check (Section 3) MUST pass.
4. At least 3 confirmations required.

Legacy `"p": "btcname", "op": "reverse"` inscriptions are treated identically.

### 6. Explicit Reverse Record JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BNRPReverseRecord",
  "type": "object",
  "required": ["p", "op", "reverse_address", "name"],
  "properties": {
    "p":               { "type": "string", "enum": ["bnrp", "btcname"] },
    "op":              { "type": "string", "const": "reverse" },
    "reverse_address": { "type": "string" },
    "name":            { "type": "string", "maxLength": 128 }
  },
  "additionalProperties": false
}
```

### 7. Response Format

A successful reverse resolution returns:

```json
{
  "address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "name": "alice.btc",
  "verified": true,
  "mechanism": "solution_b",
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

A failed or spoofed reverse lookup returns:

```json
{
  "address": "bc1p...",
  "name": null,
  "verified": false
}
```

### 8. Wallet UX Requirements

Wallets implementing BNRP reverse resolution MUST adhere to the following display rules:

1. **Only display a primary name when `verified: true`.** A name with `verified: false` MUST NOT be shown.
2. **Show a verification indicator.** Display a visual "verified" badge (checkmark or similar) alongside the name.
3. **Show full address on hover/tap.** The raw Bitcoin address MUST always be accessible — never hidden entirely behind a name.
4. **Cache duration.** Cache primary names for a maximum of 10 minutes. Re-verify on send/receive actions.
5. **Handle null gracefully.** If no primary name exists, show the truncated address (e.g., `bc1p4x9k...ef3u`).
6. **Unicode names.** For non-ASCII primary names, show the Punycode form in a secondary line.

### 9. Cross-Chain Reverse Resolution (Future Extension)

This version of BNRP-IP-04 covers Bitcoin address reverse resolution only. Resolving an Ethereum or Solana address to a `.btc` primary name is reserved for a future BNRP-IP-13.

The pattern would be: check if the ETH address appears in any `.btc` routing record's `eth` field, and if so, return that name (subject to the same anti-spoofing check).

---

## Examples

### Valid reverse resolution

Wallet `bc1p4x9kv...` self-transferred `alice.btc` at block 850000. Forward record for `alice.btc` has `btc_taproot: "bc1p4x9kv..."`.

```
reverseResolve("bc1p4x9kv...ef3u")
→ { name: "alice.btc", verified: true, mechanism: "solution_b" }
```

### Spoofing attempt rejected

Wallet `bc1pmalicious...` inscribes a reverse record claiming `satoshi.btc`, but `satoshi.btc` forward resolves to `bc1p000satoshi...`.

```
reverseResolve("bc1pmalicious...")
→ null (anti-spoofing rejection: forward address mismatch)
```

### Name transferred away

Wallet A had primary name `alice.btc` via Solution B. Alice transferred `alice.btc` to Wallet B.

```
reverseResolve("bc1p_wallet_A")
→ null (name no longer owned by this wallet)
```

---

## Backward Compatibility

All existing BtcName `"op": "reverse"` inscriptions are valid BNRP reverse records. The anti-spoofing rule described here is consistent with BtcName's documented requirement that `reverse_address` must be the owner of the `.btc` name. BNRP makes this rule mandatory and enforceable at query time.

---

## Security Considerations

**Stale cache spoofing:** If a resolver caches primary names for too long, a name that has been transferred away could still appear as the primary name. The 10-minute TTL limit mitigates this.

**Reorg attacks:** A chain reorganization could briefly invalidate a name transfer and restore a previous state. Resolvers SHOULD require 3 confirmations before considering any state change final, and SHOULD monitor for reorgs.

**Resolver compromise:** A compromised resolver could return false `verified: true` results. Wallets SHOULD support multiple resolver endpoints and cross-check when performing high-value operations (e.g., send). See BNRP-IP-06 for multi-resolver patterns.

---

## Test Vectors

| Scenario | reverseResolve result |
|----------|-----------------------|
| Solution B self-transfer + forward record matches | `{ name: "alice.btc", verified: true }` |
| Solution B self-transfer + no routing record | `null` |
| Solution B self-transfer + forward record has different address | `null` |
| Explicit reverse inscription + forward matches | `{ name: "alice.btc", verified: true }` |
| Explicit reverse inscription + forward doesn't match | `null` |
| No primary name, no reverse inscription | `null` |

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
