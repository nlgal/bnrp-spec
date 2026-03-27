# BNRP-IP-02: Routing Inscription Format

| Field | Value |
|-------|-------|
| **BNRP-IP** | 02 |
| **Title** | Routing Inscription Format |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | BNRP-IP-01 |
| **Replaces** | BtcName `"p": "btcname", "op": "routing"` (extends, does not break) |

---

## Abstract

This proposal defines the canonical inscription format for BNRP routing records — the on-chain data that maps a `.btc` name to blockchain addresses, avatar, web content, and metaverse pointers. BNRP routing inscriptions use `"p": "bnrp"` and are a strict superset of the existing BtcName `"p": "btcname"` routing format.

---

## Motivation

BtcName's existing routing format supports BTC, ETH, SOL, and MATIC addresses plus an avatar field. It does not support EVM Layer 2 chains (Base, Arbitrum), a standardized avatar URI scheme, web content pointers, bitmap records, or a coinType-based extensibility model for future chains. BNRP-IP-02 defines a backward-compatible extension that fills these gaps while preserving all existing BtcName inscriptions.

---

## Specification

### 1. Protocol Identifier

BNRP routing inscriptions MUST use `"p": "bnrp"`. Existing `"p": "btcname"` routing inscriptions are valid and MUST be indexed by all BNRP-compliant resolvers (see Section 8 — Backward Compatibility).

### 2. Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `p` | string | MUST be `"bnrp"` |
| `op` | string | MUST be `"routing"` |
| `name` | string | The normalized `.btc` name (e.g., `"alice.btc"`) |

### 3. Bitcoin Address Fields

At least one Bitcoin address field SHOULD be present. All are optional individually.

| Field | Address Type | Format |
|-------|-------------|--------|
| `btc_taproot` | Taproot (P2TR) | `bc1p...` (bech32m) |
| `btc_segwit` | Native SegWit (P2WPKH/P2WSH) | `bc1q...` (bech32) |
| `btc_p2sh` | P2SH | `3...` (base58check) |
| `btc_legacy` | Legacy P2PKH | `1...` (base58check) |
| `btc_lightning` | Lightning Network | BOLT11 invoice prefix or LNURL |

The `btc_taproot` address is the **canonical Bitcoin address** for BNRP. When a resolver returns a BTC address without a specified type, it MUST return `btc_taproot` if present, falling back to `btc_segwit`, `btc_p2sh`, `btc_legacy` in that order.

### 4. Cross-Chain Address Fields

Cross-chain addresses MUST use the SLIP-44 coin type as the field naming basis. For well-known chains, BNRP defines short field aliases:

| Field | SLIP-44 coinType | Chain | Address Format |
|-------|-----------------|-------|----------------|
| `eth` | 60 | Ethereum Mainnet | `0x...` (EIP-55 checksum) |
| `sol` | 501 | Solana | Base58 |
| `matic` | 2147483785 | Polygon | `0x...` |
| `arb` | 2189526273 | Arbitrum One | `0x...` |
| `base` | 2155872277 | Base | `0x...` |
| `chain_{coinType}` | any | Future/other chains | Per-chain format |

The generic pattern `chain_{coinType}` (e.g., `chain_2147492648` for Optimism) allows any future chain to be added without a spec update.

EVM chain coin types follow ENSIP-11: `coinType = chainId | 0x80000000`.

| Chain | Chain ID | BNRP coinType |
|-------|---------|--------------|
| Ethereum | 1 | 60 (legacy SLIP-44) |
| Polygon | 137 | 2147483785 |
| Arbitrum One | 42161 | 2189526273 |
| Base | 8453 | 2155872277 |
| Optimism | 10 | 2147483658 |
| BNB Chain | 56 | 2147483704 |

### 5. Profile / Metadata Fields

These fields are OPTIONAL in a routing inscription. A separate profile inscription (see BNRP-IP-07) may also carry these.

| Field | Type | Description |
|-------|------|-------------|
| `display` | string | Display name (max 64 chars) |
| `description` | string | Bio/description (max 256 chars) |
| `avatar` | string | Avatar URI (see BNRP-IP-05) |
| `url` | string | Website URL |
| `email` | string | Contact email |
| `com.twitter` | string | Twitter/X handle (without @) |
| `com.github` | string | GitHub handle |

### 6. Content Field

| Field | Type | Description |
|-------|------|-------------|
| `content` | string | Web/content URI: `ipfs://`, `ipns://`, `ar://`, `https://`, or `ord://` |

### 7. Bitmap / Metaverse Fields

| Field | Type | Description |
|-------|------|-------------|
| `bitmap_district` | string | Bitcoin block number (e.g., `"840000"`) |
| `bitmap_parcel` | string | Transaction index within block |
| `bitmap_world` | string | Metaverse world identifier |
| `bitmap_coordinates` | string | Coordinates in format `"x:{n},y:{n}"` |

### 8. Validity Rules

A routing inscription is **valid** if and only if:

1. `p` is `"bnrp"` (or `"btcname"` for legacy compatibility)
2. `op` is `"routing"`
3. `name` normalizes successfully per BNRP-IP-01
4. The inscription is owned by the same address that owns the `.btc` name inscription at the time of resolution
5. The inscription has at least 3 confirmations
6. For each address field present, the address is validly formatted for its chain

If condition 4 is not met, the routing inscription MUST be ignored. The name still resolves (forward to the owner's taproot address), but the custom routing record is not applied.

### 9. Ownership Verification

The indexer MUST verify that the wallet address that currently holds the routing inscription's sat is the same address that currently holds the name inscription's sat. Both must be owned by the same address for the routing record to be considered valid.

### 10. Last-Is-New Rule

If multiple valid routing inscriptions exist for the same name, the one with the highest block height (most recently confirmed) MUST be used. Within the same block, the one with the lower transaction index wins.

---

## Full JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BNRPRoutingRecord",
  "type": "object",
  "required": ["p", "op", "name"],
  "properties": {
    "p":               { "type": "string", "const": "bnrp" },
    "op":              { "type": "string", "const": "routing" },
    "name":            { "type": "string", "maxLength": 128 },
    "btc_taproot":     { "type": "string", "pattern": "^bc1p[a-z0-9]{39,87}$" },
    "btc_segwit":      { "type": "string", "pattern": "^bc1q[a-z0-9]{38,}$" },
    "btc_p2sh":        { "type": "string", "pattern": "^3[a-zA-Z0-9]{25,34}$" },
    "btc_legacy":      { "type": "string", "pattern": "^1[a-zA-Z0-9]{25,34}$" },
    "btc_lightning":   { "type": "string" },
    "eth":             { "type": "string", "pattern": "^0x[0-9a-fA-F]{40}$" },
    "sol":             { "type": "string", "minLength": 32, "maxLength": 44 },
    "matic":           { "type": "string", "pattern": "^0x[0-9a-fA-F]{40}$" },
    "arb":             { "type": "string", "pattern": "^0x[0-9a-fA-F]{40}$" },
    "base":            { "type": "string", "pattern": "^0x[0-9a-fA-F]{40}$" },
    "display":         { "type": "string", "maxLength": 64 },
    "description":     { "type": "string", "maxLength": 256 },
    "avatar":          { "type": "string" },
    "url":             { "type": "string", "format": "uri" },
    "email":           { "type": "string", "format": "email" },
    "com.twitter":     { "type": "string", "maxLength": 15 },
    "com.github":      { "type": "string", "maxLength": 39 },
    "content":         { "type": "string" },
    "bitmap_district": { "type": "string", "pattern": "^[0-9]+$" },
    "bitmap_parcel":   { "type": "string", "pattern": "^[0-9]+$" },
    "bitmap_world":    { "type": "string", "maxLength": 64 },
    "bitmap_coordinates": { "type": "string", "pattern": "^x:-?[0-9]+,y:-?[0-9]+$" }
  },
  "additionalProperties": true
}
```

`additionalProperties: true` allows future extension fields without requiring a spec update for each new chain.

---

## Complete Example: alice.btc

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
  "btc_lightning": "lnurlp1qqpsqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqxkx9h2",
  "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "base": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "arb": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "sol": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "display": "Alice",
  "description": "Bitcoin artist and builder.",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "url": "https://alice.xyz",
  "com.twitter": "aliceonbtc",
  "com.github": "alice-btc",
  "content": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "bitmap_district": "840000"
}
```

---

## Backward Compatibility

All existing `"p": "btcname", "op": "routing"` inscriptions remain fully valid. BNRP resolvers MUST index them using the following field mapping:

| BtcName field | BNRP equivalent |
|---------------|----------------|
| `btc_p2phk` | `btc_legacy` |
| `btc_p2sh` | `btc_p2sh` |
| `btc_segwit` | `btc_segwit` |
| `btc_lightning` | `btc_lightning` |
| `eth_address` | `eth` |
| `matic_address` | `matic` |
| `sol_address` | `sol` |
| `avatar` | `avatar` (treated as `ord:{value}` if no URI scheme prefix present) |

When a name has both a `"p": "btcname"` and a `"p": "bnrp"` routing inscription, the `"p": "bnrp"` inscription MUST take precedence if it is newer (higher block height). If the `"p": "btcname"` inscription is newer, it takes precedence for the fields it defines; BNRP-only fields from an older `"p": "bnrp"` inscription are not carried forward.

---

## Security Considerations

**Owner mismatch:** An inscription inscribed by one party and transferred to a name owner is only valid after the name owner also owns the routing inscription. Resolvers MUST check ownership at resolution time, not at inscription time.

**Field injection:** The `additionalProperties: true` schema allows unknown fields. Resolvers MUST ignore unknown fields rather than erroring. Applications MUST NOT render unknown fields without user confirmation.

**Address format validation:** Resolvers MUST validate address formats. An invalid ETH address in `eth` should cause that field to be silently ignored, not cause resolution to fail entirely.

---

## Test Vectors

| Input | Expected behavior |
|-------|-------------------|
| Valid `"p":"bnrp","op":"routing","name":"alice.btc"` + all fields | Resolves all addresses and metadata |
| `"name":"Alice.BTC"` (mixed case) | Normalized to `alice.btc`, resolves correctly |
| Routing inscription owned by address X, but `alice.btc` name owned by address Y | Routing record ignored; name still resolves to Y's taproot address |
| Two routing inscriptions for same name; one at block 840000, one at 841000 | Block 841000 inscription wins |
| `"eth": "not-an-address"` | `eth` field silently ignored; other fields resolve normally |

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
