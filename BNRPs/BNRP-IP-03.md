# BNRP-IP-03: Primary Name Mechanism

| Field | Value |
|-------|-------|
| **BNRP-IP** | 03 |
| **Title** | Primary Name Mechanism |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | BNRP-IP-01, BNRP-IP-02 |
| **Replaces** | BtcName Ordinals Primary Name Protocol Solution A (Draft), Solution B (Draft) |

---

## Abstract

This proposal defines the canonical mechanism for declaring a `.btc` name as the **primary name** of a Bitcoin wallet address. A primary name is the human-readable identity that wallets, explorers, and dApps display in place of a raw Bitcoin address. BNRP adopts BtcName's Solution B as the default primary name mechanism, with Solution A retained as an optional extended declaration. This proposal resolves the ambiguity between the two draft proposals.

---

## Motivation

BtcName published two competing draft proposals for primary name assignment (Solution A and Solution B). The Bitcoin ecosystem needs a single canonical answer so wallets can implement consistently. This proposal evaluates both, selects Solution B as the default, and defines how both coexist.

---

## Specification

### 1. Definitions

- **Primary Name:** The single `.btc` name associated with a wallet address that should be displayed in place of the raw address.
- **Forward Resolution:** `name.btc → address`
- **Reverse Resolution:** `address → primary name`
- A wallet has **at most one** primary name at any given time.

### 2. Default Mechanism: Solution B (Self-Transfer)

**Rule:** When a `.btc` name inscription is transferred from address A to address A (i.e., `from_address == to_address`), that name becomes the primary name of address A.

This is the canonical BNRP primary name mechanism.

#### 2.1 Requirements for a Valid Solution B Primary Name

1. The transfer transaction MUST be confirmed at block height ≥ 840000.
2. The `from_address` and `to_address` of the inscription transfer MUST be identical.
3. The address MUST be Native SegWit (`bc1q...`) or Taproot (`bc1p...`).
4. The name MUST normalize successfully per BNRP-IP-01 and be ≤ 128 characters.
5. The transferring address MUST be the current owner of the name inscription at the time of transfer.

#### 2.2 Last-Is-New

A wallet may change its primary name by performing a new self-transfer with a different name. The most recent valid self-transfer (by block height, then transaction index within block) determines the current primary name.

#### 2.3 Invalidation

A primary name is **immediately invalidated** when the name inscription is transferred to a different address. At that point, the original wallet has no primary name until a new valid self-transfer is performed.

**Example sequence:**
- Block 840100: Wallet A self-transfers `alice.btc` → primary name of A is `alice.btc`
- Block 840200: Wallet A transfers `alice.btc` to Wallet B → primary name of A is null
- Block 840300: Wallet B self-transfers `alice.btc` → primary name of B is `alice.btc`

#### 2.4 Why Solution B is Preferred

| Criterion | Solution A | Solution B |
|-----------|-----------|-----------|
| Implementation complexity | High (minter tracking required) | Low (transfer event only) |
| Proxy inscription risk | High (proxy = minter ≠ owner) | None (self-transfer is inherently self-authenticated) |
| Metadata support | Yes (avatar field) | No (name only) |
| Already deployed | Yes (block height unspecified) | Yes (block 840000) |
| Atomic with name ownership | No (separate inscription) | Yes (same UTXO event) |
| Recommended for wallets | Opt-in extended | Default |

Solution B is simpler, proxy-proof, and its primary name state is a direct function of inscription ownership events — making it trivially derivable by any indexer without additional trust assumptions.

### 3. Extended Mechanism: Solution A (Explicit Declaration)

Solution A remains valid as an **extended primary name declaration** that allows metadata (avatar) to be attached to the primary name record.

**Rule:** When a wallet self-mints (not via proxy service) an inscription with the following format, it declares a primary name:

```json
{
  "p": "bnrp",
  "op": "primary-name",
  "name": "alice.btc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
}
```

#### 3.1 Requirements for a Valid Solution A Primary Name

1. The **minter** of the `primary-name` inscription (the address that spends the tapscript input in the reveal transaction) MUST be the current owner of `name.btc` at the time of resolution.
2. The `name` field MUST normalize successfully per BNRP-IP-01.
3. The inscription MUST have at least 3 confirmations.
4. The inscription MUST NOT have been minted via a proxy inscription service.
5. The avatar field, if present, MUST be a valid BNRP avatar URI per BNRP-IP-05.

#### 3.2 Minter Definition

The minter of an inscription is the address that spends the input containing the tapscript in the reveal transaction. This address is cryptographically fixed at inscription time and cannot be changed. See [BtcName Primary Name Protocol Solution A](https://docs.btcname.id/docs/ordinals-primary-name-protocol/ordinals-primary-name-protocol-solution-a-draft) Appendix for the technical definition.

#### 3.3 Proxy Detection

An inscription is considered minted via proxy if the minter address is different from the address that received the inscription as its first owner. Indexers MUST reject Solution A declarations where `minter != first_owner`.

### 4. Conflict Resolution: Solution A vs Solution B

When both a Solution A and Solution B primary name exist for the same wallet:

1. The one with the **higher block height** wins.
2. Within the same block, Solution B (transfer) takes precedence over Solution A (inscription).

Rationale: Solution B is the default and is always authoritative for name ownership state. Solution A declarations can provide metadata but do not override more recent ownership events.

### 5. JSON Schema: Solution A Primary Name Inscription

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BNRPPrimaryNameRecord",
  "type": "object",
  "required": ["p", "op", "name"],
  "properties": {
    "p":      { "type": "string", "const": "bnrp" },
    "op":     { "type": "string", "const": "primary-name" },
    "name":   { "type": "string", "maxLength": 128 },
    "avatar": { "type": "string" }
  },
  "additionalProperties": false
}
```

Legacy BtcName Solution A inscriptions using `"p": "primary-name"` are also valid and MUST be indexed.

### 6. Indexer Implementation Requirements

Indexers MUST maintain a `primary_name` table with at minimum:

| Column | Type | Description |
|--------|------|-------------|
| `address` | string | Wallet address (normalized) |
| `name` | string | Primary name (normalized) |
| `mechanism` | enum | `solution_a` or `solution_b` |
| `block_height` | integer | Block height of setting event |
| `tx_index` | integer | Transaction index within block |
| `inscription_id` | string | Relevant inscription ID |

On each new block, indexers MUST:
1. Scan for inscription transfer events where `from_address == to_address`
2. For each self-transfer, check if the transferred inscription is a name inscription and if block height ≥ 840000
3. If valid, update the `primary_name` table
4. Scan for new `"op": "primary-name"` inscriptions
5. Validate minter == first_owner
6. If valid, update the `primary_name` table using conflict resolution rules from Section 4

---

## Examples

### Setting a primary name via Solution B

Wallet `bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u` transfers `alice.btc` to itself:

```
from: bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u
to:   bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u
inscription: alice.btc inscription sat
block: 850000
```

Result: `bc1p4x9kv...` has primary name `alice.btc`.

### Setting a primary name via Solution A (with avatar)

```json
{
  "p": "bnrp",
  "op": "primary-name",
  "name": "alice.btc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
}
```

Minted by `bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u`, which owns `alice.btc`. Result: primary name with avatar set.

---

## Backward Compatibility

- BtcName Solution B primary names (block height ≥ 840000 self-transfers) are identical to BNRP Solution B. No re-indexing needed.
- BtcName Solution A inscriptions using `"p": "primary-name"` are valid BNRP Solution A records. BNRP resolvers MUST index them.
- The `"p": "primary_name"` variant (underscore) seen in some BtcName docs is also recognized for backward compatibility.

---

## Security Considerations

**Proxy inscription attack:** Solution A is only valid when the minter and first owner match. Indexers MUST enforce this. A wallet that used a proxy inscription service to mint its primary-name inscription will not have a valid Solution A primary name.

**Name transfer invalidation:** Wallets that sell or transfer their `.btc` name immediately lose their primary name. Applications MUST re-verify primary names before displaying them (not cache indefinitely).

**Stale primary name display:** Wallets SHOULD cache primary names for no more than 10 minutes. See BNRP-IP-04 for the full verification flow.

---

## Test Vectors

| Scenario | Expected Primary Name |
|----------|-----------------------|
| Wallet A self-transfers `alice.btc` at block 850000 | `alice.btc` |
| Wallet A then transfers `alice.btc` to Wallet B | null (no primary name) |
| Wallet A self-transfers `bob.btc` at block 851000 | `bob.btc` (if A owns bob.btc) |
| Wallet A Solution A inscription at block 852000 claiming `alice.btc` but A doesn't own `alice.btc` | null |
| Both Solution A (block 853000) and Solution B (block 854000) exist; Solution B is newer | Solution B wins |
| Self-transfer at block 839999 (before activation) | null |

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
