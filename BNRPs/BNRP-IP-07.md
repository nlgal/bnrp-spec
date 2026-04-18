# BNRP-IP-07: BtcName Routing Record Compatibility

| Field | Value |
|-------|-------|
| **BNRP-IP** | 7 |
| **Title** | BtcName Routing Record Compatibility |
| **Status** | Draft |
| **Type** | Standards Track |
| **Author** | [@nlgal](https://github.com/nlgal) |
| **Created** | 2026-04-18 |
| **Requires** | BNRP-IP-01 (Routing Record Format) |

---

## Abstract

This document formally specifies the compatibility mapping between BtcName routing inscription records (`"p": "btcname"`) and the BNRP routing record format (`"p": "bnrp"`). All valid BtcName routing records are first-class BNRP records. No re-inscription is required. Resolvers implementing BNRP MUST support both protocol values.

---

## Motivation

BtcName is the dominant `.btc` name registrar with over 2 million registered names and 139,000 holders. These names are registered using BtcName's inscription format, which predates BNRP. Rather than require all existing holders to re-inscribe under the BNRP protocol, this specification defines a normalization layer that allows BNRP resolvers to natively interpret BtcName records.

This alignment:

- Gives BNRP instant compatibility with all existing BtcName holders
- Gives BtcName holders access to BNRP-aware tooling without any action
- Establishes a clear migration path for holders who want to adopt BNRP extended fields
- Supports the ongoing merger/collaboration proposal between BNRP and BtcName

---

## Specification

### 1. Protocol Identifier Equivalence

A BNRP resolver MUST treat the following `"p"` values as equivalent for routing record resolution:

```
"p": "bnrp"     — BNRP native record
"p": "btcname"  — BtcName native record (treated as valid BNRP)
```

Both are subject to the same latest-inscription-wins resolution rule defined in BNRP-IP-01.

### 2. Field Mapping

The following table maps BtcName inscription fields to their BNRP canonical field names. Resolvers MUST apply this mapping when normalizing a `"p": "btcname"` record.

| BtcName Field | BNRP Canonical Field | Notes |
|---|---|---|
| `p` | `p` | Value `"btcname"` → treated as `"bnrp"` |
| `op` | `op` | `"routing"` or `"reverse"` |
| `name` | `name` | Full name including TLD (e.g., `satoshi.btc`) |
| `btc_taproot` | `btc_taproot` | Taproot address (bc1p...) |
| `btc_segwit` | `btc_segwit` | SegWit address (bc1q...) |
| `btc_p2sh` | `btc_p2sh` | P2SH address (3...) |
| `btc_p2phk` | `btc_p2phk` | Legacy address (1...) |
| `btc_lightning` | `btc_lightning` | Lightning Network address |
| `eth_address` | `eth` | Ethereum address |
| `matic_address` | `matic` | Polygon address |
| `sol_address` | `sol` | Solana address |
| `ord_index` | `content` | Inscription ID for content/website routing |
| `ord_handle` | `ord_handle` | Subdomain handle |
| `avatar` | `avatar` | Bare inscription ID (e.g., `abc123i0`) |
| `reverse_address` | `reverse_address` | Reverse resolution target address |

BNRP extended fields not present in BtcName (`url`, `display`, `description`, `email`, `com.twitter`, `com.github`, `bitmap_district`, etc.) MAY be added to a `"p": "btcname"` record for forward compatibility. Resolvers MUST read and expose these fields if present.

### 3. Avatar Field Normalization

BtcName stores avatars as bare inscription IDs (e.g., `"avatar": "abc123i0"`). BNRP resolvers MUST treat a bare inscription ID in the `avatar` field as equivalent to the `ord:` URI scheme defined in BNRP-IP-05:

```
"avatar": "abc123i0"
// treated as:
"avatar": "ord:abc123i0"
```

Avatar ownership verification (BNRP-IP-05, Section 4) applies equally to both formats.

### 4. Content / Website Routing

BtcName uses `ord_index` to route a name to an inscription (text, image, or website). BNRP maps this to the `content` field. Resolvers MUST:

1. Treat a bare inscription ID in `content` as an `ord:` URI
2. Render or redirect to the inscription content via an Ordinals gateway (see BNRP-IP-01, Section 5.2 for gateway fallback order)

### 5. Reverse Resolution

BtcName's `"op": "reverse"` records with a `reverse_address` field are compatible with the BNRP reverse resolution mechanism defined in BNRP-IP-03. Resolvers implementing reverse lookup MUST index both `"op": "reverse"` records regardless of `"p"` value.

### 6. Primary Name

BtcName's Solution B draft for primary name resolution is compatible with BNRP-IP-03 (Primary Name). Resolvers SHOULD treat a BtcName primary name inscription as a valid BNRP primary name record.

### 7. Migration Path

Existing BtcName holders who wish to adopt BNRP extended fields (url, display, twitter, etc.) have two options:

**Option A — Hybrid record:** Re-inscribe with `"p": "btcname"` and add BNRP extended fields. Backward compatible with BtcName tooling.

**Option B — Full BNRP record:** Re-inscribe with `"p": "bnrp"`. Gains full BNRP resolver support. No loss of BtcName registration status (registration is separate from routing records).

Both options follow the latest-inscription-wins rule. The most recent inscription for a given name is the active record regardless of protocol value.

---

## Reference Implementation

The BNRP reference resolver at [bnrp.name](https://bnrp.name) implements this compatibility mapping via the `parseBtcNameRecord()` function in `app.js`. Field normalization is applied at record build time in `buildRecord()`.

Key logic:

```javascript
// Detect BtcName native record
const isBtcName = r.p === 'btcname';

// Normalize eth_address → eth, matic_address → matic, etc.
function parseBtcNameRecord(r) {
  return {
    btc_taproot:   r.btc_taproot   || null,
    btc_segwit:    r.btc_segwit    || null,
    btc_p2sh:      r.btc_p2sh      || null,
    btc_p2phk:     r.btc_p2phk     || null,
    btc_lightning: r.btc_lightning  || null,
    eth:           r.eth_address    || null,
    matic:         r.matic_address  || null,
    sol:           r.sol_address    || null,
    content:       r.ord_index      || null,
    avatar:        r.avatar         || null,
    // ... BNRP extended fields if present
  };
}
```

---

## Security Considerations

All security considerations from BNRP-IP-01 and BNRP-IP-05 apply equally to BtcName records processed under this compatibility mapping. In particular:

- Routing record ownership is validated against the inscription owner address, not the `name` field value
- Avatar ownership verification requires the inscription to be held by the same address as the name registration
- Spoofing mitigation: resolvers MUST verify that the `name` field matches the queried name (case-insensitive, normalized)

---

## Copyright

This BNRP Improvement Proposal is released under CC0 1.0 Universal (Public Domain).
