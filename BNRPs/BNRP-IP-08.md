# BNRP-IP-08: Messaging Records

| Field | Value |
|-------|-------|
| **BNRP-IP** | 08 |
| **Title** | Messaging Records |
| **Author** | BTCNative Protocol Team |
| **Status** | Draft |
| **Type** | Standards Track |
| **Created** | 2026-04-27 |
| **Requires** | BNRP-IP-00, BNRP-IP-02 |

---

## Abstract

This proposal defines a standard for attaching messaging identifiers to BNRP name records, enabling any `.btc`, `.sats`, `.x`, `.ord`, `.xbt`, `.gm`, `.unisat`, or `.sat` name to function as a messaging address. The initial implementation targets XMTP inbox IDs, with the schema designed to accommodate additional messaging protocols without breaking changes.

---

## Motivation

A Bitcoin name is already a payment address, an identity, and an ownership record. It should also be a messaging address.

Today, to message someone who owns `satoshi.btc`, you need their raw wallet address or a separate account on a centralized platform. Neither is acceptable for a trustless, Bitcoin-native identity layer.

BNRP-IP-08 adds a `messaging` field to the routing inscription. Any application that can resolve a BNRP name can also discover where to send that person a message — without a central directory, without an API key, and without trusting any intermediary.

### Why XMTP

XMTP is the most credible decentralized messaging protocol currently in production:

- End-to-end encrypted using MLS (same protocol used by Signal and WhatsApp, independently audited)
- Quantum-resistant hybrid encryption
- 55M+ reachable inboxes
- Identity-agnostic by design — inbox IDs are stable across wallet changes
- ~$5 per 100,000 messages, paid to decentralized node operators
- Fully open source (nodes, contracts, SDKs)

XMTP currently uses Ethereum addresses as the default identity anchor. BNRP-IP-08 bridges Bitcoin names to XMTP inboxes by storing the XMTP inbox ID directly in the BNRP routing inscription. The name becomes the messaging address.

---

## Specification

### 1. New Field: `messaging`

The `messaging` field is added to the BNRP routing inscription (`"op": "routing"`). It is an object containing one or more messaging protocol entries.

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p...",
  "messaging": {
    "xmtp": "inbox_id_here"
  }
}
```

#### Field definition

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `messaging` | object | No | Messaging identifiers for this name. Omit if not set. |
| `messaging.xmtp` | string | No | XMTP inbox ID (hex string, 64 chars). Must be a valid registered XMTP inbox. |

#### Validation rules

1. `messaging.xmtp` MUST be a 64-character lowercase hex string (the XMTP inbox ID format).
2. The XMTP inbox ID MUST be registered on the XMTP network before it can be meaningfully used. Resolvers MAY optionally verify registration but are not required to.
3. If `messaging` is present but `messaging.xmtp` is not a valid hex string, the field MUST be ignored by resolvers.
4. Multiple messaging protocols may coexist in the `messaging` object. Unknown keys MUST be ignored by resolvers (forward compatibility).

#### Canonical example

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "satoshi.btc",
  "btc_taproot": "bc1pkdqs4ksyha8n2ugxtyywku35pwmv7t60yrru0f860aaf3u5faujq9a6hmc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "messaging": {
    "xmtp": "a1b2c3d4e5f6..."
  }
}
```

### 2. Minimal-only inscription (messaging only)

A name owner who only wants to add messaging support without a full routing record can inscribe a minimal record:

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "messaging": {
    "xmtp": "inbox_id_here"
  }
}
```

This is valid. Fields not present are simply unresolved. A resolver querying `alice.btc` for a taproot address would return null, but querying for messaging would return the XMTP inbox ID.

### 3. REST API extension

BNRP-IP-06 REST API gains one new endpoint:

```
GET /v1/messaging/{name}
```

**Response:**
```json
{
  "name": "alice.btc",
  "messaging": {
    "xmtp": "inbox_id_here"
  },
  "resolved_at_block": 845000
}
```

**404 response** (no messaging record):
```json
{
  "name": "alice.btc",
  "messaging": null,
  "error": "no_messaging_record"
}
```

Existing `/v1/resolve/{name}` and `/v1/profile/{name}` endpoints SHOULD include the `messaging` field in their response if present.

### 4. How to get an XMTP inbox ID

A user registers their XMTP inbox by connecting to any XMTP-compatible app. The inbox ID is derived from the hash of their first associated wallet address and a nonce. It is stable and permanent.

**Steps for a BNRP name owner:**

1. Visit [xmtp.chat](https://xmtp.chat) or any XMTP app (Converse, Coinbase Wallet messaging, etc.)
2. Connect an Ethereum or passkey identity
3. Your XMTP inbox ID is generated automatically
4. Retrieve it via the XMTP SDK:
   ```javascript
   import { Client } from '@xmtp/browser-sdk';
   const client = await Client.create(signer);
   console.log(client.inboxId); // 64-char hex string
   ```
5. Inscribe a BNRP routing record with `"messaging": { "xmtp": "<your_inbox_id>" }`

### 5. Resolution flow for senders

To message `alice.btc`:

```
1. GET https://api.bnrp.name/v1/messaging/alice.btc
   → { "messaging": { "xmtp": "a1b2c3..." } }

2. Open XMTP DM to inbox ID "a1b2c3..."
   → Message delivered, E2E encrypted
```

No wallet address needed. No centralized lookup. The name is the address.

### 6. Future messaging protocols

The `messaging` object is extensible. Reserved key names for future protocols:

| Key | Protocol | Status |
|-----|----------|--------|
| `xmtp` | XMTP | Active (this spec) |
| `nostr` | Nostr (npub) | Reserved |
| `matrix` | Matrix (MXID) | Reserved |
| `simplex` | SimpleX Chat | Reserved |
| `session` | Session | Reserved |

Implementations MUST ignore unknown keys. New keys are added via future BNRP-IP proposals.

### 7. Transfer behavior

When a name changes hands, the `messaging` field from any prior inscription is invalidated along with the rest of the old record. The new owner sets up their own `messaging` record in a fresh inscription.

**Resolver rule:** Resolvers MUST NOT serve a `messaging` record from an inscription that predates the current ownership of the sat. If no valid `messaging` field exists in the canonical (post-transfer) inscription, the resolver MUST return null for messaging.

**Effect on message history:**

- The previous owner retains full access to their XMTP inbox. Their conversation history is intact. The inbox is anchored to their EVM/passkey identity, not to the sat, so transfer of the sat does not transfer or expose message content.
- The new owner cannot access the previous owner's inbox. The XMTP inbox ID is not a credential — it is a routing pointer. Without the private key that controls the inbox, the new owner cannot read or send as the old inbox.
- Senders who re-resolve the name after a transfer will get the new owner's inbox ID (or null if none is set). They cannot accidentally message the new owner while believing they are messaging the previous one.

This behavior follows directly from the clean slate rule: all records from a prior ownership period are invalid after transfer. The `messaging` field is no exception.

### 8. Privacy considerations

- The `messaging` field is public. Anyone who can read a BNRP inscription can find the inbox ID.
- XMTP has built-in spam protection (network-level consent). Senders can be blocked across the entire XMTP network.
- Linking an XMTP inbox ID to a Bitcoin name is opt-in and requires a deliberate inscription.
- Users who want a separate messaging identity from their primary name can use a subname (e.g., `dm.alice.btc`) with its own `messaging` record.
- A single XMTP inbox ID may be referenced by multiple names (e.g., a parent name and a subname). Transfer of one name does not affect routing from the others.

### 9. Security considerations

- The `messaging.xmtp` field does not grant any signing authority over the name or the wallet.
- An attacker cannot impersonate a name owner's messaging by inscribing a fake `messaging` record. Only the canonical (latest valid) BNRP inscription counts, and only the name owner can produce a new valid inscription (they hold the sat).
- XMTP E2E encryption means even BNRP resolvers cannot read message content.
- The inbox ID is not a secret. Knowing it allows someone to send messages but not to read or impersonate. Message content is protected by the recipient's XMTP key, not by the privacy of the inbox ID.

---

## Backward Compatibility

This proposal is fully backward compatible. The `messaging` field is optional. All existing BNRP records without a `messaging` field remain valid. Resolvers that do not implement BNRP-IP-08 simply ignore the new field.

---

## Reference Implementation

BTCNative will implement:

1. `/v1/messaging/{name}` endpoint in the BNRP API worker
2. "Message" button on name profile pages (btcnative.name/name.html) that resolves the inbox ID and deep-links to xmtp.chat
3. Inscription guide at btcnative.name/message.html
4. Updated bnrp.html documentation

---

## Copyright

CC0 1.0 Universal — public domain. No rights reserved.
