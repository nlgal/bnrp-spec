# BNRP-IP-00: BTC Name Resolution Protocol — Master Specification

| Field | Value |
|-------|-------|
| **BNRP-IP** | 00 |
| **Title** | BTC Name Resolution Protocol — Master Specification |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | — |
| **Replaces** | — |

> This document is the principal architecture specification for BNRP. Individual BNRP-IPs (01–06+) define each component in normative detail. This document provides the full rationale, design decisions, examples, security model, and roadmap.

---


# BTC Name Resolution Protocol (BNRP)

## Version: 0.1 — Draft Specification
### Authors: Open Protocol Working Group
### Status: Draft
### Effective Block: 900000 (suggested activation)
### Date: March 2026
### License: CC0 1.0 Universal

---

## 1. Executive Summary

The BTC Name Resolution Protocol (BNRP) is a comprehensive, open protocol specification for resolving human-readable names inscribed on Bitcoin via Ordinals to blockchain addresses, profile metadata, web content, and metaverse coordinates. BNRP extends and formalizes the foundational work of BtcName (btcname.id) — a DID standard for writing names to Bitcoin using plain-text Ordinals inscriptions — into a fully-featured, ENS-equivalent name resolution system rooted in Bitcoin's immutable ledger.

Today, the Ethereum Name Service (ENS) provides Ethereum users with a mature name resolution ecosystem: primary names, reverse resolution, avatar standards, multichain address records, profile metadata, and content resolution (IPFS/IPNS). ENS has become critical infrastructure for Ethereum-based wallets, dApps, and browsers. The Bitcoin ecosystem, despite having multiple name protocols (.btc, .sats, .unisat, .x), lacks a unified, open standard that matches ENS's depth. BtcName has established the foundational layer — name ownership via Ordinals inscriptions, routing records for forward resolution, and draft proposals for primary name assignment — but significant gaps remain in cross-chain address resolution, avatar standardization, web/content resolution, profile records, bitmap/metaverse integration, and open indexer specifications.

BNRP's core design philosophy rests on three pillars. First, **Bitcoin as root of truth**: all name ownership is determined by UTXO ownership of inscription sats, and all canonical records are Ordinals inscriptions on Bitcoin mainnet. There is no trusted registry contract, no centralized database, and no governance token. Second, **open and permissionless**: anyone can run a BNRP indexer, anyone can implement a BNRP resolver, and the specification is governed as an open standard via the BNRP Improvement Proposal (BNRP-IP) process. Third, **no vendor lock-in**: the protocol is designed so that no single company, wallet, or service provider controls resolution infrastructure. Multiple independent indexers and resolvers can coexist, and the specification includes canonicalization rules for handling disagreements between them.

BNRP adds the following capabilities on top of existing BtcName infrastructure: (1) a standardized primary name mechanism with clear winner determination between the two draft proposals (Solution A and Solution B); (2) bidirectional reverse resolution with cryptographic anti-spoofing guarantees; (3) multichain address records supporting EVM L2 chains (Base, Arbitrum, Polygon, and any future chain) via SLIP-44 coin type identifiers; (4) a comprehensive avatar standard supporting Ordinals inscriptions, IPFS, IPNS, HTTPS, and data URIs with ownership verification and fallback chains; (5) web/content resolution via IPFS, IPNS, Arweave, and HTTP gateways, enabling `.btc` domains to resolve to decentralized websites; (6) full profile text records equivalent to ENS's ENSIP-5 (display name, bio, social handles, email, URL); (7) bitmap/metaverse record integration, allowing `.btc` names to point to Bitcoin block districts and parcels; and (8) an open REST API and SDK specification for resolver implementations.

BNRP maintains full backward compatibility with existing BtcName inscriptions. All current `"p": "btcname"` routing and reverse inscriptions remain valid and are indexed by BNRP-compliant resolvers. BNRP introduces a new protocol prefix `"p": "bnrp"` for extended record types while continuing to recognize `"p": "btcname"` inscriptions as authoritative for the fields they define. Existing BtcName users and applications require no migration for current functionality; BNRP is a superset that layers additional capability on an unchanged foundation.

The target adopters for BNRP include: Bitcoin wallets (UniSat, Xverse, Leather, OKX Wallet), blockchain explorers (mempool.space, blockchain.com), Ordinals indexers and marketplaces, dApp developers building on Bitcoin, bitmap/metaverse applications (Atlas, bitmap.directory), cross-chain bridges and aggregators, browser extension developers, and the broader Bitcoin developer community. BNRP is designed to be implementable by any team within weeks using the reference resolver and indexer specifications provided herein.

This document constitutes the principal architecture specification for BNRP. It provides complete JSON schemas, full pseudocode implementations, worked examples, REST API definitions, TypeScript SDK interfaces, security analysis, and a phased roadmap for adoption. It is intended to serve the same role for Bitcoin name resolution that the ENSIP series serves for ENS: a definitive, implementable standard that enables interoperability across the entire ecosystem.

---

## 2. What BtcName Already Provides

### 2.1 Overview

BtcName (btcname.id) is a Decentralized Identifier (DID) standard for writing names to Bitcoin using Ordinals Protocol inscriptions in plain-text JSON format. Names are inscribed as sats on the Bitcoin blockchain, and ownership is determined by the current UTXO holder of the inscription sat. BtcName supports multiple name protocols: `.btc`, `.sats`, `.unisat`, `.x` (on Bitcoin mainnet), and `.fb` (on Fractal). Resolution follows a 3-confirmation rule: an inscription resolves to an address only after receiving 3 block confirmations.

### 2.2 Name Routing Inscription

The routing inscription provides forward resolution — mapping a name to a set of blockchain addresses and metadata:

```json
{
  "p": "btcname",
  "op": "routing",
  "name": "xxx.btc",
  "ord_handle": "xxx",
  "ord_index": "xxxi0",
  "btc_p2phk": "1xxx",
  "btc_p2sh": "3xxx",
  "btc_segwit": "bc1qxxx",
  "btc_lightning": "xxx",
  "eth_address": "0xxxx",
  "matic_address": "0xxxx",
  "sol_address": "xxx",
  "avatar": "xxxi0"
}
```

**Field List:**

| Field | Type | Description |
|-------|------|-------------|
| `p` | string | Protocol identifier (`"btcname"`) |
| `op` | string | Operation type (`"routing"`) |
| `name` | string | The full domain name (e.g., `"alice.btc"`) |
| `ord_handle` | string | Ordinals handle for the name owner |
| `ord_index` | string | Ordinals inscription index |
| `btc_p2phk` | string | Bitcoin P2PKH address (legacy, `1...`) |
| `btc_p2sh` | string | Bitcoin P2SH address (`3...`) |
| `btc_segwit` | string | Bitcoin SegWit address (`bc1q...`) |
| `btc_lightning` | string | Lightning Network address or LNURL |
| `eth_address` | string | Ethereum address (`0x...`) |
| `matic_address` | string | Polygon address (`0x...`) |
| `sol_address` | string | Solana address (base58) |
| `avatar` | string | Inscription ID for avatar image |

### 2.3 Reverse Resolution Inscription

The reverse resolution inscription maps an address back to a name:

```json
{
  "p": "btcname",
  "op": "reverse",
  "reverse_address": "xxx",
  "name": "xxx.btc"
}
```

**Anti-spoofing rule:** The `reverse_address` must be the current owner of `xxx.btc`. Reverse lookup only succeeds if forward lookup of `xxx.btc` returns an address set that includes `reverse_address`.

### 2.4 Primary Name Protocol — Solution Comparison

| Feature | Solution A | Solution B |
|---------|-----------|-----------|
| **Mechanism** | Wallet inscribes a dedicated `primary-name` record | Wallet transfers name to itself (`from == to`) |
| **Verification** | Minter of primary-name inscription must be the owner | Transfer event with matching from/to addresses |
| **Metadata** | Supports avatar field in inscription | No metadata; name only |
| **Mutability** | "Last is New" — latest inscription wins | "Last is New" — latest self-transfer wins |
| **Proxy risk** | Must be self-minted, not via proxy inscribe services | No proxy risk (self-transfer is inherently authenticated) |
| **Effective block** | Not specified | Block 840000 |
| **Max name length** | Not specified | 128 characters |
| **Address types** | Not specified | Native SegWit (`bc1q`) and Taproot (`bc1p`) only |
| **Complexity** | Moderate — requires separate inscription | Simple — leverages existing transfer mechanism |
| **API** | Not specified | `GET /api/v1/primary_name/query?address={wallet_address}` |

### 2.5 Current API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/primary_name/query?address={addr}` | GET | Query primary name for a wallet address |
| Name routing resolution | Via indexer | Forward resolution via BtcName indexer |
| Reverse resolution | Via indexer | Reverse lookup via BtcName indexer |

### 2.6 Current Limitations

1. **No standardized avatar rules** — avatar field accepts inscription IDs but lacks URI scheme definitions, fallback behavior, ownership verification, or MIME type validation.
2. **No web/IPFS content resolution** — no mechanism to point a `.btc` name at a decentralized website or IPFS content.
3. **No bitmap/metaverse records** — no standardized fields for linking `.btc` names to bitmap districts or parcels.
4. **No cross-chain EVM L2 support** — supports ETH and MATIC but not Base, Arbitrum, Optimism, or other EVM L2 chains. No extensible coin-type-based schema.
5. **Single-vendor API** — resolution depends on BtcName's API; no open specification for third-party resolvers.
6. **No open indexer specification** — indexer behavior is implementation-specific, not standardized.
7. **No profile text records** — no bio, display name, social handles, or structured profile metadata beyond the routing inscription.
8. **No name normalization standard** — no defined rules for handling Unicode, case folding, or homoglyph detection.
9. **No browser/gateway resolution** — no mechanism for resolving `.btc` names in web browsers.
10. **Primary name protocol undecided** — two competing draft solutions with no clear standard.

---

## 3. Gaps vs. ENS

| Feature | ENS Status | BtcName Status | BNRP Addresses |
|---------|-----------|---------------|----------------|
| Primary name setting mechanism | Mature (ENSIP-19); on-chain via Reverse Registrar | Draft (Solution A & B proposed, neither standardized) | Standardizes Solution B as default, Solution A as opt-in extension (§9.2) |
| Reverse resolution with anti-spoofing | Mature; bidirectional verification via Reverse Registrar | Implemented with `reverse_address` must match owner | Formalizes bidirectional verification with explicit pseudocode and edge case handling (§9) |
| Multichain address records (EVM L2s) | Mature (ENSIP-9/11); SLIP-44 coin types with EVM chain ID encoding | Partial — supports ETH, MATIC, SOL; no L2s, no coin-type schema | Full SLIP-44 coin type registry; explicit support for Base, Arbitrum, Polygon, extensible to any chain (§5, Appendix A) |
| Avatar standard with ownership verification | Mature (ENSIP-12); supports HTTPS, IPFS, data URIs, NFT references (CAIP-22/29) with ownership verification | Basic — inscription ID in `avatar` field; no URI schemes, no verification, no fallback | Complete avatar standard with `ord:`, `ipfs:`, `ipns:`, `https:`, `data:` URI schemes, ownership verification, fallback chain (§10) |
| Web/content resolution (IPFS/IPNS/contenthash) | Mature; `contenthash` record resolves to IPFS/IPNS/Swarm; eth.limo gateway | Not supported | Full content resolution with IPFS, IPNS, Arweave, `ord://`, HTTPS; gateway specification (§11) |
| Profile text records (bio, twitter, github, email) | Mature (ENSIP-5); arbitrary key-value text records | Not supported beyond routing fields | Full profile record type with ENSIP-5 equivalent fields plus Bitcoin-specific extensions (§6.2) |
| Bitmap/metaverse records | Not applicable (Ethereum-based) | Not supported | Bitmap record type with district, parcel, world, coordinates fields (§12) |
| Open indexer specification | N/A (ENS uses smart contract events) | Not published; single implementation | Open indexer spec with canonicalization rules, conflict resolution, and reference implementation notes (§13, Appendix C) |
| Browser/gateway resolution | eth.limo gateway; ENS browser extension | Not supported | Gateway specification, browser extension architecture, DNS bridge option (§11) |
| Cross-indexer canonicalization rules | N/A (single source of truth via Ethereum state) | Not defined; indexers may diverge | Explicit rules using Bitcoin RPC `gettxout` as ground truth, reorg handling, confirmation thresholds (§8.3) |
| Developer SDK / standardized API | ethers.js, viem, wagmi native integration | Single-vendor API, no SDK | 8-endpoint REST API spec, TypeScript/JavaScript SDK interface with full type definitions (§14) |
| Offchain signed records (CCIP-Read equivalent) | Mature (EIP-3668); revert + gateway callback pattern | Not supported | Bitcoin taproot key signature scheme for off-chain records with verification protocol (§19.3) |

---

## 4. Proposed BNRP Architecture

### 4.1 Design Principles

1. **Bitcoin-Rooted Authority**: All name ownership derives from Bitcoin UTXO ownership of inscription sats. No sidechain, L2, or off-chain system can override on-chain ownership. The Bitcoin blockchain is the sole root of trust for name registration and transfer.

2. **Inscription-First Data Model**: All canonical name records (routing, profile, content, bitmap, primary name, reverse) are Ordinals inscriptions on Bitcoin mainnet. Off-chain records are an optional extension requiring cryptographic proof (taproot key signatures) and are always subordinate to on-chain records.

3. **Permissionless Participation**: Any entity can run a BNRP indexer, any entity can implement a BNRP resolver, and any entity can operate a BNRP gateway. No API keys, no registration, no approval process. The specification is the only requirement.

4. **Backward Compatibility**: All existing BtcName `"p": "btcname"` inscriptions are valid BNRP records. BNRP resolvers MUST index both `"p": "btcname"` and `"p": "bnrp"` inscriptions. No existing user action is invalidated by BNRP adoption.

5. **Deterministic Resolution**: Given the same set of Bitcoin blocks and inscriptions, any two compliant BNRP indexers MUST produce identical resolution results. This requires explicit ordering rules, conflict resolution procedures, and canonicalization standards.

6. **Minimal On-Chain Footprint**: BNRP records are plain-text JSON inscriptions with no custom script requirements. No new opcodes, no new transaction formats, no changes to Bitcoin consensus. Everything operates within the existing Ordinals protocol framework.

7. **Defense in Depth**: Security is layered — name normalization prevents homoglyph attacks, bidirectional verification prevents reverse resolution spoofing, avatar ownership verification prevents impersonation, and multi-resolver quorum (optional) prevents single-point-of-failure resolver attacks.

8. **Progressive Enhancement**: A minimal BNRP implementation can support only forward resolution (routing records). Additional capabilities (profile, avatar, content, bitmap, reverse) are additive. Wallets and apps can adopt BNRP incrementally.

9. **Chain-Agnostic Address Model**: Address records use SLIP-44 coin types as the canonical identifier for each chain, following ENS's ENSIP-9 precedent. This allows any blockchain to be added to a `.btc` name record without protocol changes.

10. **Human-Readable JSON**: All inscription data is plain-text JSON, readable by any text editor or HTTP client. No binary encoding, no protobuf, no custom serialization. This maximizes auditability and reduces implementation complexity.

### 4.2 System Components

**Bitcoin Ordinals Layer (Source of Truth)**

The base layer of BNRP is the Bitcoin blockchain with the Ordinals protocol. Name inscriptions (`.btc`, `.sats`, `.unisat`, `.x`, `.fb`) are individual sats inscribed with name data. Routing, profile, content, bitmap, primary-name, and reverse inscriptions are additional inscriptions authored by the name owner. Ownership of any inscription is determined by the current UTXO holder of the sat on which the inscription resides. The Ordinals layer provides: immutable registration ordering (first-come-first-served for name claims), cryptographically proven minter identity (the address that spent the tapscript input in the reveal transaction), and transferable ownership (via standard Bitcoin UTXO transfers).

**BNRP Indexer Layer (Open Specification)**

A BNRP indexer is a service that reads the Bitcoin blockchain, identifies BNRP-relevant inscriptions (both `"p": "btcname"` and `"p": "bnrp"`), and maintains a queryable database of name records, ownership, and resolution state. The indexer specification defines: which inscription types to track, how to determine current ownership (UTXO lookup), how to handle chain reorganizations (rollback to last common ancestor), the minimum confirmation threshold (3 confirmations), and the "Last is New" override rule. Multiple independent indexer implementations can coexist; canonicalization rules (§8.3) ensure they converge on the same state.

**BNRP Resolver API (Standardized REST API)**

The resolver is an HTTP service that exposes BNRP resolution endpoints. Any party can implement and host a resolver. The API specification (§14.1) defines 8 endpoints covering forward resolution, reverse resolution, address lookup, profile retrieval, avatar resolution, content resolution, bitmap lookup, and health checking. Resolvers query their own indexer instance (or a shared indexer) and return standardized JSON responses. Multiple resolvers can be operated independently for redundancy and decentralization.

**BNRP Gateway (HTTP Gateway for Web Resolution)**

A BNRP gateway enables `.btc` names to resolve to web content in standard browsers without extensions. The gateway operates as a web server that accepts requests like `https://alice.btc.gateway.io/`, resolves `alice.btc` via BNRP, retrieves the content pointer (IPFS CID, HTTPS URL, Arweave TX, or Ordinals inscription), and serves the content to the browser. Multiple independent gateways can operate simultaneously, similar to ENS's eth.limo service.

**BNRP SDK (Client Library)**

The BNRP SDK is a standardized client library (initially JavaScript/TypeScript, with Python planned) that wallets and dApps use to perform BNRP resolution. The SDK wraps the resolver REST API, handles retry logic across multiple resolvers, performs client-side name normalization, and provides type-safe interfaces for all resolution operations. The SDK specification (§14.2) defines the interface contract that any conforming implementation must satisfy.

**Browser Extension (Optional)**

A browser extension intercepts `.btc` TLD navigation in the browser, resolves the name via the BNRP SDK, and either redirects to a gateway or serves content directly via IPFS. This is an optional component for power users who want native `.btc` resolution without depending on a gateway.

### 4.3 Resolution Layers Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER / APPLICATION                        │
│   (Wallet, dApp, Browser, Explorer, Bitmap App)                  │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    BNRP SDK / API CLIENT                          │
│   • Name normalization (§8.1)                                    │
│   • Multi-resolver failover                                      │
│   • Client-side caching                                          │
│   • Type-safe resolution interfaces                              │
└──────────────────────┬───────────────────────────────────────────┘
                       │  REST API calls (§14.1)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                 BNRP RESOLVER NETWORK                             │
│   (Multiple independent resolvers — any party can operate)       │
│                                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│   │ Resolver A   │  │ Resolver B   │  │ Resolver C   │            │
│   │ (BtcName.id) │  │ (Community)  │  │ (Self-hosted)│            │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│          │                │                │                     │
└──────────┼────────────────┼────────────────┼─────────────────────┘
           │                │                │
           ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    BNRP INDEXER LAYER                             │
│   (Reads Bitcoin blocks, indexes Ordinals inscriptions)          │
│   • Tracks "p": "btcname" and "p": "bnrp" inscriptions          │
│   • Maintains ownership via UTXO tracking                        │
│   • Applies "Last is New" ordering                               │
│   • Enforces 3-confirmation threshold                            │
│   • Handles chain reorganizations                                │
└──────────────────────┬───────────────────────────────────────────┘
                       │  Bitcoin RPC / block data
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    BITCOIN BLOCKCHAIN                             │
│   (Root of truth for all name ownership and inscription data)    │
│                                                                  │
│   • Ordinals inscriptions (name registrations, routing,          │
│     profile, content, bitmap, primary-name, reverse)             │
│   • UTXO set (determines current inscription ownership)          │
│   • Block confirmations (minimum 3 for state changes)            │
└──────────────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────────────────┐
│                    BNRP GATEWAY (for web resolution)             │
│                                                                  │
│   Browser → https://alice.btc.gateway.io/                        │
│                       │                                          │
│                       ▼                                          │
│              BNRP Resolver API                                   │
│                       │                                          │
│                       ▼                                          │
│           Content record for alice.btc                           │
│           (ipfs://Qm..., https://..., ar://..., ord://...)       │
│                       │                                          │
│                       ▼                                          │
│           Fetch content → serve to browser                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Canonical Data Model

### 5.1 Name Record Types

BNRP defines six canonical record types, each identified by the `op` field in the inscription JSON:

| Record Type | `op` Value | Purpose | Inscriber Requirement |
|-------------|-----------|---------|----------------------|
| **Routing** | `"routing"` | Forward resolution: maps a name to blockchain addresses and metadata | Must be inscribed by current owner of the name inscription |
| **Reverse** | `"reverse"` | Reverse resolution claim: maps an address to a primary name | Must be inscribed by the address owner; subject to anti-spoofing verification |
| **Primary Name** | `"primary-name"` | Primary name declaration (Solution A extended) | Must be self-minted by the name owner (minter == current owner) |
| **Profile** | `"profile"` | Profile metadata: display name, bio, social handles, contact info | Must be inscribed by current owner of the name inscription |
| **Content** | `"content"` | Web/IPFS content pointer for browser/gateway resolution | Must be inscribed by current owner of the name inscription |
| **Bitmap** | `"bitmap"` | Bitmap/metaverse pointer: district, parcel, world coordinates | Must be inscribed by current owner of the name inscription |

All record types share a common header:

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "<record_type>",
  "name": "<fully_qualified_name>"
}
```

- `p`: Protocol identifier. MUST be `"bnrp"` for new BNRP records. Resolvers also accept `"btcname"` for backward compatibility.
- `v`: Protocol version. MUST be `"1"` for this specification.
- `op`: Operation type. One of: `"routing"`, `"reverse"`, `"primary-name"`, `"profile"`, `"content"`, `"bitmap"`.
- `name`: The fully qualified domain name (e.g., `"alice.btc"`), normalized per §8.1.

### 5.2 Field Registry

The following table defines all standardized fields across all record types. Fields marked with `*` are required for that record type; all others are optional.

| Field Name | Used In | Type | Description |
|------------|---------|------|-------------|
| `p` | All* | string | Protocol identifier (`"bnrp"` or `"btcname"`) |
| `v` | All | string | Protocol version (`"1"`) |
| `op` | All* | string | Operation type |
| `name` | All* | string | Fully qualified domain name |
| `btc_taproot` | routing | string | Bitcoin Taproot address (`bc1p...`) |
| `btc_segwit` | routing | string | Bitcoin Native SegWit address (`bc1q...`) |
| `btc_legacy` | routing | string | Bitcoin Legacy P2PKH address (`1...`) |
| `btc_p2sh` | routing | string | Bitcoin P2SH address (`3...`) |
| `btc_lightning` | routing | string | Lightning Network address, invoice, or LNURL |
| `eth` | routing | string | Ethereum mainnet address (`0x...`) |
| `sol` | routing | string | Solana address (base58) |
| `addr_{coinType}` | routing | string | Generic chain address by SLIP-44 coin type |
| `avatar` | routing, profile, primary-name | string | Avatar URI (see §10.1 for schemes) |
| `avatar_unverified` | routing, profile | boolean | If `true`, avatar ownership is not verified |
| `avatar_proof` | routing, profile | string | Signature of avatar URI by owner key |
| `content` | routing, content | string | Content/web URI (IPFS, IPNS, HTTPS, Arweave, Ordinals) |
| `display` | profile | string | Display name (must case-fold match `name`) |
| `description` | profile | string | Free-text description/bio |
| `url` | profile | string | Website URL |
| `email` | profile | string | Email address |
| `com.twitter` | profile | string | Twitter/X username |
| `com.github` | profile | string | GitHub username |
| `com.discord` | profile | string | Discord username |
| `com.nostr` | profile | string | Nostr npub or NIP-05 identifier |
| `com.telegram` | profile | string | Telegram username |
| `theme` | profile | string | Profile theme identifier or hex color |
| `location` | profile | string | Geographic location string |
| `primary_contact` | profile | string | Preferred contact method (e.g., `"email"`, `"twitter"`, `"nostr"`) |
| `keywords` | profile | string | Comma-separated keywords |
| `reverse_address` | reverse* | string | Address claiming reverse resolution |
| `bitmap_district` | bitmap | string | Bitmap block height (district number) |
| `bitmap_parcel` | bitmap | string | Transaction index within the district |
| `bitmap_world` | bitmap | string | Metaverse world identifier |
| `bitmap_coordinates` | bitmap | string | Coordinate string (e.g., `"x:100,y:200"`) |
| `bitmap_avatar` | bitmap | string | Bitmap-specific avatar URI |
| `bitmap_profile_url` | bitmap | string | External profile URL for bitmap presence |
| `ord_handle` | routing | string | Ordinals handle (BtcName legacy field) |
| `ord_index` | routing | string | Ordinals inscription index (BtcName legacy field) |

---

## 6. JSON Schemas

All schemas use JSON Schema Draft-07 format. These schemas are normative — BNRP indexers MUST validate inscriptions against these schemas before indexing them.

### 6.1 Routing Record Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/routing.json",
  "title": "BNRP Routing Record",
  "description": "Forward resolution record mapping a .btc name to blockchain addresses and metadata.",
  "type": "object",
  "required": ["p", "op", "name"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp", "btcname"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "routing",
      "description": "Operation type for routing records."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,251}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 253,
      "description": "Fully qualified, normalized domain name."
    },
    "btc_taproot": {
      "type": "string",
      "pattern": "^bc1p[a-z0-9]{58}$",
      "description": "Bitcoin Taproot (P2TR) address."
    },
    "btc_segwit": {
      "type": "string",
      "pattern": "^bc1q[a-z0-9]{38,58}$",
      "description": "Bitcoin Native SegWit (P2WPKH/P2WSH) address."
    },
    "btc_legacy": {
      "type": "string",
      "pattern": "^1[a-km-zA-HJ-NP-Z1-9]{25,34}$",
      "description": "Bitcoin Legacy P2PKH address."
    },
    "btc_p2sh": {
      "type": "string",
      "pattern": "^3[a-km-zA-HJ-NP-Z1-9]{25,34}$",
      "description": "Bitcoin P2SH address."
    },
    "btc_lightning": {
      "type": "string",
      "description": "Lightning Network payment address, BOLT-11 invoice, or LNURL."
    },
    "eth": {
      "type": "string",
      "pattern": "^0x[0-9a-fA-F]{40}$",
      "description": "Ethereum mainnet address (EIP-55 checksummed)."
    },
    "sol": {
      "type": "string",
      "pattern": "^[1-9A-HJ-NP-Za-km-z]{32,44}$",
      "description": "Solana address (base58)."
    },
    "ord_handle": {
      "type": "string",
      "description": "Ordinals handle (BtcName legacy compatibility)."
    },
    "ord_index": {
      "type": "string",
      "description": "Ordinals inscription index (BtcName legacy compatibility)."
    },
    "avatar": {
      "type": "string",
      "description": "Avatar URI. Supported schemes: ord:, ipfs://, ipns://, https://, data:."
    },
    "avatar_unverified": {
      "type": "boolean",
      "default": false,
      "description": "If true, avatar ownership verification is skipped."
    },
    "avatar_proof": {
      "type": "string",
      "description": "Base64-encoded signature of avatar URI by owner's taproot key."
    },
    "content": {
      "type": "string",
      "description": "Content/web URI. Supported schemes: ipfs://, ipns://, https://, ar://, ord://."
    },
    "bitmap_district": {
      "type": "string",
      "pattern": "^[0-9]+$",
      "description": "Bitmap district (block height)."
    },
    "bitmap_parcel": {
      "type": "string",
      "pattern": "^[0-9]+$",
      "description": "Bitmap parcel (transaction index within district)."
    }
  },
  "patternProperties": {
    "^addr_[0-9]+$": {
      "type": "string",
      "description": "Chain-specific address keyed by SLIP-44 coin type. E.g., addr_2155872277 for Base."
    }
  },
  "additionalProperties": false
}
```

### 6.2 Profile Record Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/profile.json",
  "title": "BNRP Profile Record",
  "description": "Profile metadata record for a .btc name. Equivalent to ENS ENSIP-5 text records.",
  "type": "object",
  "required": ["p", "op", "name"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "profile",
      "description": "Operation type for profile records."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,251}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 253,
      "description": "Fully qualified, normalized domain name."
    },
    "display": {
      "type": "string",
      "maxLength": 253,
      "description": "Canonical display name. When case-folded, MUST match the name field."
    },
    "description": {
      "type": "string",
      "maxLength": 1000,
      "description": "Free-text description or bio."
    },
    "url": {
      "type": "string",
      "format": "uri",
      "description": "Primary website URL."
    },
    "email": {
      "type": "string",
      "format": "email",
      "description": "Contact email address."
    },
    "com.twitter": {
      "type": "string",
      "maxLength": 50,
      "description": "Twitter/X username (without @)."
    },
    "com.github": {
      "type": "string",
      "maxLength": 50,
      "description": "GitHub username."
    },
    "com.discord": {
      "type": "string",
      "maxLength": 50,
      "description": "Discord username."
    },
    "com.nostr": {
      "type": "string",
      "description": "Nostr public key (npub) or NIP-05 identifier."
    },
    "com.telegram": {
      "type": "string",
      "maxLength": 50,
      "description": "Telegram username (without @)."
    },
    "avatar": {
      "type": "string",
      "description": "Avatar URI. Supported schemes: ord:, ipfs://, ipns://, https://, data:."
    },
    "avatar_unverified": {
      "type": "boolean",
      "default": false,
      "description": "If true, avatar ownership verification is skipped."
    },
    "theme": {
      "type": "string",
      "maxLength": 100,
      "description": "Profile theme identifier or hex color code (e.g., '#FF5733' or 'dark')."
    },
    "location": {
      "type": "string",
      "maxLength": 200,
      "description": "Geographic location (e.g., 'San Francisco, CA')."
    },
    "primary_contact": {
      "type": "string",
      "enum": ["email", "twitter", "github", "discord", "nostr", "telegram", "url"],
      "description": "Preferred contact method."
    },
    "keywords": {
      "type": "string",
      "maxLength": 500,
      "description": "Comma-separated keywords, ordered by significance."
    }
  },
  "additionalProperties": false
}
```

### 6.3 Content Record Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/content.json",
  "title": "BNRP Content Record",
  "description": "Web/IPFS content pointer for browser and gateway resolution.",
  "type": "object",
  "required": ["p", "op", "name", "content"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "content",
      "description": "Operation type for content records."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,251}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 253,
      "description": "Fully qualified, normalized domain name."
    },
    "content": {
      "type": "string",
      "description": "Content URI. MUST use one of: ipfs://{CID}, ipns://{key}, https://{url}, ar://{txid}, ord://{inscriptionId}."
    },
    "content_type": {
      "type": "string",
      "enum": ["website", "dapp", "file", "redirect"],
      "description": "Hint for how the content should be interpreted."
    },
    "content_encoding": {
      "type": "string",
      "enum": ["identity", "gzip", "br"],
      "default": "identity",
      "description": "Content encoding, if applicable."
    },
    "content_hash": {
      "type": "string",
      "description": "SHA-256 hash of the content for integrity verification."
    },
    "ttl": {
      "type": "integer",
      "minimum": 60,
      "maximum": 86400,
      "default": 3600,
      "description": "Cache TTL in seconds for the content record."
    }
  },
  "additionalProperties": false
}
```

### 6.4 Bitmap Record Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/bitmap.json",
  "title": "BNRP Bitmap Record",
  "description": "Bitmap/metaverse pointer linking a .btc name to a Bitcoin block district and parcel.",
  "type": "object",
  "required": ["p", "op", "name", "bitmap_district"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "bitmap",
      "description": "Operation type for bitmap records."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,251}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 253,
      "description": "Fully qualified, normalized domain name."
    },
    "bitmap_district": {
      "type": "string",
      "pattern": "^[0-9]{1,10}$",
      "description": "Bitmap district number (Bitcoin block height)."
    },
    "bitmap_parcel": {
      "type": "string",
      "pattern": "^[0-9]+$",
      "description": "Bitmap parcel number (transaction index within the block)."
    },
    "bitmap_world": {
      "type": "string",
      "maxLength": 100,
      "description": "Metaverse world or realm identifier."
    },
    "bitmap_coordinates": {
      "type": "string",
      "pattern": "^x:-?[0-9]+,y:-?[0-9]+$",
      "description": "Coordinate pair within the bitmap world (e.g., 'x:100,y:200')."
    },
    "bitmap_avatar": {
      "type": "string",
      "description": "Avatar URI specific to the bitmap/metaverse presence."
    },
    "bitmap_profile_url": {
      "type": "string",
      "format": "uri",
      "description": "External URL for the bitmap profile or landing page."
    },
    "bitmap_inscription_id": {
      "type": "string",
      "description": "Inscription ID of the bitmap inscription ({txid}i{index})."
    }
  },
  "additionalProperties": false
}
```

### 6.5 Primary Name Record Schema (Solution A Extended)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/primary-name.json",
  "title": "BNRP Primary Name Record (Solution A)",
  "description": "Primary name declaration via dedicated inscription. The minter of this inscription MUST be the current owner of the referenced name.",
  "type": "object",
  "required": ["p", "op", "name"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp", "btcname"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "primary-name",
      "description": "Operation type for primary name declaration."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,126}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 128,
      "description": "The name being declared as primary. Max 128 characters."
    },
    "avatar": {
      "type": "string",
      "description": "Avatar URI to associate with the primary name."
    },
    "display": {
      "type": "string",
      "maxLength": 128,
      "description": "Preferred display form of the name (case-fold must match name)."
    }
  },
  "additionalProperties": false
}
```

**Validation Rules for Solution A:**
1. The minter of the primary-name inscription (the address that spent the tapscript input in the reveal transaction) MUST be the current owner of the `name` inscription at the time of the primary-name inscription's confirmation.
2. The inscription MUST be self-minted — it must NOT have been created via a proxy inscribe service (verified by checking that the reveal transaction input address matches the name inscription owner).
3. "Last is New" rule: if multiple primary-name inscriptions exist for the same address, the most recently confirmed one is canonical.

### 6.6 Reverse Resolution Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://bnrp.org/schemas/v1/reverse.json",
  "title": "BNRP Reverse Resolution Record",
  "description": "Reverse resolution claim mapping a blockchain address to a .btc name.",
  "type": "object",
  "required": ["p", "op", "reverse_address", "name"],
  "properties": {
    "p": {
      "type": "string",
      "enum": ["bnrp", "btcname"],
      "description": "Protocol identifier."
    },
    "v": {
      "type": "string",
      "const": "1",
      "description": "Protocol version."
    },
    "op": {
      "type": "string",
      "const": "reverse",
      "description": "Operation type for reverse resolution."
    },
    "reverse_address": {
      "type": "string",
      "description": "The blockchain address claiming reverse resolution. Must be the current owner of the name."
    },
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{0,251}\\.(btc|sats|unisat|x|fb)$",
      "maxLength": 253,
      "description": "The name this address claims as its reverse resolution."
    }
  },
  "additionalProperties": false
}
```

---

## 7. Full Example: alice.btc

### 7.1 alice.btc — Full Routing Inscription

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
  "btc_legacy": "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
  "btc_p2sh": "3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy",
  "btc_lightning": "alice@getalby.com",
  "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "sol": "7EcDhSYGxXyscszYEp35KHN8vvw3svAuLKTzXwCFLtV",
  "addr_2155872277": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "addr_2189526273": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "addr_2147483785": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "content": "ipfs://QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn",
  "bitmap_district": "840000",
  "bitmap_parcel": "42",
  "ord_handle": "alice",
  "ord_index": "9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
}
```

**Notes:**
- `addr_2155872277` is the Base address (chainId 8453 | 0x80000000 = 2155872277).
- `addr_2189526273` is the Arbitrum One address (chainId 42161 | 0x80000000 = 2189526273).
- `addr_2147483785` is the Polygon address (chainId 137 | 0x80000000 = 2147483785).
- Alice uses the same EVM address across all EVM chains.
- The avatar points to an Ordinals inscription using the `ord:` URI scheme.
- The content points to an IPFS directory for her decentralized website.

### 7.2 alice.btc — Profile Inscription

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "profile",
  "name": "alice.btc",
  "display": "Alice.btc",
  "description": "Bitcoin maximalist, Ordinals collector, and builder of decentralized identity infrastructure. Contributing to the open Bitcoin name standard.",
  "url": "https://alice.btc.example.com",
  "email": "alice@protonmail.com",
  "com.twitter": "alice_btc",
  "com.github": "alice-btc-dev",
  "com.discord": "alice.btc",
  "com.nostr": "npub1qe3e5wrvnsgpggtkytxteaqfprz0rgxr8c3l34kk3a9t7e2l3ach5xfhz",
  "com.telegram": "alice_btc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "theme": "#F7931A",
  "location": "Cyberspace",
  "primary_contact": "nostr",
  "keywords": "bitcoin,ordinals,btcname,identity,decentralized"
}
```

### 7.3 alice.btc — Primary Name Declaration (Solution A)

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "primary-name",
  "name": "alice.btc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "display": "Alice.btc"
}
```

**Verification context:** This inscription was minted by address `bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297`, which is the same address that currently holds the `alice.btc` name inscription. The minter identity is immutable and was cryptographically established when the tapscript input was spent in the reveal transaction. The indexer verifies that the minter address matches the current owner of `alice.btc` at the time this primary-name inscription achieves 3 confirmations.

---

## 8. Resolution Rules

### 8.1 Name Normalization

BNRP defines a name normalization procedure analogous to ENSIP-15/UTS-46 but adapted for Bitcoin name conventions. All names MUST be normalized before any resolution operation.

**Normalization Rules:**

1. **Case folding**: Convert the entire name to lowercase. `Alice.BTC` → `alice.btc`.
2. **Unicode NFC normalization**: Apply Unicode Normalization Form C (NFC) to compose characters. `e\u0301` → `é`.
3. **Label separation**: Split the name on `.` (U+002E FULL STOP). Each segment between dots is a label. The final label is the TLD (e.g., `btc`, `sats`).
4. **Allowed characters per label**: Each label (excluding the TLD) MUST match: `[a-z0-9\u00C0-\u024F\u0400-\u04FF\u4E00-\u9FFF-]` — lowercase ASCII alphanumerics, hyphens, and select Unicode ranges (Latin Extended, Cyrillic, CJK Unified Ideographs). Unicode labels MUST be flagged for homoglyph warning in UX.
5. **Leading/trailing hyphens forbidden**: No label may begin or end with a hyphen (`-`).
6. **Double hyphens restricted**: Labels MUST NOT contain `--` (double hyphen) at positions 3-4, except for valid Internationalized Domain Name (IDN) `xn--` prefixes.
7. **Empty labels forbidden**: No label may be empty (no `..` in the name).
8. **Maximum total length**: 253 characters (including dots and TLD).
9. **Maximum label length**: 63 characters per label (excluding the TLD).
10. **Valid TLDs**: `btc`, `sats`, `unisat`, `x`, `fb`. Any name with a TLD not in this set MUST be rejected.
11. **Reserved labels**: The following labels are reserved and MUST NOT be registered as standalone names: `btc`, `bitmap`, `ord`, `sat`, `bnrp`, `protocol`. These may appear as part of longer names (e.g., `mybitcoin.btc` is valid).
12. **Zero-width characters**: All zero-width characters (U+200B, U+200C, U+200D, U+FEFF) MUST be stripped before validation.
13. **Confusable detection**: Resolvers SHOULD maintain a confusable character map (per Unicode Technical Report #39) and flag names containing characters from multiple scripts (e.g., mixed Latin and Cyrillic) for UX warning.

**Normalization Pseudocode:**

```pseudocode
function normalize(name: string) -> string | Error:
    // Step 1: Strip zero-width characters
    name = stripZeroWidth(name)
    
    // Step 2: Unicode NFC normalization
    name = unicodeNFC(name)
    
    // Step 3: Case fold to lowercase
    name = toLowercase(name)
    
    // Step 4: Split into labels
    labels = name.split(".")
    if labels.length < 2:
        return Error("INVALID_NAME: must have at least one label and a TLD")
    
    tld = labels[labels.length - 1]
    nameLabels = labels[0 .. labels.length - 2]
    
    // Step 5: Validate TLD
    if tld not in ["btc", "sats", "unisat", "x", "fb"]:
        return Error("INVALID_TLD: unsupported TLD '" + tld + "'")
    
    // Step 6: Validate each label
    for label in nameLabels:
        if label.length == 0:
            return Error("INVALID_NAME: empty label")
        if label.length > 63:
            return Error("INVALID_NAME: label exceeds 63 characters")
        if label[0] == '-' or label[label.length - 1] == '-':
            return Error("INVALID_NAME: label must not start or end with hyphen")
        if label[2..4] == '--' and label[0..2] != 'xn':
            return Error("INVALID_NAME: double hyphen at positions 3-4 reserved for IDN")
        if not matchesAllowedCharset(label):
            return Error("INVALID_NAME: label contains disallowed characters")
    
    // Step 7: Check total length
    normalizedName = join(nameLabels, ".") + "." + tld
    if normalizedName.length > 253:
        return Error("INVALID_NAME: exceeds 253 character limit")
    
    // Step 8: Check reserved labels
    if nameLabels.length == 1 and nameLabels[0] in RESERVED_LABELS:
        return Error("RESERVED_NAME: '" + nameLabels[0] + "' is reserved")
    
    // Step 9: Flag confusables (non-blocking warning)
    scripts = detectScripts(normalizedName)
    if scripts.length > 1:
        warn("CONFUSABLE_WARNING: name contains characters from multiple scripts")
    
    return normalizedName
```

### 8.2 Forward Resolution Flow

```pseudocode
function resolve(name: string) -> BNRPRecord | Error:
    // Step 1: Normalize the name
    normalizedName = normalize(name)
    if normalizedName is Error:
        return normalizedName
    
    // Step 2: Query indexer for the name inscription
    nameInscription = indexer.getNameInscription(normalizedName)
    if nameInscription is null:
        return Error("NAME_NOT_FOUND: no inscription exists for '" + normalizedName + "'")
    
    // Step 3: Verify confirmation depth
    currentBlock = bitcoin.getBlockHeight()
    inscriptionBlock = nameInscription.confirmationBlock
    if (currentBlock - inscriptionBlock) < 3:
        return Error("INSUFFICIENT_CONFIRMATIONS: name inscription has " +
                     (currentBlock - inscriptionBlock) + " confirmations, requires 3")
    
    // Step 4: Determine current owner via UTXO
    ownerAddress = indexer.getInscriptionOwner(nameInscription.inscriptionId)
    if ownerAddress is null:
        return Error("OWNER_NOT_FOUND: cannot determine UTXO owner for inscription")
    
    // Step 5: Find the most recent valid routing record
    //   Search for both "p": "bnrp" and "p": "btcname" routing inscriptions
    //   Filter: inscription must have been created by ownerAddress (or a previous owner at time of creation)
    //   Order by: block height DESC, transaction index DESC (Last is New)
    routingInscriptions = indexer.getRoutingInscriptions(normalizedName)
    routingInscriptions = routingInscriptions.filter(r =>
        r.creatorAddress == indexer.getInscriptionOwner(nameInscription.inscriptionId,
                                                         atBlock=r.confirmationBlock)
        and (currentBlock - r.confirmationBlock) >= 3
    )
    routingInscriptions = routingInscriptions.sortBy(r => r.confirmationBlock DESC,
                                                            r.txIndex DESC)
    
    if routingInscriptions.length == 0:
        // No routing record — return minimal record with just ownership info
        return BNRPRecord {
            name: normalizedName,
            owner: ownerAddress,
            addresses: {},
            profile: null,
            content: null,
            bitmap: null,
            avatar: null
        }
    
    latestRouting = routingInscriptions[0]
    
    // Step 6: Parse routing record
    record = parseRoutingJSON(latestRouting.content)
    if record is Error:
        return Error("INVALID_RECORD: routing inscription contains invalid JSON")
    
    // Step 7: Validate address formats for each chain
    addresses = {}
    if record.btc_taproot and isValidTaprootAddress(record.btc_taproot):
        addresses["btc_taproot"] = record.btc_taproot
    if record.btc_segwit and isValidSegwitAddress(record.btc_segwit):
        addresses["btc_segwit"] = record.btc_segwit
    if record.btc_legacy and isValidLegacyAddress(record.btc_legacy):
        addresses["btc_legacy"] = record.btc_legacy
    if record.btc_p2sh and isValidP2SHAddress(record.btc_p2sh):
        addresses["btc_p2sh"] = record.btc_p2sh
    if record.btc_lightning:
        addresses["btc_lightning"] = record.btc_lightning
    if record.eth and isValidEthAddress(record.eth):
        addresses["eth"] = record.eth
    if record.sol and isValidSolAddress(record.sol):
        addresses["sol"] = record.sol
    // Parse all addr_{coinType} fields
    for field in record.fields matching "addr_[0-9]+":
        coinType = parseInt(field.replace("addr_", ""))
        addresses[field] = record[field]
    // BtcName legacy field mapping
    if record.eth_address and not addresses["eth"]:
        addresses["eth"] = record.eth_address
    if record.matic_address and not addresses["addr_2147483785"]:
        addresses["addr_2147483785"] = record.matic_address
    if record.sol_address and not addresses["sol"]:
        addresses["sol"] = record.sol_address
    
    // Step 8: Fetch profile record (if exists, most recent wins)
    profile = fetchLatestRecord(normalizedName, ownerAddress, "profile")
    
    // Step 9: Fetch content record
    contentRecord = fetchLatestRecord(normalizedName, ownerAddress, "content")
    contentURI = contentRecord?.content ?? record.content ?? null
    
    // Step 10: Fetch bitmap record
    bitmapRecord = fetchLatestRecord(normalizedName, ownerAddress, "bitmap")
    
    // Step 11: Determine avatar (profile overrides routing)
    avatarURI = profile?.avatar ?? record.avatar ?? null
    
    // Step 12: Assemble and return complete record
    return BNRPRecord {
        name: normalizedName,
        owner: ownerAddress,
        inscriptionId: nameInscription.inscriptionId,
        addresses: addresses,
        profile: profile,
        content: contentURI,
        bitmap: bitmapRecord,
        avatar: avatarURI,
        resolvedAt: currentBlock,
        ttl: 600   // 10 minutes default cache TTL
    }
```

### 8.3 Canonical Ownership Determination

When multiple indexers disagree on the current owner of an inscription, the following canonicalization rules apply:

1. **Ground truth**: The canonical owner of an inscription is the address that controls the UTXO containing the inscription sat at the current chain tip. This can be verified by any Bitcoin full node using the `gettxout` RPC command on the transaction output holding the inscription.

2. **Verification procedure**: If two resolvers return different owners for the same inscription:
    ```pseudocode
    function verifyOwner(inscriptionId: string) -> string:
        // Parse the inscription ID to get the genesis txid and index
        genesisTxid = inscriptionId.split("i")[0]
        genesisVout = parseInt(inscriptionId.split("i")[1])
        
        // Trace the inscription sat through the UTXO graph
        // This requires an Ordinals-aware tracer that follows sat flow
        currentUtxo = ordTracer.traceInscriptionSat(genesisTxid, genesisVout)
        
        // Verify the UTXO is unspent using Bitcoin Core RPC
        utxoInfo = bitcoinRPC.gettxout(currentUtxo.txid, currentUtxo.vout, true)
        if utxoInfo is null:
            return Error("UTXO_SPENT: inscription sat UTXO has been spent")
        
        // Extract the address from the scriptPubKey
        ownerAddress = utxoInfo.scriptPubKey.address
        return ownerAddress
    ```

3. **Minimum confirmation threshold**: Any state change (new inscription, transfer) requires 3 block confirmations before it is considered canonical. Resolvers MUST NOT return data from inscriptions with fewer than 3 confirmations.

4. **Reorganization handling**: When a chain reorganization (reorg) occurs:
    - The indexer MUST detect the reorg by monitoring for parent block hash mismatches.
    - Roll back all state changes introduced by the orphaned blocks.
    - Re-index from the last common ancestor block.
    - Any inscriptions that existed only in orphaned blocks are invalidated.
    - Any transfers that existed only in orphaned blocks are reverted.
    - Resolvers MUST return HTTP 503 during reorg processing with a `Retry-After` header.

5. **Inscription ID is canonical**: Inscription IDs (`{txid}i{index}`) are the only stable identifier for inscriptions. Inscription numbers (e.g., `#35648`) may change due to reindexing and MUST NOT be used as canonical references in any BNRP record or resolution logic.

### 8.4 Record Priority and Override Rules

1. **Last is New**: When multiple valid records of the same type exist for the same name, the most recently confirmed record is canonical. Ordering: block height descending, then transaction index within the block descending, then inscription index within the transaction descending.

2. **Owner validation**: A routing, profile, content, or bitmap inscription is valid only if its creator was the owner of the name inscription at the time the record inscription was confirmed. If the name has since been transferred to a new owner, existing records remain valid until the new owner creates replacement records.

3. **Profile overrides routing for profile fields**: If both a routing inscription and a profile inscription define the `avatar` field, the most recent profile inscription's value takes precedence. Routing inscriptions remain authoritative for all address fields.

4. **Content record overrides routing for content field**: A dedicated content inscription (`op: "content"`) takes precedence over the `content` field in a routing inscription.

5. **Bitmap record overrides routing for bitmap fields**: A dedicated bitmap inscription (`op: "bitmap"`) takes precedence over `bitmap_district`/`bitmap_parcel` fields in a routing inscription.

6. **Protocol version precedence**: A `"p": "bnrp"` inscription of the same type and with a more recent confirmation takes precedence over a `"p": "btcname"` inscription. If only `"p": "btcname"` inscriptions exist, they are used with field name mapping (e.g., `eth_address` → `eth`).

---

## 9. Reverse Resolution Rules

### 9.1 Anti-Spoofing Guarantee

Reverse resolution in BNRP employs a mandatory bidirectional verification to prevent spoofing. The guarantee works as follows:

**Forward direction**: `name.btc` resolves to a set of addresses `{A1, A2, ..., An}` via the routing inscription.

**Reverse direction**: Wallet address `W` claims `name.btc` as its primary name (via Solution A inscription, Solution B self-transfer, or explicit reverse inscription).

**Verification**: The reverse resolution is valid if and only if `W` appears in the resolved address set for `name.btc`. Formally:

```
reverseResolve(W) = name.btc  IF AND ONLY IF
    W ∈ resolve(name.btc).allAddresses()
```

Where `allAddresses()` returns the union of all address fields: `{btc_taproot, btc_segwit, btc_legacy, btc_p2sh, eth, sol, addr_*}`.

If the forward lookup does not include `W` in any address field, the reverse resolution MUST return null, regardless of the reverse/primary-name inscription status. This prevents:
- An attacker who briefly owned a name from claiming it as their primary name after transferring it away.
- A user from claiming a name they own but have not configured routing for.

### 9.2 Solution A vs Solution B — Recommendation

**BNRP recommends Solution B as the default primary name mechanism**, with Solution A as an opt-in extension for metadata-rich declarations.

**Rationale:**

| Criterion | Solution B (Default) | Solution A (Extension) |
|-----------|---------------------|----------------------|
| Simplicity | Self-transfer (`from == to`) — no new inscription needed | Requires creating a separate `primary-name` inscription |
| Proxy risk | Zero — self-transfer is inherently authenticated by the Bitcoin transaction signature | Requires verifying minter == owner; proxy inscribe services could create invalid claims |
| Cost | No additional inscription fee (just the transfer fee) | Additional inscription fee for the primary-name record |
| Metadata | None — name only | Supports avatar and display fields |
| Ecosystem support | Already specified with block height, max length, address types | Draft status with fewer implementation details |
| Immutability | Transfer events are immutable Bitcoin transactions | Inscription minter is immutable |

**Coexistence rules:**
1. Solution B is the primary mechanism. If a wallet has a valid Solution B primary name, it is used.
2. Solution A is checked as a fallback if no Solution B primary name exists.
3. If both exist for the same wallet, the one with the most recent block height wins ("Last is New" applies across both solutions).
4. Profile metadata from Solution A (avatar, display) is used only when Solution A is the active primary name.

### 9.3 Reverse Resolution Pseudocode

```pseudocode
function reverseResolve(address: string) -> string | null:
    // Step 1: Normalize the address
    normalizedAddress = normalizeAddress(address)
    //   For BTC addresses: no change (case-sensitive bech32)
    //   For EVM addresses: convert to checksummed form (EIP-55), then lowercase for comparison
    //   For SOL addresses: no change (base58 is case-sensitive)
    
    // Step 2: Query indexer for primary name of the address
    //   Check Solution B first: find the most recent self-transfer (from==to) for this address
    primaryNameB = indexer.getSolutionBPrimaryName(normalizedAddress)
    //   Check Solution A: find the most recent valid primary-name inscription minted by this address
    primaryNameA = indexer.getSolutionAPrimaryName(normalizedAddress)
    
    // Step 3: Determine which primary name is active
    claimedName = null
    if primaryNameB is not null and primaryNameA is not null:
        // Both exist — use the one with the most recent block height
        if primaryNameB.blockHeight >= primaryNameA.blockHeight:
            claimedName = primaryNameB.name
        else:
            claimedName = primaryNameA.name
    else if primaryNameB is not null:
        claimedName = primaryNameB.name
    else if primaryNameA is not null:
        claimedName = primaryNameA.name
    else:
        // No primary name claim exists for this address
        return null
    
    // Step 4: Validate name length
    if claimedName.length > 128:
        return null  // Exceeds primary name length limit
    
    // Step 5: Forward-resolve the claimed name
    forwardRecord = resolve(claimedName)
    if forwardRecord is Error:
        return null  // Name doesn't resolve — anti-spoofing rejection
    
    // Step 6: Verify the address appears in the forward resolution
    allAddresses = collectAllAddresses(forwardRecord)
    //   allAddresses includes: btc_taproot, btc_segwit, btc_legacy, btc_p2sh,
    //   eth, sol, and all addr_{coinType} values, plus the owner address
    allAddresses.add(forwardRecord.owner)
    
    // Normalize all collected addresses for comparison
    normalizedAllAddresses = allAddresses.map(a => normalizeAddress(a))
    
    if normalizedAddress in normalizedAllAddresses:
        return claimedName  // Anti-spoofing check passed
    else:
        return null  // Anti-spoofing rejection: address not in forward record
```

### 9.4 Edge Cases

**Name transferred away:**
If user A sets `alice.btc` as their primary name (via Solution B self-transfer), then transfers `alice.btc` to user B, user A's reverse resolution MUST fail on the next query because: (1) A is no longer the owner of the name inscription, and (2) the forward resolution of `alice.btc` will no longer include A's address (once B creates a new routing record). Until B creates a new routing record, A's old routing record may still be valid (owner-at-time-of-creation rule), but the anti-spoofing check verifies against the current owner address, which is now B.

**Proxy inscription (Solution A):**
If a proxy inscribe service creates a `primary-name` inscription on behalf of user A, the minter address will be the proxy service's address, not A's address. BNRP indexers MUST reject this inscription because the minter does not match the name owner. Solution B avoids this entirely because self-transfers cannot be proxied — the Bitcoin transaction must be signed by the key controlling the UTXO.

**Empty forward record:**
If `alice.btc` exists as a name inscription but has no routing record, the forward resolution returns only the owner address. Reverse resolution for the owner address against `alice.btc` will succeed because the owner address is always included in the anti-spoofing check (it is implicitly part of the address set).

**Multiple names owned:**
A wallet may own multiple `.btc` names. Only one can be the primary name at a time. The "Last is New" rule ensures that the most recent primary name action (self-transfer or primary-name inscription) determines the active primary name. Previous primary names become inactive but the inscriptions remain on-chain.

---

## 10. Avatar Rules

### 10.1 Avatar URI Schemes Supported

| Scheme | Format | Example | Resolution Method |
|--------|--------|---------|-------------------|
| `ord:` | `ord:{inscriptionId}` | `ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0` | Fetch content from Ordinals indexer API: `GET /content/{inscriptionId}` |
| `ord-num:` | `ord-num:{number}` | `ord-num:35648` | Resolve inscription number to ID via indexer, then fetch content. **Warning: unstable — number may change.** |
| `ipfs://` | `ipfs://{CID}` | `ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio` | Fetch via IPFS gateway: `https://ipfs.io/ipfs/{CID}` or local IPFS node |
| `ipns://` | `ipns://{key}` | `ipns://k51qzi5uqu5dlvj2baxnqndepeb86cbk3ng7n3i46uzyxzyqj2xjonzllnv0v8` | Resolve IPNS key to CID, then fetch via IPFS |
| `https://` | `https://{url}` | `https://example.com/avatars/alice.png` | Direct HTTP GET request |
| `data:` | `data:{mime};base64,{data}` | `data:image/png;base64,iVBORw0KGgo...` | Decode inline base64 data |

### 10.2 Ordinal Avatar Resolution Flow

```pseudocode
function resolveOrdinalAvatar(avatarURI: string) -> ImageResult | Error:
    // Step 1: Parse the inscription ID from the URI
    if not avatarURI.startsWith("ord:"):
        return Error("NOT_ORDINAL_URI")
    
    inscriptionId = avatarURI.substring(4)  // Remove "ord:" prefix
    
    // Step 2: Validate inscription ID format ({txid}i{index})
    if not matchesPattern(inscriptionId, "^[a-f0-9]{64}i[0-9]+$"):
        return Error("INVALID_INSCRIPTION_ID: must be {txid}i{index}")
    
    // Step 3: Query Ordinals indexer for inscription metadata
    inscriptionMeta = ordinalsIndexer.getInscription(inscriptionId)
    if inscriptionMeta is null:
        return Error("INSCRIPTION_NOT_FOUND: " + inscriptionId)
    
    // Step 4: Verify the inscription still exists (not pruned/cursed)
    if inscriptionMeta.status != "active":
        return Error("INSCRIPTION_INACTIVE: inscription may have been pruned")
    
    // Step 5: Check MIME type
    mimeType = inscriptionMeta.contentType
    allowedMIMETypes = [
        "image/jpeg", "image/png", "image/gif",
        "image/webp", "image/svg+xml"
    ]
    if mimeType not in allowedMIMETypes:
        return Error("INVALID_MIME_TYPE: " + mimeType + " is not a supported image type")
    
    // Step 6: If SVG, flag for sanitization
    requiresSanitization = (mimeType == "image/svg+xml")
    
    // Step 7: Construct the content URL
    contentURL = ordinalsIndexer.baseURL + "/content/" + inscriptionId
    
    // Step 8: Return the result
    return ImageResult {
        url: contentURL,
        mimeType: mimeType,
        inscriptionId: inscriptionId,
        requiresSanitization: requiresSanitization,
        ownerAddress: inscriptionMeta.ownerAddress
    }
```

### 10.3 Avatar Ownership Verification

For Ordinal avatars (using the `ord:` scheme), BNRP requires ownership verification: the inscription used as an avatar MUST be owned by the same address that owns the `.btc` name inscription. This prevents users from displaying someone else's ordinal NFT as their avatar.

**Verification procedure:**

```pseudocode
function verifyAvatarOwnership(nameOwner: string, avatarURI: string) -> VerificationResult:
    if not avatarURI.startsWith("ord:"):
        // Non-ordinal avatars don't require ownership verification
        return VerificationResult { verified: true, type: "non-ordinal" }
    
    inscriptionId = avatarURI.substring(4)
    avatarOwner = indexer.getInscriptionOwner(inscriptionId)
    
    if avatarOwner is null:
        return VerificationResult { verified: false, reason: "avatar inscription not found" }
    
    if avatarOwner == nameOwner:
        return VerificationResult { verified: true, type: "owner-match" }
    else:
        return VerificationResult { verified: false, reason: "owner mismatch",
                                     nameOwner: nameOwner, avatarOwner: avatarOwner }
```

**Exception — Unverified Avatars:**

A profile or routing inscription can declare `"avatar_unverified": true` to use an ordinal avatar without ownership verification. Wallets and applications displaying such avatars MUST show a visible "Unverified" badge or warning. This allows users to reference well-known ordinals (e.g., a famous collection piece) as aspirational avatars while maintaining transparency.

### 10.4 Fallback Chain

When an avatar URI fails to resolve (inscription moved, wrong MIME type, HTTP error, IPFS timeout), the following fallback chain is applied in order:

1. **Primary avatar URI**: Attempt to resolve the `avatar` field from the most recent profile or routing inscription.
2. **`ord:` scheme**: If the primary URI is `ord:` and fails, check if the inscription has been transferred. If a new owner holds it, fail this step.
3. **`ipfs://` fallback**: If an `ipfs://` avatar was previously set in any historical profile/routing inscription for this name, attempt to resolve it.
4. **`https://` fallback**: If an `https://` avatar was previously set, attempt to fetch it.
5. **Initials avatar**: Generate a deterministic avatar from the first character of the name label. For example, `alice.btc` → an avatar containing the letter "A" on a background color derived from a hash of the full name: `backgroundColor = '#' + sha256(name).substring(0, 6)`.
6. **Placeholder**: Display a generic placeholder image (e.g., a Bitcoin logo silhouette).

Applications SHOULD cache successful avatar resolutions with a TTL of 10 minutes and SHOULD NOT attempt fallback on every request. A failed resolution SHOULD be cached for 1 minute before retrying.

### 10.5 Avatar Storage Recommendation

- **On-chain (preferred)**: The `avatar` field in a routing or profile inscription stores the avatar URI directly on Bitcoin. This is the most durable and censorship-resistant option. Cost: one additional field in an existing inscription.
- **Off-chain signed (optional)**: For users who update avatars frequently, an off-chain signed avatar record can be used. The record is a JSON object containing the `avatar` URI, signed by the owner's taproot key. The signature is stored in the `avatar_proof` field. Resolvers verify the signature against the name owner's key before accepting the off-chain avatar.

**Avatar field placement:**
- Canonical field name: `avatar` (in routing, profile, or primary-name inscriptions)
- Proof field (optional): `avatar_proof` — contains a base64-encoded Schnorr signature of the string `"bnrp-avatar:" + avatar_uri` signed by the owner's taproot internal key.

**Signature verification:**
```pseudocode
function verifyAvatarProof(ownerTaprootAddress: string, avatarURI: string, proof: string) -> boolean:
    message = "bnrp-avatar:" + avatarURI
    messageHash = sha256(sha256("Bitcoin Signed Message:\n" + message))
    publicKey = extractTaprootInternalKey(ownerTaprootAddress)
    signature = base64Decode(proof)
    return schnorrVerify(publicKey, messageHash, signature)
```

### 10.6 Image Safety and Display Recommendations

**Supported MIME types:**

| MIME Type | Notes |
|-----------|-------|
| `image/jpeg` | Standard photographic format. No security concerns. |
| `image/png` | Standard raster format. No security concerns. |
| `image/gif` | Animated GIFs supported. Cap animation frames at 300 for display. |
| `image/webp` | Modern web format. Supported in all current browsers. |
| `image/svg+xml` | **Must be sanitized** — strip `<script>`, `<foreignObject>`, event handlers (`onload`, `onclick`, etc.), `javascript:` URIs, and external resource references. |

**Rejected types:** `text/html`, `application/javascript`, `application/octet-stream`, and any executable content types.

**Display recommendations:**
- Minimum source resolution: 400×400 pixels (smaller images should be accepted but flagged as low-quality).
- Display aspect ratio: 1:1 (square crop from center if non-square).
- Compact view (wallet address bar): 32×32px rendered size.
- Profile view: 128×128px rendered size.
- Full view: 400×400px rendered size.
- Maximum file size: 5MB. Larger images SHOULD be rejected or downsampled by the resolver/gateway.

**Content Security Policy:**
- Avatars SHOULD be served from a sandboxed origin (not the same origin as the application).
- For web-based applications: `Content-Security-Policy: sandbox; default-src 'none'; img-src data: blob:;`
- SVG avatars MUST be sanitized server-side before rendering. Use an SVG sanitizer that removes all active content, scripts, and external references.
- Data URIs in SVG `<image>` elements MUST be validated to ensure they contain only image data.

### 10.7 Full Example: alice.btc with Ordinal Avatar

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
  "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "sol": "7EcDhSYGxXyscszYEp35KHN8vvw3svAuLKTzXwCFLtV",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "btc_lightning": "alice@getalby.com"
}
```

Resolution: The resolver reads the `avatar` field, extracts inscription ID `9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0`, verifies that the inscription owner matches the name owner (`bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297`), fetches the inscription content from the Ordinals indexer, verifies the MIME type is `image/*`, and returns the image URL.

### 10.8 Full Example: alice.btc with IPFS Fallback Avatar

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
  "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "avatar": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "avatar_unverified": true,
  "btc_lightning": "alice@getalby.com"
}
```

Resolution: The resolver reads the `avatar` field, recognizes the `ipfs://` scheme, constructs the gateway URL `https://ipfs.io/ipfs/QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio`, and returns it. Because `avatar_unverified` is `true`, wallets display the avatar with an "Unverified" badge — no ownership check is performed for IPFS avatars.

---

## 11. Web/Content Resolution

### 11.1 The Challenge

`.btc` names do not work natively in web browsers because `.btc` is not a registered ICANN top-level domain. When a user types `alice.btc` into a browser's address bar, the browser attempts a DNS lookup, fails to find a DNS record, and either shows an error or redirects to a search engine. This is the same challenge ENS faced with `.eth` domains, and ENS solved it through a combination of browser extensions, HTTP gateways (eth.limo), and eventually native browser integration (Brave, Opera).

BNRP must bridge this gap to enable `.btc` names to serve as decentralized web identities. The options and their tradeoffs are:

1. **Browser extension**: Intercepts `.btc` navigation, resolves via BNRP, and serves content. Requires installation — limited to power users.
2. **HTTP gateway**: Proxy service that maps `{name}.btc.{gateway}/` to the name's content record. Works in any browser but depends on the gateway operator.
3. **DNS bridge**: Maps `.btc` names to DNS records via a centralized bridge. Works universally but is centralized and censorable.
4. **Native browser support**: Browser directly supports `.btc` TLD resolution. Requires browser vendor cooperation — long-term goal.

### 11.2 Content Record Format

The `content` field in routing or content inscriptions supports the following URI schemes:

| Scheme | Format | Example | Use Case |
|--------|--------|---------|----------|
| `ipfs://` | `ipfs://{CIDv0 or CIDv1}` | `ipfs://QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn` | Static websites, immutable content |
| `ipns://` | `ipns://{IPNS key or DNSLink}` | `ipns://k51qzi5uqu5dlvj2baxnqndepeb86cbk3ng7n3i46uzyxzyqj2xjonzllnv0v8` | Dynamic websites, mutable content via IPNS |
| `https://` | `https://{domain}/{path}` | `https://alice-btc.netlify.app` | Traditional web hosting |
| `ar://` | `ar://{transaction ID}` | `ar://bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U` | Arweave permanent storage |
| `ord://` | `ord://{inscriptionId}` | `ord://abc123def456...i0` | Ordinals HTML inscription served directly |

**Content type hints:** The optional `content_type` field in the content record schema provides a hint to gateways:
- `"website"` — serves as a full website (expects HTML index)
- `"dapp"` — serves as a decentralized application (expects JavaScript bundle)
- `"file"` — serves a single file (image, PDF, etc.)
- `"redirect"` — HTTP 302 redirect to the content URL

### 11.3 Resolution Methods Comparison

| Method | Works Today | Requires Extension | Decentralized | Censorship Resistant | Setup Complexity | Recommended For |
|--------|------------|-------------------|---------------|---------------------|-----------------|-----------------|
| Browser Extension | Yes | Yes | Yes | Yes | Medium (install ext) | Power users, developers |
| HTTP Gateway (`alice.btc.gateway.io`) | Yes | No | Partial (gateway is centralized) | Partial (gateway can be censored) | None for user | Mass adoption, MVP |
| DNS Bridge | Yes | No | No | No | DNS config required | Corporate/enterprise use |
| IPFS/IPNS contenthash | Yes (with IPFS companion) | Yes | Yes | Yes | Medium (IPFS node) | Decentralization maximalists |
| `ord://` native | Future | Yes (requires Ordinals browser) | Yes | Yes (data on Bitcoin) | Not available yet | Bitcoin-native applications |

### 11.4 Gateway Resolution (Recommended for MVP)

**Pattern:** `https://{name}.btc.{gateway-domain}/`

**Example:** `https://alice.btc.bnrp.io/`

**Gateway Resolution Flow:**

```pseudocode
function gatewayResolve(httpRequest: Request) -> Response:
    // Step 1: Extract the name from the subdomain
    // Request: GET https://alice.btc.bnrp.io/about
    host = httpRequest.headers["Host"]  // "alice.btc.bnrp.io"
    gatewayDomain = "bnrp.io"
    namePart = host.replace("." + gatewayDomain, "")  // "alice.btc"
    path = httpRequest.path  // "/about"
    
    // Step 2: Normalize and resolve the name
    record = resolve(namePart)
    if record is Error:
        return Response(404, "Name not found: " + namePart)
    
    // Step 3: Get the content URI
    contentURI = record.content
    if contentURI is null:
        return Response(404, "No content record for: " + namePart)
    
    // Step 4: Fetch content based on URI scheme
    if contentURI.startsWith("ipfs://"):
        cid = contentURI.substring(7)
        content = ipfsGateway.fetch(cid + path)
    else if contentURI.startsWith("ipns://"):
        key = contentURI.substring(7)
        content = ipnsGateway.resolve(key, path)
    else if contentURI.startsWith("https://"):
        content = httpProxy.fetch(contentURI + path)
    else if contentURI.startsWith("ar://"):
        txid = contentURI.substring(5)
        content = arweaveGateway.fetch(txid + path)
    else if contentURI.startsWith("ord://"):
        inscriptionId = contentURI.substring(6)
        content = ordinalsIndexer.getContent(inscriptionId)
    else:
        return Response(400, "Unsupported content scheme")
    
    // Step 5: Set security headers and return
    response = Response(200, content.body)
    response.headers["Content-Type"] = content.contentType
    response.headers["X-BNRP-Name"] = namePart
    response.headers["X-BNRP-Content-Source"] = contentURI
    response.headers["Content-Security-Policy"] = "default-src 'self' 'unsafe-inline' data: blob: ipfs: https:;"
    response.headers["X-Frame-Options"] = "SAMEORIGIN"
    return response
```

**Open Gateway Specification:**
- Any entity can operate a BNRP gateway by implementing the resolution flow above.
- Gateways MUST resolve names via a BNRP-compliant resolver (local or remote).
- Gateways MUST support at least `ipfs://` and `https://` content schemes.
- Gateways SHOULD support `ar://` and `ord://` schemes.
- Gateways MUST set appropriate security headers (CSP, X-Frame-Options).
- Gateways SHOULD provide a landing page at the root domain explaining the service.
- Multiple gateways can operate for redundancy: `alice.btc.bnrp.io`, `alice.btc.ordinals.com`, `alice.btc.btcname.id`.

### 11.5 Browser Extension Resolution

A BNRP browser extension operates as follows:

1. The extension registers a handler for navigation events to URLs matching `*.btc`, `*.sats`, `*.unisat`, `*.x`, and `*.fb`.
2. When a user navigates to `alice.btc`:
    - The extension intercepts the navigation before DNS lookup.
    - It calls the BNRP resolver API: `GET /resolve/alice.btc`.
    - If a `content` field exists, the extension either:
        a. Redirects to a configured gateway: `https://alice.btc.bnrp.io/`
        b. Fetches IPFS content directly (if the user has an IPFS node or the extension includes a light IPFS client)
        c. Renders the `ord://` inscription content inline
    - If no `content` field exists, the extension shows a profile card with the name's avatar, addresses, and profile metadata.
3. The extension provides a popup showing the resolved profile for any `.btc` name.
4. The extension caches resolution results with a 10-minute TTL.

### 11.6 What is Realistic Now vs Long-Term

**Now (MVP — achievable in 6 months):**
- HTTP gateway resolution (`alice.btc.bnrp.io`)
- Browser extension (Chrome, Firefox, Brave) with gateway redirect
- IPFS content support via public IPFS gateways
- Basic `ord://` support for HTML inscriptions

**Long-term (12-24 months):**
- Native `.btc` TLD support in Brave browser (Brave already supports `.eth` via ENS)
- Decentralized gateway network with resolver diversity
- IPFS content serving directly in browser extension (no external gateway)
- Opera, Edge, and Safari integration discussions
- `ord://` content rendering with full Ordinals browser capabilities

---

## 12. Bitmap / Metaverse Extension

### 12.1 Bitmap Background

The Bitmap protocol is a convention on Bitcoin Ordinals where every Bitcoin block can be claimed as a "district" in an open metaverse. A user inscribes `{blockheight}.bitmap` to claim ownership of that block's district. Within each district, individual transactions become "parcels" — addressable locations within the block. The protocol follows a first-come-first-served principle: the first valid inscription of `{blockheight}.bitmap` claims that district.

The Bitmap ecosystem includes metaverse explorers (Atlas), parcel extraction tools, and various applications that render Bitcoin blocks as navigable 3D/2D spaces. Parent-child inscription relationships enable districts (parent inscriptions) to contain parcels (child inscriptions) as a hierarchical data structure.

### 12.2 .btc to .bitmap Mapping (6-Digit Rule)

An existing convention maps 6-digit `.btc` names to matching bitmap district numbers:

| .btc Name | Bitmap District | Mapping Rule |
|-----------|----------------|--------------|
| `840000.btc` | `840000.bitmap` | Numeric-only .btc names with 1-7 digits map to the bitmap district with the same block height |
| `100000.btc` | `100000.bitmap` | Same rule applies |
| `999999.btc` | `999999.bitmap` | Same rule applies |
| `alice.btc` | (no auto-mapping) | Non-numeric names require an explicit bitmap record |

**BNRP formalizes this mapping:**
- A `.btc` name consisting entirely of digits (matching `^[0-9]{1,7}\.btc$`) automatically resolves to the bitmap district with the corresponding block height, provided the block exists.
- For non-numeric `.btc` names, the bitmap mapping MUST be established via an explicit `bitmap` record (either in the routing inscription or a dedicated bitmap inscription).
- The auto-mapping is read-only — it does not establish ownership of the bitmap inscription. The `.btc` name owner is not necessarily the `.bitmap` inscription owner.

### 12.3 Bitmap Record Fields

```json
{
  "p": "bnrp",
  "v": "1",
  "op": "bitmap",
  "name": "alice.btc",
  "bitmap_district": "840000",
  "bitmap_parcel": "42",
  "bitmap_world": "atlas_prime",
  "bitmap_coordinates": "x:100,y:200",
  "bitmap_avatar": "ord:a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1i0",
  "bitmap_profile_url": "https://atlas.btc/alice",
  "bitmap_inscription_id": "15cd2a9189681a581e4cc3fed42d9bc7ec7a148e2fcb11981c8045522a793ef1i0"
}
```

**Field definitions:**
- `bitmap_district`: The Bitcoin block height representing the district. MUST be a valid block height (integer, ≤ current chain tip).
- `bitmap_parcel`: The transaction index within the block, identifying a specific parcel. `"0"` is the coinbase transaction.
- `bitmap_world`: An identifier for the metaverse world or realm in which this bitmap presence exists (e.g., `"atlas_prime"`, `"bitmap_earth"`).
- `bitmap_coordinates`: A coordinate pair for positioning within the world. Format: `"x:{int},y:{int}"`.
- `bitmap_avatar`: A separate avatar URI for the bitmap/metaverse presence, which may differ from the main profile avatar.
- `bitmap_profile_url`: An external URL for the user's bitmap profile page.
- `bitmap_inscription_id`: The inscription ID of the actual `.bitmap` inscription, if the user owns one.

### 12.4 Resolution: name.btc → bitmap

```pseudocode
function resolveBitmap(name: string) -> BNRPBitmapRecord | null:
    normalizedName = normalize(name)
    if normalizedName is Error:
        return null
    
    // Check for automatic numeric mapping
    label = normalizedName.split(".")[0]
    tld = normalizedName.split(".")[1]
    
    if tld == "btc" and isNumeric(label) and label.length <= 7:
        blockHeight = parseInt(label)
        // Verify block exists
        if blockHeight <= bitcoin.getBlockHeight():
            autoRecord = BNRPBitmapRecord {
                name: normalizedName,
                bitmap_district: label,
                bitmap_parcel: null,
                bitmap_world: null,
                bitmap_coordinates: null,
                bitmap_avatar: null,
                bitmap_profile_url: null,
                bitmap_inscription_id: null,
                auto_mapped: true
            }
            // Check if an explicit bitmap record also exists (overrides auto)
            explicitRecord = fetchExplicitBitmapRecord(normalizedName)
            if explicitRecord is not null:
                return mergeRecords(autoRecord, explicitRecord)
            return autoRecord
    
    // For non-numeric names, look for explicit bitmap record
    // Check dedicated bitmap inscription first
    bitmapInscription = fetchLatestRecord(normalizedName, getNameOwner(normalizedName), "bitmap")
    if bitmapInscription is not null:
        return parseBitmapRecord(bitmapInscription)
    
    // Fall back to bitmap fields in routing inscription
    routingRecord = resolve(normalizedName)
    if routingRecord is not Error and routingRecord.bitmap is not null:
        return routingRecord.bitmap
    
    return null
```

---

## 13. Resolver Architecture

### 13.1 Open Resolver Specification

A compliant BNRP resolver MUST implement the following:

**Required endpoints:**

| Endpoint | Method | Required | Description |
|----------|--------|----------|-------------|
| `GET /v1/resolve/{name}` | GET | Yes | Full record resolution |
| `GET /v1/reverse/{address}` | GET | Yes | Reverse name lookup |
| `GET /v1/address/{name}?chain={coinType}` | GET | Yes | Single-chain address lookup |
| `GET /v1/health` | GET | Yes | Health check |
| `GET /v1/profile/{name}` | GET | Recommended | Profile metadata |
| `GET /v1/avatar/{name}` | GET | Recommended | Avatar URL resolution |
| `GET /v1/content/{name}` | GET | Recommended | Content/web resolution |
| `GET /v1/bitmap/{name}` | GET | Optional | Bitmap/metaverse lookup |

**Required behavior:**
1. All name inputs MUST be normalized per §8.1 before processing.
2. Resolution MUST respect the 3-confirmation threshold.
3. The "Last is New" rule MUST be applied for record ordering.
4. Reverse resolution MUST apply bidirectional anti-spoofing verification (§9.1).
5. Responses MUST include `X-BNRP-Block-Height` header indicating the chain tip at resolution time.
6. Responses MUST include a `Cache-Control` header with appropriate TTL (default: `max-age=600`).
7. Error responses MUST use standard HTTP status codes and include a JSON error body with `code` and `message` fields.
8. The resolver MUST index both `"p": "btcname"` and `"p": "bnrp"` inscriptions.

**Optional capabilities:**
- Multi-resolver quorum (query multiple upstream resolvers and require agreement).
- Signed responses (resolver signs response with a known public key).
- WebSocket subscriptions for real-time name update notifications.
- GraphQL endpoint as an alternative to REST.
- Batch resolution (resolve multiple names in a single request).

### 13.2 Resolver Discovery

**Known public resolvers (bootstrap list):**
Applications SHOULD ship with a hardcoded list of known public resolvers:

```json
{
  "resolvers": [
    {
      "url": "https://resolver.bnrp.org/v1",
      "operator": "BNRP Foundation",
      "status": "active"
    },
    {
      "url": "https://api.btcname.id/v1",
      "operator": "BtcName",
      "status": "active"
    },
    {
      "url": "https://bnrp.unisat.io/v1",
      "operator": "UniSat",
      "status": "planned"
    }
  ]
}
```

**DNS-based discovery:**
Organizations can advertise their BNRP resolver via DNS TXT records:

```
_bnrp.example.com  TXT  "v=bnrp1; url=https://resolver.example.com/v1; pubkey=02abc..."
```

Fields:
- `v=bnrp1` — protocol version identifier
- `url=` — resolver base URL
- `pubkey=` — (optional) resolver's public key for signed responses

**On-chain resolver registry (future):**
A future BNRP-IP will define an on-chain resolver registry — an Ordinals inscription that maintains a list of active resolvers. This inscription would be updated by the BNRP governance process and provides a decentralized, censorship-resistant resolver discovery mechanism.

### 13.3 Full Resolution Flow Pseudocode

```pseudocode
// ─────────────────────────────────────────────────
// Forward resolution: resolve(name) → BNRPRecord
// ─────────────────────────────────────────────────
function resolve(name: string) -> BNRPRecord | Error:
    normalizedName = normalize(name)
    if normalizedName is Error:
        return Error("NORMALIZATION_FAILED", normalizedName.message)
    
    nameInscription = indexer.getNameInscription(normalizedName)
    if nameInscription is null:
        return Error("NAME_NOT_FOUND", "No inscription for " + normalizedName)
    
    if not hasMinConfirmations(nameInscription, 3):
        return Error("UNCONFIRMED", "Inscription has insufficient confirmations")
    
    owner = indexer.getInscriptionOwner(nameInscription.inscriptionId)
    if owner is null:
        return Error("OWNER_UNKNOWN", "Cannot determine owner")
    
    routing = indexer.getLatestValidRecord(normalizedName, "routing", owner)
    profile = indexer.getLatestValidRecord(normalizedName, "profile", owner)
    content = indexer.getLatestValidRecord(normalizedName, "content", owner)
    bitmap  = indexer.getLatestValidRecord(normalizedName, "bitmap", owner)
    
    addresses = routing ? parseAddresses(routing) : {}
    avatarURI = profile?.avatar ?? routing?.avatar ?? null
    contentURI = content?.content ?? routing?.content ?? null
    
    return BNRPRecord {
        name: normalizedName, owner: owner,
        inscriptionId: nameInscription.inscriptionId,
        addresses: addresses, profile: profile,
        avatar: avatarURI, content: contentURI,
        bitmap: bitmap, resolvedAtBlock: currentBlockHeight(), ttl: 600
    }

// ─────────────────────────────────────────────────
// Reverse resolution: reverseResolve(address) → name
// ─────────────────────────────────────────────────
function reverseResolve(address: string) -> string | null:
    normalizedAddr = normalizeAddress(address)
    
    // Check Solution B then Solution A, use most recent
    primaryB = indexer.getSolutionBPrimaryName(normalizedAddr)
    primaryA = indexer.getSolutionAPrimaryName(normalizedAddr)
    
    claimed = selectMostRecent(primaryB, primaryA)
    if claimed is null:
        return null
    
    if claimed.name.length > 128:
        return null
    
    // Anti-spoofing: forward resolve and verify
    forward = resolve(claimed.name)
    if forward is Error:
        return null
    
    allAddrs = getAllAddresses(forward)
    allAddrs.add(forward.owner)
    
    if normalizedAddr in allAddrs.map(normalizeAddress):
        return claimed.name
    
    return null

// ─────────────────────────────────────────────────
// Avatar resolution: resolveAvatar(name) → imageURL
// ─────────────────────────────────────────────────
function resolveAvatar(name: string) -> string | null:
    record = resolve(name)
    if record is Error or record.avatar is null:
        return generateInitialsAvatar(name)
    
    avatarURI = record.avatar
    
    if avatarURI.startsWith("ord:"):
        result = resolveOrdinalAvatar(avatarURI)
        if result is not Error:
            if not record.avatar_unverified:
                ownership = verifyAvatarOwnership(record.owner, avatarURI)
                if not ownership.verified:
                    return generateInitialsAvatar(name)  // ownership failed
            return result.url
        // Ordinal resolution failed — try fallback
    
    if avatarURI.startsWith("ipfs://"):
        cid = avatarURI.substring(7)
        return "https://ipfs.io/ipfs/" + cid
    
    if avatarURI.startsWith("ipns://"):
        key = avatarURI.substring(7)
        return "https://ipfs.io/ipns/" + key
    
    if avatarURI.startsWith("https://"):
        return avatarURI
    
    if avatarURI.startsWith("data:"):
        return avatarURI  // inline data URI
    
    return generateInitialsAvatar(name)

// ─────────────────────────────────────────────────
// Content resolution: resolveContent(name) → URL
// ─────────────────────────────────────────────────
function resolveContent(name: string) -> string | null:
    record = resolve(name)
    if record is Error or record.content is null:
        return null
    
    contentURI = record.content
    
    if contentURI.startsWith("ipfs://"):
        cid = contentURI.substring(7)
        return "https://ipfs.io/ipfs/" + cid
    else if contentURI.startsWith("ipns://"):
        key = contentURI.substring(7)
        return "https://ipfs.io/ipns/" + key
    else if contentURI.startsWith("https://"):
        return contentURI
    else if contentURI.startsWith("ar://"):
        txid = contentURI.substring(5)
        return "https://arweave.net/" + txid
    else if contentURI.startsWith("ord://"):
        inscriptionId = contentURI.substring(6)
        return "https://ordinals.com/content/" + inscriptionId
    
    return null

// ─────────────────────────────────────────────────
// Bitmap resolution: resolveBitmap(name) → record
// ─────────────────────────────────────────────────
function resolveBitmap(name: string) -> BNRPBitmapRecord | null:
    normalizedName = normalize(name)
    if normalizedName is Error:
        return null
    
    label = normalizedName.split(".")[0]
    tld = normalizedName.split(".")[1]
    
    // Automatic numeric mapping for .btc
    autoDistrict = null
    if tld == "btc" and isAllDigits(label) and label.length <= 7:
        blockHeight = parseInt(label)
        if blockHeight <= bitcoin.getBlockHeight():
            autoDistrict = label
    
    // Explicit bitmap record takes precedence
    record = resolve(normalizedName)
    if record is not Error and record.bitmap is not null:
        return record.bitmap
    
    if autoDistrict is not null:
        return BNRPBitmapRecord {
            bitmap_district: autoDistrict, auto_mapped: true
        }
    
    return null
```

### 13.4 Canonicalization and Normalization Rules

The following normalization rules are applied before any resolution operation. These rules are deterministic — any conforming implementation MUST produce identical outputs for identical inputs.

1. **Name normalization**: Per §8.1, applied to all name inputs.
2. **Address normalization for comparison**:
   - Bitcoin bech32/bech32m addresses: lowercase (bech32 is case-insensitive by spec, but canonical form is lowercase).
   - Bitcoin base58 addresses: case-sensitive (no transformation).
   - Ethereum/EVM addresses: lowercase for comparison (EIP-55 checksum is case-based but comparison is case-insensitive).
   - Solana addresses: case-sensitive base58 (no transformation).
3. **Inscription ID normalization**: Lowercase hexadecimal for the txid portion. The `i` separator and index are preserved. Example: `9BA5066E...I0` → `9ba5066e...i0`.
4. **JSON parsing**: All inscription content is parsed as UTF-8 JSON. BOM (byte order mark) is stripped. Trailing commas are rejected. Duplicate keys: the last value wins.
5. **Whitespace**: Leading and trailing whitespace in all string field values is stripped.
6. **Empty strings**: Fields with empty string values (`""`) are treated as absent.
7. **Protocol prefix normalization**: `"btcname"` is mapped to `"bnrp"` internally for unified processing. Legacy field names (e.g., `eth_address`) are mapped to BNRP field names (e.g., `eth`).

---

## 14. Developer APIs / SDK

### 14.1 REST API Specification

**Base URL:** `https://{resolver}/v1/`

**Common Headers (all responses):**

| Header | Value | Description |
|--------|-------|-------------|
| `Content-Type` | `application/json` | Response body format |
| `X-BNRP-Block-Height` | `{number}` | Chain tip block height at resolution time |
| `X-BNRP-Version` | `1` | BNRP protocol version |
| `Cache-Control` | `max-age=600` | Default 10-minute cache TTL |

**Error Response Format:**

```json
{
  "error": {
    "code": "NAME_NOT_FOUND",
    "message": "No inscription found for alice.btc"
  }
}
```

---

**1. `GET /v1/resolve/{name}` — Full Record Resolution**

Returns the complete BNRP record for a name including all addresses, profile, content, bitmap, and avatar data.

**Parameters:**
- `name` (path, required): The domain name to resolve (e.g., `alice.btc`)

**Example Request:** `GET /v1/resolve/alice.btc`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "owner": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "inscriptionId": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2i0",
  "addresses": {
    "btc_taproot": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
    "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
    "btc_lightning": "alice@getalby.com",
    "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "sol": "7EcDhSYGxXyscszYEp35KHN8vvw3svAuLKTzXwCFLtV",
    "addr_2155872277": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "addr_2189526273": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "addr_2147483785": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "profile": {
    "display": "Alice.btc",
    "description": "Bitcoin maximalist, Ordinals collector.",
    "url": "https://alice.btc.example.com",
    "email": "alice@protonmail.com",
    "com.twitter": "alice_btc",
    "com.github": "alice-btc-dev",
    "com.nostr": "npub1qe3e5wrvnsgpggtkytxteaqfprz0rgxr8c3l34kk3a9t7e2l3ach5xfhz",
    "location": "Cyberspace",
    "primary_contact": "nostr"
  },
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "avatarVerified": true,
  "content": "ipfs://QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn",
  "bitmap": {
    "bitmap_district": "840000",
    "bitmap_parcel": "42",
    "bitmap_world": "atlas_prime"
  },
  "resolvedAtBlock": 899500,
  "ttl": 600
}
```

---

**2. `GET /v1/reverse/{address}` — Reverse Name Lookup**

Returns the primary name for a given blockchain address, after anti-spoofing verification.

**Parameters:**
- `address` (path, required): The blockchain address (BTC, ETH, or SOL)

**Example Request:** `GET /v1/reverse/bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297`

**Example Response (200 OK):**

```json
{
  "address": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "primaryName": "alice.btc",
  "method": "solution_b",
  "verified": true,
  "resolvedAtBlock": 899500
}
```

**Example Response (404 — no primary name):**

```json
{
  "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "primaryName": null,
  "method": null,
  "verified": false,
  "resolvedAtBlock": 899500
}
```

---

**3. `GET /v1/address/{name}?chain={coinType}` — Single Chain Address**

Returns a single address for the specified chain.

**Parameters:**
- `name` (path, required): The domain name
- `chain` (query, required): SLIP-44 coin type (0 for BTC, 60 for ETH, 501 for SOL, etc.)
- `subtype` (query, optional): For BTC (coinType 0): `taproot`, `segwit`, `legacy`, `p2sh`, `lightning`. Defaults to `taproot`.

**Example Request:** `GET /v1/address/alice.btc?chain=60`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "chain": 60,
  "chainName": "Ethereum",
  "address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "resolvedAtBlock": 899500
}
```

**Example Request:** `GET /v1/address/alice.btc?chain=0&subtype=lightning`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "chain": 0,
  "chainName": "Bitcoin",
  "subtype": "lightning",
  "address": "alice@getalby.com",
  "resolvedAtBlock": 899500
}
```

---

**4. `GET /v1/profile/{name}` — Profile Fields Only**

**Example Request:** `GET /v1/profile/alice.btc`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "display": "Alice.btc",
  "description": "Bitcoin maximalist, Ordinals collector.",
  "url": "https://alice.btc.example.com",
  "email": "alice@protonmail.com",
  "com.twitter": "alice_btc",
  "com.github": "alice-btc-dev",
  "com.discord": "alice.btc",
  "com.nostr": "npub1qe3e5wrvnsgpggtkytxteaqfprz0rgxr8c3l34kk3a9t7e2l3ach5xfhz",
  "com.telegram": "alice_btc",
  "theme": "#F7931A",
  "location": "Cyberspace",
  "primary_contact": "nostr",
  "keywords": "bitcoin,ordinals,btcname,identity,decentralized",
  "resolvedAtBlock": 899500
}
```

---

**5. `GET /v1/avatar/{name}` — Avatar URL**

Returns the resolved avatar image URL (after resolving `ord:`, `ipfs://`, etc. schemes to an HTTP-fetchable URL) and verification status.

**Example Request:** `GET /v1/avatar/alice.btc`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "avatarURI": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "resolvedURL": "https://ordinals.com/content/9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "mimeType": "image/png",
  "verified": true,
  "resolvedAtBlock": 899500
}
```

---

**6. `GET /v1/content/{name}` — Web/Content Target**

**Example Request:** `GET /v1/content/alice.btc`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "contentURI": "ipfs://QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn",
  "resolvedURL": "https://ipfs.io/ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn",
  "contentType": "website",
  "resolvedAtBlock": 899500
}
```

---

**7. `GET /v1/bitmap/{name}` — Bitmap/Metaverse Record**

**Example Request:** `GET /v1/bitmap/alice.btc`

**Example Response (200 OK):**

```json
{
  "name": "alice.btc",
  "bitmap_district": "840000",
  "bitmap_parcel": "42",
  "bitmap_world": "atlas_prime",
  "bitmap_coordinates": "x:100,y:200",
  "bitmap_avatar": "ord:a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1i0",
  "bitmap_profile_url": "https://atlas.btc/alice",
  "auto_mapped": false,
  "resolvedAtBlock": 899500
}
```

---

**8. `GET /v1/health` — Resolver Health Check**

**Example Request:** `GET /v1/health`

**Example Response (200 OK):**

```json
{
  "status": "healthy",
  "version": "1.0.0",
  "bnrpVersion": "1",
  "chainTip": 899500,
  "indexerSyncedTo": 899500,
  "indexerLag": 0,
  "protocols": ["btcname", "bnrp"],
  "supportedTLDs": ["btc", "sats", "unisat", "x", "fb"],
  "uptime": 864000,
  "timestamp": "2026-03-26T16:00:00Z"
}
```

### 14.2 SDK Interface (TypeScript/JavaScript)

```typescript
// ─────────────────────────────────────────────────
// BNRP SDK Type Definitions
// ─────────────────────────────────────────────────

/** SLIP-44 coin type constants */
export enum CoinType {
  BTC = 0,
  ETH = 60,
  SOL = 501,
  MATIC = 2147483785,    // 137 | 0x80000000
  BASE = 2155872277,     // 8453 | 0x80000000
  ARBITRUM = 2189526273, // 42161 | 0x80000000
}

/** Bitcoin address subtypes (all use CoinType.BTC = 0) */
export type BTCSubtype = 'taproot' | 'segwit' | 'legacy' | 'p2sh' | 'lightning';

/** Address record map */
export interface BNRPAddresses {
  btc_taproot?: string;
  btc_segwit?: string;
  btc_legacy?: string;
  btc_p2sh?: string;
  btc_lightning?: string;
  eth?: string;
  sol?: string;
  [key: `addr_${number}`]: string | undefined;
}

/** Profile metadata */
export interface BNRPProfile {
  display?: string;
  description?: string;
  url?: string;
  email?: string;
  'com.twitter'?: string;
  'com.github'?: string;
  'com.discord'?: string;
  'com.nostr'?: string;
  'com.telegram'?: string;
  avatar?: string;
  theme?: string;
  location?: string;
  primary_contact?: string;
  keywords?: string;
}

/** Bitmap/metaverse record */
export interface BNRPBitmapRecord {
  bitmap_district: string;
  bitmap_parcel?: string;
  bitmap_world?: string;
  bitmap_coordinates?: string;
  bitmap_avatar?: string;
  bitmap_profile_url?: string;
  bitmap_inscription_id?: string;
  auto_mapped?: boolean;
}

/** Complete resolution record */
export interface BNRPRecord {
  name: string;
  owner: string;
  inscriptionId: string;
  addresses: BNRPAddresses;
  profile: BNRPProfile | null;
  avatar: string | null;
  avatarVerified: boolean;
  content: string | null;
  bitmap: BNRPBitmapRecord | null;
  resolvedAtBlock: number;
  ttl: number;
}

/** Reverse resolution result */
export interface BNRPReverseResult {
  address: string;
  primaryName: string | null;
  method: 'solution_a' | 'solution_b' | null;
  verified: boolean;
  resolvedAtBlock: number;
}

/** Avatar resolution result */
export interface BNRPAvatarResult {
  avatarURI: string;
  resolvedURL: string;
  mimeType: string;
  verified: boolean;
}

/** Resolver configuration */
export interface BNRPConfig {
  resolvers: string[];         // Array of resolver base URLs
  timeout: number;             // Request timeout in ms (default: 5000)
  cacheTTL: number;            // Cache TTL in seconds (default: 600)
  retryCount: number;          // Number of retries per resolver (default: 2)
  requireVerification: boolean; // Require anti-spoofing for reverse (default: true)
}

// ─────────────────────────────────────────────────
// BNRP Client Interface
// ─────────────────────────────────────────────────

export interface BNRPClient {
  /**
   * Resolve a .btc name to its full record.
   * @param name - The domain name (e.g., "alice.btc")
   * @returns The complete BNRP record, or null if the name doesn't exist.
   */
  resolve(name: string): Promise<BNRPRecord | null>;

  /**
   * Reverse-resolve an address to its primary .btc name.
   * Includes anti-spoofing verification.
   * @param address - A blockchain address (BTC, ETH, or SOL)
   * @returns The primary name, or null if none is set or verification fails.
   */
  reverseResolve(address: string): Promise<string | null>;

  /**
   * Get a single chain address for a name.
   * @param name - The domain name
   * @param coinType - SLIP-44 coin type (use CoinType enum)
   * @param subtype - For BTC (coinType=0): address subtype
   * @returns The address string, or null if not set.
   */
  getAddress(name: string, coinType: number, subtype?: BTCSubtype): Promise<string | null>;

  /**
   * Get profile metadata for a name.
   * @param name - The domain name
   * @returns Profile metadata object.
   */
  getProfile(name: string): Promise<BNRPProfile | null>;

  /**
   * Get the resolved avatar URL for a name.
   * @param name - The domain name
   * @returns HTTP-fetchable avatar URL, or null.
   */
  getAvatar(name: string): Promise<string | null>;

  /**
   * Get the web/content URL for a name.
   * @param name - The domain name
   * @returns HTTP-fetchable content URL, or null.
   */
  getContent(name: string): Promise<string | null>;

  /**
   * Get the bitmap/metaverse record for a name.
   * @param name - The domain name
   * @returns Bitmap record, or null.
   */
  getBitmap(name: string): Promise<BNRPBitmapRecord | null>;
}
```

**Example SDK Usage:**

```typescript
import { createBNRPClient, CoinType } from '@bnrp/sdk';

const client = createBNRPClient({
  resolvers: ['https://resolver.bnrp.org/v1', 'https://api.btcname.id/v1'],
  timeout: 5000,
  cacheTTL: 600,
  retryCount: 2,
  requireVerification: true,
});

// Resolve a name
const record = await client.resolve('alice.btc');
console.log(record?.addresses.btc_taproot);
// → "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297"

// Get ETH address
const ethAddr = await client.getAddress('alice.btc', CoinType.ETH);
// → "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"

// Get Base L2 address
const baseAddr = await client.getAddress('alice.btc', CoinType.BASE);
// → "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"

// Reverse resolve
const name = await client.reverseResolve(
  'bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297'
);
// → "alice.btc"

// Get avatar
const avatarURL = await client.getAvatar('alice.btc');
// → "https://ordinals.com/content/9ba5066e...i0"
```

### 14.3 Wallet Integration Example

The following describes how a Bitcoin wallet (e.g., UniSat, Xverse) would integrate BNRP:

**1. Address Display — Show Primary Name**

```pseudocode
// On wallet main screen, where the address is normally displayed:
function displayWalletIdentity(walletAddress: string):
    primaryName = bnrpClient.reverseResolve(walletAddress)
    
    if primaryName is not null:
        avatarURL = bnrpClient.getAvatar(primaryName)
        display.showAvatar(avatarURL, size=32)       // 32x32 compact view
        display.showName(primaryName)                 // "alice.btc"
        display.showBadge("Verified")                 // green checkmark
        display.showAddressSubtext(truncate(walletAddress))  // "bc1p5d7...9rus"
    else:
        display.showDefaultIcon()
        display.showAddress(truncate(walletAddress))
```

**2. Send — Resolve Name to Address**

```pseudocode
// User types "bob.btc" in the send-to field:
function resolveSendTarget(input: string, selectedChain: string):
    if input.endsWith(".btc") or input.endsWith(".sats") or ...:
        // Input is a name — resolve it
        if selectedChain == "BTC":
            address = bnrpClient.getAddress(input, CoinType.BTC, "taproot")
        else if selectedChain == "ETH":
            address = bnrpClient.getAddress(input, CoinType.ETH)
        else if selectedChain == "SOL":
            address = bnrpClient.getAddress(input, CoinType.SOL)
        
        if address is null:
            display.showError("No " + selectedChain + " address found for " + input)
            return
        
        // Show confirmation with full details
        avatarURL = bnrpClient.getAvatar(input)
        display.showSendConfirmation(
            name: input,
            avatar: avatarURL,
            resolvedAddress: address,
            chain: selectedChain,
            inscriptionId: record.inscriptionId  // show on first use
        )
    else:
        // Input is a raw address — use directly
        display.showSendConfirmation(address: input, chain: selectedChain)
```

**3. Profile View**

```pseudocode
// User taps on a .btc name to see the full profile:
function showProfile(name: string):
    record = bnrpClient.resolve(name)
    if record is null:
        display.showError("Name not found")
        return
    
    avatarURL = bnrpClient.getAvatar(name)
    display.showAvatar(avatarURL, size=128)   // 128x128 profile view
    display.showDisplayName(record.profile?.display ?? name)
    display.showBio(record.profile?.description)
    
    // Show chain addresses
    for (field, address) in record.addresses:
        display.showAddressRow(chainName(field), address)
    
    // Show social links
    if record.profile?.["com.twitter"]:
        display.showSocialLink("Twitter", "@" + record.profile["com.twitter"])
    if record.profile?.["com.github"]:
        display.showSocialLink("GitHub", record.profile["com.github"])
    if record.profile?.["com.nostr"]:
        display.showSocialLink("Nostr", record.profile["com.nostr"])
    
    // Show bitmap info
    if record.bitmap:
        display.showBitmapLink(record.bitmap.bitmap_district, record.bitmap.bitmap_parcel)
```

---

## 15. Security Model

### 15.1 Threat Model Table

| # | Threat | Attack Vector | Impact | Mitigation |
|---|--------|--------------|--------|------------|
| 1 | Indexer spoofing | Malicious indexer returns false ownership data, claiming a name belongs to the attacker | Users send funds to wrong address; impersonation | Multi-resolver quorum; client-side verification via Bitcoin RPC `gettxout`; allow custom resolver URLs |
| 2 | Stale indexer data | Indexer falls behind the chain tip, returning outdated ownership or record data | Sending funds to a previous owner's address after a name transfer | `X-BNRP-Block-Height` header allows clients to detect stale data; health endpoint exposes sync lag; minimum 3-confirmation rule provides buffer |
| 3 | Primary name spoofing | Attacker claims `alice.btc` as their primary name without owning it | False identity display in wallets | Bidirectional anti-spoofing verification (§9.1): forward resolve must match reverse claim |
| 4 | Avatar abuse | Avatar URI points to CSAM, malware download, or phishing page | Legal liability; user harm; malware infection | MIME type validation (only `image/*`); resolver-side content scanning; CSP headers for avatar display; SVG sanitization |
| 5 | Homoglyph phishing | Attacker registers `аlice.btc` (Cyrillic 'а') to impersonate `alice.btc` (Latin 'a') | Phishing; fund theft | Name normalization (§8.1); confusable detection (Unicode TR#39); wallet UX: warn on mixed-script names; show Punycode alongside display name |
| 6 | Reorg race condition | Attacker exploits a chain reorg to temporarily own a name, set a primary name, and revert | Brief impersonation; potential fund interception | 3-confirmation minimum; reorg rollback procedure (§8.3); resolvers return 503 during reorg processing |
| 7 | Malicious resolver | Resolver returns fabricated records (wrong addresses, fake profiles) | Fund theft; identity spoofing | SDK queries multiple resolvers and compares results; signed responses option; users can self-host resolvers |
| 8 | Off-chain record replay | Attacker replays a previously valid off-chain signed record after the user has revoked it | Stale/revoked data displayed | Nonce in signed records; timestamp with expiry; on-chain revocation inscription overrides off-chain records |
| 9 | Proxy inscription attack | Proxy inscribe service creates a `primary-name` inscription; minter is the proxy, not the user | Invalid primary name if Solution A minter check fails; loss of primary name | BNRP explicitly checks minter == owner for Solution A; Solution B (self-transfer) is immune to proxy attacks |
| 10 | Inscription number instability | Inscription numbers change during reindexing; records referencing `ord-num:` break | Avatar or content references become invalid | BNRP uses inscription ID (txid + index) as canonical identifier; `ord-num:` scheme carries an explicit instability warning |
| 11 | DNS gateway compromise | Attacker compromises a BNRP gateway's DNS, redirecting traffic | Man-in-the-middle; content substitution; credential theft | DNSSEC for gateway domains; certificate pinning in browser extensions; multiple independent gateways; content hash verification |
| 12 | MIME type attack via SVG | SVG inscription contains embedded JavaScript, triggering XSS when rendered as avatar | Cross-site scripting; session hijacking; credential theft | SVG sanitization (strip `<script>`, event handlers, `javascript:` URIs); serve avatars from sandboxed origin; CSP: `script-src 'none'` |

### 15.2 Anti-Phishing Recommendations

1. **Name normalization**: All names MUST be normalized per §8.1 before display or resolution. This catches basic case and Unicode normalization issues.
2. **Confusable detection**: Wallets SHOULD implement Unicode Technical Report #39 confusable detection. Names containing characters from multiple Unicode scripts (e.g., Latin + Cyrillic) MUST be flagged.
3. **Display warnings**: For names flagged as confusable, wallets MUST display a prominent warning: "This name contains characters from multiple scripts and may be a phishing attempt."
4. **Punycode tooltip**: For any name containing non-ASCII characters, wallets SHOULD display the Punycode representation on hover/tap.
5. **First-use confirmation**: The first time a user interacts with a `.btc` name (e.g., sending funds), the wallet MUST display the full inscription ID alongside the display name and require explicit confirmation.
6. **Name similarity warning**: When resolving a name, wallets SHOULD check if the name is visually similar to a well-known name (e.g., `satoshi.btc`) and warn if the current name is not the well-known one.
7. **Verified name registries**: The community MAY maintain a public list of verified high-value names (e.g., exchange names, project names) with their canonical inscription IDs, allowing wallets to flag impostors.

### 15.3 Resolver Trust Model

**No single resolver is trusted.** The BNRP trust model operates on the principle that the Bitcoin blockchain is the only trusted data source. Resolvers are convenience infrastructure that caches and serves blockchain-derived data.

**Trust mitigation strategies:**
1. **Multi-resolver quorum (optional)**: Clients can be configured to query 2 or more resolvers and require agreement on critical fields (owner address, primary address records). If resolvers disagree, the client can fall back to direct Bitcoin RPC verification.
2. **Self-hosted resolvers**: Wallets MUST allow users to configure a custom resolver URL. Power users and organizations can run their own resolver connected to their own Bitcoin full node.
3. **Signed responses (optional)**: Resolvers MAY sign their responses with a known Schnorr key. The signature covers the response body and the `X-BNRP-Block-Height` header. Clients with the resolver's public key can verify response integrity. This prevents man-in-the-middle attacks between client and resolver but does not guarantee correctness — a malicious resolver can sign false data.
4. **Block height verification**: Clients SHOULD verify that the `X-BNRP-Block-Height` reported by the resolver is within a reasonable range of the current time (approximately 1 block per 10 minutes). A resolver reporting a block height significantly behind the chain tip is potentially serving stale data.

### 15.4 Avatar Safety Enforcement

1. Resolvers MUST validate avatar content MIME type before returning the avatar URL. Only `image/jpeg`, `image/png`, `image/gif`, `image/webp`, and `image/svg+xml` are permitted.
2. SVG avatars MUST be processed through a server-side sanitizer that removes all active content (scripts, event handlers, external references, `<foreignObject>` elements).
3. Wallets MUST render avatars in a sandboxed context — never in the same origin as the application.
4. Wallets SHOULD implement a maximum file size for avatars (5MB) and reject larger files.
5. Wallets SHOULD NOT auto-play animated GIFs in compact view (32×32px). Animation should only play in expanded profile view.
6. For `data:` URIs, wallets MUST validate the MIME type prefix before rendering.

### 15.5 Stale Data Handling

**Minimum confirmation requirements:**
- Name inscription validity: 3 confirmations
- Routing/profile/content/bitmap record validity: 3 confirmations
- Name transfer recognition: 3 confirmations
- Primary name (Solution B self-transfer) recognition: 3 confirmations

**Cache TTL recommendations:**

| Data Type | Recommended TTL | Justification |
|-----------|----------------|---------------|
| Address records | 600 seconds (10 min) | Addresses change infrequently; 10 min balances freshness and load |
| Ownership data | 60 seconds (1 min) | Ownership changes need prompt detection |
| Profile metadata | 3600 seconds (1 hour) | Profile data changes rarely |
| Avatar URL | 600 seconds (10 min) | Avatar changes are infrequent |
| Content URL | 600 seconds (10 min) | Content changes are infrequent |
| Health check | 30 seconds | Stale health data is useless |

**Indexer disagreement handling:**
If two resolvers return different data for the same name:
1. Compare `X-BNRP-Block-Height` headers — prefer the resolver with the higher (more recent) block height.
2. If block heights are equal but data differs, query a third resolver if available.
3. If no third resolver is available, the client SHOULD verify the owner address via direct Bitcoin RPC (`gettxout`) if possible.
4. For non-critical fields (profile, avatar), accept the data from the resolver with the higher block height.
5. For critical fields (addresses used for fund transfers), require agreement from at least 2 resolvers or direct verification.

---

## 16. Governance / Open Standard

### 16.1 BNRP as Open Standard

BNRP is designed as an open, community-governed protocol specification. The governance model is inspired by the Bitcoin Improvement Proposal (BIP) and Ethereum Improvement Proposal (EIP) processes but adapted for the Bitcoin name resolution domain.

**Core principles:**
- **No controlling entity**: No company, foundation, or individual controls the BNRP specification. It is a public good.
- **Public repository**: The specification is maintained in a public GitHub repository (`github.com/bnrp/bnrp-spec`) under a CC0 license.
- **Community governance**: Changes are proposed via BNRP Improvement Proposals (BNRP-IPs), reviewed by the community, and merged by maintainers elected by rough consensus.
- **Rough consensus**: Decisions are made by rough consensus among active contributors, not by formal voting. Contentious issues are resolved through technical discussion and, if necessary, through running code (implementations that demonstrate the viability of a proposal).
- **Communication channels**: Primary discussion occurs on the BNRP GitHub repository (issues and pull requests), with supplementary discussion on Discord (`discord.gg/bnrp`) and the BtcName Discord `#bnrp-spec` channel.

### 16.2 BNRP Improvement Proposal (BNRP-IP) Process

**Submitting a BNRP-IP:**
1. Author drafts the proposal using the BNRP-IP template (Appendix B).
2. Author opens a pull request to the `bnrp/bnrp-spec` repository with the proposal in the `proposals/` directory.
3. The PR is assigned a BNRP-IP number by a maintainer.
4. Community review period: minimum 14 days for Draft status, minimum 30 days for Review status.

**Status levels:**

| Status | Description | Requirements |
|--------|-------------|-------------|
| Draft | Initial proposal, open for feedback | Author submits PR; basic format compliance |
| Review | Proposal is substantively complete and under active review | At least 14 days as Draft; addressed initial feedback |
| Final | Accepted and canonical | At least 30 days in Review; rough consensus achieved; at least one reference implementation |
| Obsolete | Superseded by a newer BNRP-IP | A Final BNRP-IP explicitly supersedes this one |

**Review criteria:**
- Technical soundness and completeness
- Backward compatibility analysis
- Security implications documented
- Reference implementation (required for Final status)
- Community consensus

### 16.3 Core Spec vs Extensions

**Core specifications (MUST be standardized first):**
1. BNRP-IP-01: Name normalization rules (§8.1)
2. BNRP-IP-02: Routing inscription format with SLIP-44 address model (§5, §6.1)
3. BNRP-IP-03: Primary name mechanism — Solution B default, Solution A opt-in (§9.2)
4. BNRP-IP-04: Reverse resolution with anti-spoofing (§9)
5. BNRP-IP-05: Avatar URI schemes with ownership verification (§10)
6. BNRP-IP-06: REST API specification v1 (§14.1)

These six specifications form the minimum viable standard. A resolver that implements all six is a "BNRP Core Compliant" resolver.

**Optional extensions (can be standardized incrementally):**
- BNRP-IP-07: Profile text records (§6.2)
- BNRP-IP-08: Cross-chain address records with coinType schema (Appendix A)
- BNRP-IP-09: Web/content resolution and gateway specification (§11)
- BNRP-IP-10: Bitmap/metaverse records (§12)
- BNRP-IP-11: Signed off-chain records (§19.3)
- BNRP-IP-12: Decentralized resolver network
- BNRP-IP-13: Cross-chain reverse resolution

### 16.4 Third-Party Adoption Path

**Step 1 — Read-Only Integration (2-4 weeks):**
Implement the BNRP resolver API (§14.1) as a read-only service. This requires an Ordinals indexer that tracks `"p": "btcname"` and `"p": "bnrp"` inscriptions. UniSat can expose their existing indexer via the standardized API.

**Step 2 — Primary Name Display (1-2 weeks after Step 1):**
Wallet integrates the BNRP SDK and displays primary names where addresses normally appear. Calls `reverseResolve()` on the user's wallet address at startup.

**Step 3 — Full Resolution Support (2-4 weeks after Step 2):**
Wallet supports resolving `.btc` names in send flows, profile views, and contact lists. Implements avatar display and chain-specific address resolution.

**Step 4 — Governance Participation (ongoing):**
The adopting team contributes to spec governance: reviewing BNRP-IPs, submitting proposals for improvements discovered during implementation, and participating in community discussions.

---

## 17. Migration from Current BtcName

### 17.1 What BtcName Already Provides (Reuse Directly)

| BtcName Feature | BNRP Equivalent | Migration Action |
|-----------------|-----------------|-----------------|
| Name ownership via Ordinals inscription | Identical — BNRP relies on same mechanism | None. Fully reused. |
| `"p": "btcname", "op": "routing"` format | `"p": "bnrp", "op": "routing"` (superset) | None for existing inscriptions. BNRP resolvers index `btcname` routing records. |
| `"op": "reverse"` with `reverse_address` | `"p": "bnrp", "op": "reverse"` (same semantics) | None. BNRP resolvers index `btcname` reverse records. |
| Forward resolution: name → addresses | Identical (extended with more chains) | None. Existing routing records work. |
| Anti-spoofing on reverse resolution | Formalized in §9.1 with explicit pseudocode | None. Existing behavior is codified. |
| `avatar` field (inscription ID) | Extended with URI schemes (§10) | None. Bare inscription IDs are interpreted as `ord:{id}` for backward compatibility. |
| Solution B primary name (self-transfer) | Adopted as default primary name mechanism | None. Existing self-transfers at/after block 840000 are recognized. |
| 3-confirmation rule | Identical | None. |
| `.btc`, `.sats`, `.unisat`, `.x`, `.fb` TLDs | All supported | None. |

### 17.2 What Needs Extension

**Avatar enhancement:**
BtcName supports a bare inscription ID in the `avatar` field. BNRP extends this with URI schemes (`ord:`, `ipfs://`, `ipns://`, `https://`, `data:`), a fallback chain, ownership verification, and MIME type validation. **Backward compatibility**: a bare inscription ID (no scheme prefix) in an existing `btcname` routing record is automatically interpreted as `ord:{id}`.

**Cross-chain address records:**
BtcName supports `eth_address`, `matic_address`, and `sol_address` as explicit fields. BNRP adds Base, Arbitrum, and an extensible `addr_{coinType}` schema. **Backward compatibility**: BNRP resolvers map `eth_address` → `eth`, `matic_address` → `addr_2147483785`, `sol_address` → `sol`.

**Profile text records:**
BtcName has no profile record type. BNRP adds a complete `"op": "profile"` inscription type with ENSIP-5-equivalent fields. **Migration**: users create new BNRP profile inscriptions. No existing data is affected.

**Web/content resolution:**
Not in BtcName. BNRP adds the `"op": "content"` record type and the `content` field in routing inscriptions. **Migration**: users create new content inscriptions. No existing data is affected.

**Bitmap/metaverse records:**
Not in BtcName. BNRP adds the `"op": "bitmap"` record type. **Migration**: users create new bitmap inscriptions. No existing data is affected.

**Open governance:**
BtcName is maintained by a single team. BNRP establishes an open governance process (§16). **Migration**: the BtcName team participates in BNRP governance as a stakeholder, not a sole authority.

### 17.3 Backward Compatibility Rules

1. **All existing BtcName routing inscriptions remain valid.** A BNRP resolver encountering `"p": "btcname", "op": "routing"` MUST parse and index it using the field mapping in §17.2.
2. **BNRP uses `"p": "bnrp"` for new inscriptions.** New records inscribed under BNRP use the `bnrp` protocol prefix. This allows indexers to distinguish BNRP-native records from legacy BtcName records.
3. **BNRP resolvers MUST index both protocols.** A compliant resolver indexes inscriptions with `"p": "btcname"` and `"p": "bnrp"`. When both exist for the same name, the most recently confirmed record wins (regardless of protocol prefix).
4. **Existing reverse resolution maps directly.** `"p": "btcname", "op": "reverse"` inscriptions are treated identically to `"p": "bnrp", "op": "reverse"` inscriptions.
5. **Legacy field name mapping:**
   - `btc_p2phk` → `btc_legacy`
   - `btc_p2sh` → `btc_p2sh` (unchanged)
   - `btc_segwit` → `btc_segwit` (unchanged)
   - `eth_address` → `eth`
   - `matic_address` → `addr_2147483785`
   - `sol_address` → `sol`
   - `avatar` (bare inscription ID) → `avatar` with implicit `ord:` prefix

### 17.4 Breaking Changes / Risks

**Solution A vs Solution B primary name:**
BNRP declares Solution B as the default mechanism, with Solution A as an opt-in extension. This is not a breaking change because both solutions were in draft status and no canonical standard existed. However, any implementations that exclusively supported Solution A will need to add Solution B support.

**Protocol prefix:**
Going forward, BNRP recommends that new inscriptions use `"p": "bnrp"`. However, inscriptions with `"p": "btcname"` remain fully valid and indexed. The community may eventually converge on one prefix, but BNRP does not force deprecation of `"btcname"`.

**Recommendation:** BtcName.id should update their inscription creation tools to offer both `"p": "btcname"` (legacy) and `"p": "bnrp"` (new) options, with `"p": "bnrp"` as the default for new inscriptions. This allows a gradual, non-breaking transition.

---

## 18. MVP Architecture

### 18.1 MVP Scope

The minimum viable implementation for BNRP adoption includes:

1. **BNRP Indexer** — Open-source indexer that reads Bitcoin blocks, identifies `btcname` and `bnrp` inscriptions, maintains ownership state, and applies the "Last is New" rule. Written in Rust or Go for performance.
2. **BNRP REST API** — HTTP service implementing the 8 endpoints defined in §14.1. Queries the indexer database.
3. **BNRP SDK** — JavaScript/TypeScript client library implementing the interface in §14.2. Published to npm as `@bnrp/sdk`.
4. **UniSat integration** — UniSat wallet displays primary names via the BNRP SDK. Users see `alice.btc` instead of `bc1p...`.
5. **Gateway** — A web server at `{name}.btc.bnrp.io` that resolves BNRP content records and serves the content.

### 18.2 MVP Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    MVP DEPLOYMENT                             │
│                                                              │
│  ┌──────────────┐     ┌──────────────────────────────────┐   │
│  │  UniSat       │     │   BNRP Gateway                   │   │
│  │  Wallet       │     │   (alice.btc.bnrp.io)            │   │
│  │  (SDK client) │     │                                  │   │
│  └──────┬───────┘     └──────────────┬───────────────────┘   │
│         │                            │                       │
│         │  HTTPS                     │  HTTPS                │
│         ▼                            ▼                       │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              BNRP REST API Server                      │   │
│  │    /v1/resolve  /v1/reverse  /v1/address  /v1/avatar  │   │
│  │    /v1/profile  /v1/content  /v1/bitmap   /v1/health  │   │
│  └──────────────────────┬────────────────────────────────┘   │
│                         │                                    │
│                         │  SQL queries                       │
│                         ▼                                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              BNRP Indexer                               │   │
│  │    • Reads Bitcoin blocks via bitcoind RPC              │   │
│  │    • Identifies btcname/bnrp inscriptions               │   │
│  │    • Tracks UTXO ownership                              │   │
│  │    • Maintains PostgreSQL database                      │   │
│  └──────────────────────┬────────────────────────────────┘   │
│                         │                                    │
│                         │  Bitcoin RPC                       │
│                         ▼                                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              Bitcoin Core Node                          │   │
│  │    • Full node with txindex=1                           │   │
│  │    • Ordinals-aware (ord or equivalent)                 │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 18.3 MVP Data Flow Example

**Scenario:** User types "alice.btc" in UniSat wallet to send 0.1 BTC.

1. **User input**: User types `alice.btc` into the "Send to" field of UniSat wallet.
2. **SDK call**: UniSat calls `bnrpClient.getAddress("alice.btc", CoinType.BTC, "taproot")`.
3. **SDK normalization**: The SDK normalizes `alice.btc` → `alice.btc` (already normalized).
4. **API request**: SDK sends `GET https://resolver.bnrp.org/v1/address/alice.btc?chain=0&subtype=taproot`.
5. **Resolver processing**: The resolver queries its indexer database for the latest valid routing inscription for `alice.btc`, extracts the `btc_taproot` field, and verifies it has 3+ confirmations.
6. **API response**: `{ "name": "alice.btc", "chain": 0, "subtype": "taproot", "address": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297" }`
7. **Avatar fetch**: UniSat also calls `bnrpClient.getAvatar("alice.btc")` to display alice's avatar in the confirmation dialog.
8. **Avatar resolution**: The resolver resolves `ord:9ba5066e...i0` → `https://ordinals.com/content/9ba5066e...i0`, verifies MIME type is `image/png`, verifies ownership match.
9. **User confirmation**: UniSat displays a confirmation dialog showing:
   - Alice's avatar (128×128px)
   - Name: "alice.btc"
   - Resolved address: `bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297`
   - Amount: 0.1 BTC
   - "Verified" badge (forward resolution matches)
10. **Transaction**: User confirms, UniSat constructs and broadcasts the Bitcoin transaction to Alice's taproot address.

---

## 19. Ideal Long-Term Architecture

### 19.1 Long-Term Vision

The long-term vision for BNRP is a fully decentralized, multi-resolver name resolution network for Bitcoin that rivals ENS in capability while staying rooted in Bitcoin's security model.

**Key long-term capabilities:**
- **Decentralized resolver network**: Dozens of independent resolver implementations operated by different organizations, discoverable via on-chain registry and DNS.
- **On-chain resolver registry**: A canonical Ordinals inscription that lists active resolvers with their public keys, enabling trustless resolver discovery.
- **Native browser support**: The Brave browser already supports `.eth` via ENS; long-term, Brave and other browsers should natively support `.btc` via BNRP.
- **Signed off-chain records**: A CCIP-Read equivalent for Bitcoin (§19.3) that allows frequent record updates without per-update inscription costs.
- **Cross-chain reverse resolution**: An Ethereum address can resolve to a `.btc` name, enabling unified identity across Bitcoin and Ethereum ecosystems.
- **Lightning Network integration**: Deep integration with Lightning, including LNURL resolution, Lightning Address support, and BOLT-12 offer mapping.
- **Decentralized avatar CDN**: Popular ordinals avatars pinned to IPFS by the resolver network, ensuring availability even if individual ordinals indexers are down.

### 19.2 Long-Term Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     LONG-TERM BNRP NETWORK                          │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CLIENT LAYER                                │  │
│  │                                                               │  │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │  │
│  │  │ Wallets  │  │ Browsers │  │ dApps    │  │ Bitmap Apps  │   │  │
│  │  │ UniSat   │  │ Brave    │  │ DeFi     │  │ Atlas        │   │  │
│  │  │ Xverse   │  │ Chrome   │  │ DEX      │  │ bitmap.land  │   │  │
│  │  │ Leather  │  │ + ext    │  │          │  │              │   │  │
│  │  │ OKX      │  │          │  │          │  │              │   │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │  │
│  │       └──────────────┴──────────────┴───────────────┘           │  │
│  │                          │  BNRP SDK                           │  │
│  └──────────────────────────┼────────────────────────────────────┘  │
│                             ▼                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  RESOLVER NETWORK                              │  │
│  │                                                               │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │  │
│  │  │Resolver A│ │Resolver B│ │Resolver C│ │ Self-hosted       │ │  │
│  │  │(BNRP.org)│ │(BtcName) │ │(UniSat)  │ │ Resolvers        │ │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────────────┘ │  │
│  │       │             │            │             │               │  │
│  │       ├─── Signed responses ─────┤             │               │  │
│  │       ├─── Quorum verification ──┤             │               │  │
│  │       │             │            │             │               │  │
│  └───────┼─────────────┼────────────┼─────────────┼──────────────┘  │
│          ▼             ▼            ▼             ▼                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  INDEXER LAYER                                 │  │
│  │  Multiple independent indexer implementations                 │  │
│  │  • Canonicalization rules ensure convergence                  │  │
│  │  • Bitcoin RPC gettxout as ground truth                       │  │
│  └──────────────────────┬────────────────────────────────────────┘  │
│                         ▼                                           │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  BITCOIN BLOCKCHAIN                            │  │
│  │  + On-chain resolver registry (Ordinals inscription)          │  │
│  │  + Name inscriptions + BNRP records                           │  │
│  │  + Signed off-chain record anchors                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  GATEWAY NETWORK                               │  │
│  │                                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │ bnrp.io      │  │ btcname.id   │  │ community    │        │  │
│  │  │ gateway      │  │ gateway      │  │ gateways     │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────┐                     │  │
│  │  │ Decentralized Avatar CDN             │                     │  │
│  │  │ (IPFS-pinned Ordinals content)       │                     │  │
│  │  └──────────────────────────────────────┘                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.3 Signed Off-Chain Records (CCIP-Read Equivalent for Bitcoin)

On Ethereum, EIP-3668 (CCIP-Read) enables smart contracts to revert with a gateway URL, allowing clients to fetch off-chain data and verify it on-chain. Bitcoin lacks smart contracts, but Bitcoin taproot keys can sign arbitrary messages, enabling a conceptually similar mechanism.

**Problem:** Updating a BNRP record (e.g., changing a Twitter handle) requires a new Ordinals inscription, costing transaction fees and consuming block space. For frequently-updated metadata, this is impractical.

**Solution: Signed Off-Chain Records**

A name owner can publish record updates off-chain, signed by their taproot key, and any resolver can verify and serve these records without a new inscription.

**Record format:**

```json
{
  "type": "bnrp-offchain-record",
  "name": "alice.btc",
  "records": {
    "com.twitter": "alice_btc_new",
    "description": "Updated bio text"
  },
  "nonce": 42,
  "expiry": 1750000000,
  "signer": "bc1p5d7rjq7g6rdk2yhzks9smlaqtedr4dekq08ge8ztwac72sfr9rusxg3297",
  "signature": "base64-encoded-schnorr-signature..."
}
```

**Verification flow:**

```pseudocode
function verifyOffchainRecord(record: OffchainRecord) -> boolean:
    // Step 1: Verify the signer is the current owner of the name
    nameOwner = resolve(record.name).owner
    if nameOwner != record.signer:
        return false  // Signer is not the name owner
    
    // Step 2: Verify the record has not expired
    if currentTimestamp() > record.expiry:
        return false  // Record has expired
    
    // Step 3: Verify the nonce is higher than the last known nonce for this name
    lastNonce = offchainStore.getLastNonce(record.name)
    if record.nonce <= lastNonce:
        return false  // Replay attack — nonce too low
    
    // Step 4: Construct the message to verify
    message = canonicalJSON({
        type: record.type,
        name: record.name,
        records: record.records,
        nonce: record.nonce,
        expiry: record.expiry,
        signer: record.signer
    })
    
    // Step 5: Verify the Schnorr signature
    messageHash = taggedHash("BIP0340/bnrp", sha256(message))
    publicKey = extractTaprootInternalKey(record.signer)
    signature = base64Decode(record.signature)
    
    if not schnorrVerify(publicKey, messageHash, signature):
        return false  // Invalid signature
    
    // Step 6: Record is valid — store and serve
    offchainStore.storeRecord(record)
    return true
```

**Key properties:**
- **No on-chain cost** for record updates.
- **Verifiable** by any party using the owner's public taproot key.
- **Expirable** — records include an expiry timestamp; stale records are automatically invalidated.
- **Nonce-protected** — prevents replay of revoked records.
- **Subordinate to on-chain** — if an on-chain inscription contradicts an off-chain record, the on-chain inscription always wins.
- **Revocable** — the owner can revoke all off-chain records by inscribing an on-chain revocation: `{"p": "bnrp", "op": "revoke-offchain", "name": "alice.btc", "min_nonce": 100}`.

---

## 20. 3-Phase Roadmap

### Phase 1: Foundation (Months 1-6)

| Deliverable | BNRP-IP | Description | Priority |
|-------------|---------|-------------|----------|
| Name normalization spec | BNRP-IP-01 | Complete normalization rules (§8.1) with reference implementation | Critical |
| Routing inscription format | BNRP-IP-02 | Backward-compatible routing format with SLIP-44 `addr_{coinType}` extension | Critical |
| Primary name standard | BNRP-IP-03 | Adopt Solution B as default, Solution A as opt-in; coexistence rules | Critical |
| Reverse resolution rules | BNRP-IP-04 | Anti-spoofing verification; bidirectional check; edge case handling | Critical |
| Avatar URI schemes | BNRP-IP-05 | `ord:`, `ipfs://`, `ipns://`, `https://`, `data:` schemes; ownership verification; fallback chain | Critical |
| REST API spec v1 | BNRP-IP-06 | 8 endpoints; request/response formats; error codes; health check | Critical |
| Reference indexer | — | Open-source indexer (Rust/Go) that indexes `btcname` and `bnrp` inscriptions | Critical |
| Reference resolver | — | Open-source resolver implementing BNRP-IP-06 | Critical |

### Phase 2: Expansion (Months 7-12)

| Deliverable | BNRP-IP | Description | Priority |
|-------------|---------|-------------|----------|
| Profile text records | BNRP-IP-07 | Profile inscription format with ENSIP-5-equivalent fields | High |
| Cross-chain address records | BNRP-IP-08 | Formal coin type registry; EVM L2 chain encoding rules | High |
| Web/content resolution | BNRP-IP-09 | Content record format; gateway specification; contenthash semantics | High |
| Bitmap/metaverse extension | BNRP-IP-10 | Bitmap record format; auto-mapping rules; metaverse coordinate system | Medium |
| SDK release (JS/TS) | — | npm package `@bnrp/sdk` implementing full client interface | High |
| SDK release (Python) | — | PyPI package `bnrp-sdk` for server-side integrations | Medium |
| UniSat wallet integration | — | Primary name display, send-to-name, avatar display in UniSat | High |
| First public gateways | — | At least 2 independent gateways operating (`bnrp.io`, `btcname.id`) | Medium |

### Phase 3: Maturity (Months 13-24)

| Deliverable | BNRP-IP | Description | Priority |
|-------------|---------|-------------|----------|
| Signed off-chain records | BNRP-IP-11 | Schnorr-signed off-chain record updates with nonce and expiry | High |
| Decentralized resolver network spec | BNRP-IP-12 | On-chain resolver registry; resolver discovery protocol; quorum verification | Medium |
| Cross-chain reverse resolution | BNRP-IP-13 | Resolve ETH/SOL addresses to `.btc` names; ENS interoperability | Medium |
| Browser extension | — | Chrome/Firefox/Brave extension for native `.btc` resolution | High |
| Multiple resolver implementations | — | At least 3 independent resolver implementations from different teams | Medium |
| Multi-wallet adoption | — | Xverse, Leather, OKX Wallet integrate BNRP | High |
| Security audit | — | Independent security audit of reference indexer, resolver, and SDK | High |
| BNRP governance formalization | — | Elected maintainers; formal BNRP-IP review committee | Medium |

---

## 21. Risks and Tradeoffs

### 21.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Ordinals protocol changes break inscription tracking | Low | High — all BNRP records become unindexable | BNRP indexer abstracts Ordinals specifics; fallback to raw Bitcoin transaction parsing |
| Bitcoin chain reorg invalidates recently-set records | Low | Medium — temporary state inconsistency | 3-confirmation minimum; reorg rollback procedure; 503 during processing |
| Inscription number instability causes avatar/content breakage | Medium | Low — only affects `ord-num:` references | BNRP mandates inscription ID as canonical; `ord-num:` carries instability warning |
| IPFS content becomes unavailable (pinning failure) | Medium | Medium — avatars and websites go offline | Fallback chain (§10.4); multiple IPFS gateways; Ordinals as fallback storage |
| Indexer divergence due to implementation bugs | Medium | High — different resolvers return different data | Deterministic canonicalization rules; cross-indexer test suite; Bitcoin RPC ground truth |
| JSON inscription size limits hit Bitcoin relay policy | Low | Medium — large profile records rejected by mempool | Keep individual records under 80KB; split large records across multiple inscriptions |

### 21.2 Ecosystem Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Vendor capture — single entity controls all resolvers | Medium | High — defeats decentralization goal | Open specification; reference implementations; governance prevents single-vendor lock-in |
| Indexer centralization — everyone uses the same indexer | High (early) | Medium — single point of failure | Encourage multiple implementations; make indexer easy to self-host; document deployment |
| Low adoption — wallets don't integrate BNRP | Medium | High — standard becomes irrelevant | Start with UniSat (strong BtcName relationship); SDK reduces integration cost; show clear user benefit |
| Competing standard emerges | Medium | Medium — ecosystem fragmentation | BNRP is backward-compatible with BtcName; community governance prevents fork incentive; include rather than compete |
| Regulatory risk — name services become regulated | Low | High — compliance requirements may conflict with permissionless design | BNRP is a protocol specification, not a service; no central operator to regulate |

### 21.3 Tradeoffs Made

**On-chain vs off-chain records:**
BNRP prioritizes on-chain records as the canonical source of truth. This provides immutability and censorship resistance but costs transaction fees for every update. The tradeoff is addressed in Phase 3 with signed off-chain records (§19.3), which enable frequent updates while keeping on-chain records as the authoritative fallback.

**Solution A vs Solution B for primary names:**
Solution B (self-transfer) was chosen as the default because it is simpler, immune to proxy inscription attacks, and requires no additional inscription. The tradeoff is the loss of metadata attachment (avatar, display) in the primary name declaration. Solution A is retained as an opt-in extension for users who want metadata-rich primary name declarations.

**Single protocol prefix (bnrp) vs multi-protocol support:**
BNRP introduces the `"p": "bnrp"` prefix while maintaining backward compatibility with `"p": "btcname"`. The tradeoff: two protocol prefixes increase implementation complexity, but dropping `btcname` support would break all existing records.

**Inscription ID as canonical identifier vs inscription number:**
BNRP mandates inscription IDs (`{txid}i{index}`) as the only canonical identifier. Inscription numbers are easier for humans to reference but are unstable during reindexing. The tradeoff: less human-friendly references in exchange for reliable, immutable identifiers.

---

## 22. Final Recommendation

### 22.1 Immediate Actions

1. **Publish BNRP-IP-01 through BNRP-IP-06 as draft specifications** in a public GitHub repository. These six core specs (normalization, routing format, primary name, reverse resolution, avatar, REST API) form the minimum viable standard. Solicit community feedback for 30 days.

2. **Build and open-source the reference indexer** — a Rust or Go implementation that indexes both `btcname` and `bnrp` inscriptions from a Bitcoin Core node with `txindex=1`. This is the single most important infrastructure component, as everything else depends on it.

3. **Implement the BNRP REST API as a reference resolver** — a lightweight HTTP server (Node.js or Go) that queries the reference indexer and serves the 8 standardized endpoints. Deploy at `resolver.bnrp.org` and publish the source code.

4. **Engage UniSat for primary name display integration** — UniSat already supports BtcName and has the largest Ordinals wallet user base. Adding primary name display (calling `reverseResolve()` on wallet addresses) is a high-visibility, low-effort integration that demonstrates immediate value.

5. **Publish the `@bnrp/sdk` npm package** — a TypeScript/JavaScript SDK implementing the client interface. This reduces integration cost for every subsequent wallet, dApp, and explorer that adopts BNRP.

### 22.2 Key Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Solution B as default primary name mechanism | Simplest, most secure (no proxy risk), no additional inscription cost. Solution A retained for metadata-rich use cases. |
| 2 | SLIP-44 `addr_{coinType}` for multichain addresses | Proven standard used by ENS (ENSIP-9); infinitely extensible; no protocol changes needed to add new chains. |
| 3 | Inscription ID as canonical identifier (never inscription number) | Inscription numbers can change during reindexing; inscription IDs are immutable by construction. |
| 4 | Bidirectional anti-spoofing for reverse resolution | Prevents the most dangerous attack vector (false identity → fund misdirection). Forward must match reverse, always. |
| 5 | `"p": "bnrp"` as new prefix while keeping `"p": "btcname"` support | Clean namespace for extended features without breaking any existing inscription. |

### 22.3 Wallet UX Recommendations

Wallets integrating BNRP SHOULD follow these UX guidelines to ensure consistent, secure, and user-friendly name display:

- **Show primary name prominently** where the address would normally appear. The name should be the primary identifier; the address is secondary. Example: display "alice.btc" in large text with the truncated address `bc1p5d7...9rus` below in smaller text.
- **Show avatar next to name** — 1:1 square crop, 32×32px minimum in compact view (address bar, transaction list), 128×128px in profile view. Use the initials fallback if avatar resolution fails.
- **Show "Verified" badge** (green checkmark or equivalent) when both forward and reverse resolution match. This indicates that `alice.btc` resolves to the displayed address AND that address has `alice.btc` as its primary name.
- **Show "Unverified" warning** when forward and reverse resolution don't match, or when the avatar is marked `avatar_unverified: true`. Use an orange or yellow badge.
- **Never show Unicode names without Punycode tooltip** — any name containing non-ASCII characters MUST show the Punycode representation on hover or in a subtitle. This prevents homoglyph attacks.
- **On first use of a .btc name, show the full inscription ID** as a confirmation step before sending funds. This allows users to verify the name's identity independently via a block explorer.
- **Display chain-specific address when selected** — when a user picks a chain to send on, show both the name and the resolved address for that chain. Example: "Sending to alice.btc on Ethereum: 0x71C7...976F".
- **Name resolution indicator** — while resolving, show a loading spinner on the name field. If resolution fails, show a clear error ("Name not found" or "No ETH address set for this name").

### 22.4 The One Thing That Matters Most

The single most important step in launching BNRP is **getting the normalization rules, routing inscription format, and primary name mechanism right before anything else**. These three specifications (BNRP-IP-01, 02, 03) define the foundational data structures that every future BNRP inscription will rely on. Once a user inscribes a `"p": "bnrp"` routing record on Bitcoin, that record exists forever — it cannot be edited, it cannot be deleted, and it cannot be reformatted. If the field names, validation rules, or normalization behavior are wrong, every inscription made under the wrong rules becomes permanently invalid or ambiguous. This is fundamentally different from ENS, where smart contract upgrades can change resolution logic for existing records. On Bitcoin, inscriptions are immutable. The protocol specification must be correct before the first inscription is made, because there are no second chances on the blockchain.

---

## Appendix A: Chain Coin Type Registry

The following table defines the supported chains and their BNRP identifiers. The `coinType` values follow SLIP-44 for non-EVM chains and the EVM encoding rule (`chainId | 0x80000000`) for EVM L2/L3 chains, consistent with ENSIP-9/11.

| Chain | SLIP-44 coinType | BNRP Field Name | Address Format | Address Regex |
|-------|-----------------|-----------------|----------------|---------------|
| Bitcoin (Taproot) | 0 | `btc_taproot` | `bc1p...` (62 chars) | `^bc1p[a-z0-9]{58}$` |
| Bitcoin (Native SegWit) | 0 | `btc_segwit` | `bc1q...` (42-62 chars) | `^bc1q[a-z0-9]{38,58}$` |
| Bitcoin (Legacy P2PKH) | 0 | `btc_legacy` | `1...` (25-34 chars) | `^1[a-km-zA-HJ-NP-Z1-9]{25,34}$` |
| Bitcoin (P2SH) | 0 | `btc_p2sh` | `3...` (25-34 chars) | `^3[a-km-zA-HJ-NP-Z1-9]{25,34}$` |
| Bitcoin (Lightning) | 0 | `btc_lightning` | `lnbc...` / `LNURL...` / `user@domain` | (varies) |
| Ethereum | 60 | `eth` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Solana | 501 | `sol` | Base58 (32-44 chars) | `^[1-9A-HJ-NP-Za-km-z]{32,44}$` |
| Polygon (PoS) | 2147483785 | `addr_2147483785` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Arbitrum One | 2189526273 | `addr_2189526273` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Base | 2155872277 | `addr_2155872277` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Optimism | 2147483658 | `addr_2147483658` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Avalanche C-Chain | 2147526762 | `addr_2147526762` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| BNB Smart Chain | 2147483708 | `addr_2147483708` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Fantom | 2147484098 | `addr_2147484098` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| zkSync Era | 2147483972 | `addr_2147483972` | `0x...` (42 chars) | `^0x[0-9a-fA-F]{40}$` |
| Litecoin | 2 | `ltc` | `L...` / `M...` / `ltc1...` | (varies) |
| Dogecoin | 3 | `doge` | `D...` | `^D[5-9A-HJ-NP-U][1-9A-HJ-NP-Za-km-z]{32}$` |
| Cosmos Hub | 118 | `atom` | `cosmos1...` | `^cosmos1[a-z0-9]{38}$` |
| Ripple | 144 | `xrp` | `r...` | `^r[1-9A-HJ-NP-Za-km-z]{24,34}$` |
| Cardano | 1815 | `ada` | `addr1...` | (varies) |
| Polkadot | 354 | `dot` | `1...` | (varies) |
| Tron | 195 | `trx` | `T...` | `^T[a-zA-Z0-9]{33}$` |
| (Future chain) | Per SLIP-44 or `chainId \| 0x80000000` | `addr_{coinType}` | Per chain spec | Per chain spec |

**EVM L2 coinType Calculation:**

For any EVM-compatible chain, the coinType is computed as:

```
coinType = chainId | 0x80000000
```

Examples:
- Polygon: chainId 137 → `137 | 0x80000000` = `2147483785`
- Base: chainId 8453 → `8453 | 0x80000000` = `2155872277`
- Arbitrum One: chainId 42161 → `42161 | 0x80000000` = `2189526273`
- Optimism: chainId 10 → `10 | 0x80000000` = `2147483658`
- BNB Smart Chain: chainId 56 → `56 | 0x80000000` = `2147483704`

**Backward Compatibility Note:** The legacy BtcName field names (`eth_address`, `matic_address`, `sol_address`) are recognized by BNRP resolvers as aliases for the corresponding coinType-based fields. When both are present, the coinType-based field takes precedence.

---

## Appendix B: BNRP-IP Template

The following template MUST be used for all BNRP Improvement Proposals submitted to the BNRP specification repository.

```markdown
---
bnrp-ip: [number]
title: [Concise title]
description: [One-line description]
author: [Author name(s) and/or handle(s)]
status: Draft | Review | Final | Obsolete
created: [YYYY-MM-DD]
requires: [list of BNRP-IP numbers this depends on, if any]
replaces: [list of BNRP-IP numbers this obsoletes, if any]
---

# BNRP-IP-[number]: [Title]

## Abstract

A concise (~200 word) description of the proposal. This section should
be sufficient for a reader to determine if the BNRP-IP is relevant
to them.

## Motivation

Why is this proposal needed? What problem does it solve? What gap
in the current BNRP specification does it address?

## Specification

The technical specification. This section must be detailed enough
that an independent implementer could build a conformant implementation
from this section alone.

### Data Structures

Define any new inscription formats, JSON schemas, or data structures.
Include full JSON Schema definitions (Draft-07) for any new record types.

### Resolution Rules

Define how the new feature interacts with existing resolution logic.
Include pseudocode for any new resolution paths.

### Validation Rules

Define all validation rules: field formats, length limits, allowed
characters, required fields, and error conditions.

### API Changes

If this BNRP-IP adds or modifies REST API endpoints, define them here
with full request/response examples.

## Rationale

Why were these particular design choices made? What alternatives were
considered and why were they rejected?

## Backward Compatibility

How does this proposal interact with existing BtcName inscriptions
and earlier BNRP-IP specifications? Are any breaking changes introduced?

## Security Considerations

What security implications does this proposal have? What attack vectors
does it introduce or mitigate?

## Test Vectors

Provide at least three test vectors:
1. A valid example
2. An invalid example that must be rejected
3. An edge case

## Reference Implementation

Link to or include a reference implementation. This may be pseudocode
for the initial Draft status, but must be working code for Final status.

## Copyright

Copyright and related rights waived via CC0 1.0 Universal.
```

**BNRP-IP Status Lifecycle:**

```
  Draft ──────► Review ──────► Final
    │              │              │
    │              │              ▼
    │              │          (Immutable)
    ▼              ▼
  Withdrawn    Rejected
                   │
                   ▼
               Obsolete
              (replaced by
              newer BNRP-IP)
```

**Status Definitions:**

| Status | Meaning |
|--------|---------|
| **Draft** | Initial submission. Open for community feedback. May change substantially. |
| **Review** | Considered stable by author. Undergoing formal review by at least 2 independent reviewers. |
| **Final** | Accepted as part of the BNRP specification. Immutable — any changes require a new BNRP-IP. |
| **Withdrawn** | Withdrawn by the author before reaching Final status. |
| **Rejected** | Rejected during Review due to technical issues, security concerns, or lack of consensus. |
| **Obsolete** | Superseded by a newer BNRP-IP. The replacement BNRP-IP must reference the obsoleted one. |

**Review Criteria:**

1. **Technical correctness** — the specification must be implementable and internally consistent.
2. **Security** — the proposal must not introduce unmitigated attack vectors.
3. **Backward compatibility** — the proposal must not break existing valid inscriptions.
4. **Completeness** — the specification must include full schemas, pseudocode, and test vectors.
5. **Consensus** — at least two independent reviewers and one indexer implementer must approve.

---

## Appendix C: Reference Implementation Notes

### C.1 Reference Indexer Architecture

A conformant BNRP reference indexer MUST implement the following components:

**1. Bitcoin Node Interface**
- Connect to a Bitcoin Core node via RPC (`bitcoind` with `txindex=1` and `rest=1`).
- Subscribe to new blocks via ZMQ (`zmqpubhashblock`) or polling (`getbestblockhash`).
- For each new block, retrieve all transactions and scan for Ordinals inscription envelopes.

**2. Ordinals Parser**
- Parse tapscript witness data to detect inscription envelopes.
- Extract inscription content (the JSON payload) and content type.
- Compute the inscription ID as `{txid}i{index}` where `txid` is the reveal transaction ID and `index` is the inscription's position within that transaction.
- Track the inscription's sat assignment using ordinal theory (first-in-first-out sat tracking).

**3. BNRP Record Parser**
- For each inscription with `content-type: text/plain` or `content-type: application/json`:
  - Attempt to parse as JSON.
  - Check for `"p": "bnrp"` or `"p": "btcname"` protocol identifier.
  - Validate the `"op"` field against known operations: `routing`, `reverse`, `primary-name`, `profile`, `content`, `bitmap`.
  - Validate all fields against the corresponding JSON Schema (see Section 6).
  - Store the parsed record in the indexer database, keyed by name (normalized) and operation type.

**4. Ownership Tracker**
- For each inscription, track the current UTXO that holds the inscription sat.
- On each new block, update ownership by scanning for spent UTXOs and tracing the inscription sat to its new output.
- Maintain a mapping: `inscription_id → current_utxo → current_address`.

**5. Name Resolution Index**
- Maintain a mapping: `normalized_name → [inscription_ids]` (all inscriptions that reference this name).
- For routing records: filter to only those inscribed by the current owner of the name inscription.
- Apply "Last is New" rule: the most recent valid routing record from the owner is authoritative.

**6. Primary Name Index**
- For Solution B: monitor all transfer events where `from_address == to_address` for supported name protocols.
- Maintain a mapping: `address → primary_name` (most recent self-transfer wins).
- For Solution A: monitor `primary-name` inscriptions and verify minter matches current owner.

**7. Confirmation Tracking**
- Do not consider any inscription or transfer final until it has >= 3 confirmations.
- Maintain a "pending" queue for records with < 3 confirmations.
- On reorg: roll back all affected records and re-process from the last common ancestor block.

### C.2 Database Schema (Recommended)

```sql
-- Core tables for a BNRP reference indexer

CREATE TABLE inscriptions (
    inscription_id    TEXT PRIMARY KEY,        -- {txid}i{index}
    inscription_number BIGINT,                 -- may be unstable
    content_type      TEXT NOT NULL,
    content           JSONB,
    reveal_txid       TEXT NOT NULL,
    reveal_block      BIGINT NOT NULL,
    reveal_timestamp  BIGINT NOT NULL,
    minter_address    TEXT NOT NULL,           -- immutable
    current_utxo      TEXT,                    -- {txid}:{vout}
    current_owner     TEXT,                    -- derived from current_utxo
    confirmations     INTEGER DEFAULT 0,
    is_valid_bnrp     BOOLEAN DEFAULT FALSE,
    created_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE names (
    name_normalized   TEXT PRIMARY KEY,        -- normalized name (e.g., "alice.btc")
    name_raw          TEXT NOT NULL,           -- original inscription text
    name_inscription_id TEXT NOT NULL REFERENCES inscriptions(inscription_id),
    current_owner     TEXT,                    -- current owner address
    protocol          TEXT NOT NULL,           -- "btc", "sats", "unisat", "x", "fb"
    created_block     BIGINT NOT NULL,
    updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE routing_records (
    id                SERIAL PRIMARY KEY,
    name_normalized   TEXT NOT NULL REFERENCES names(name_normalized),
    inscription_id    TEXT NOT NULL REFERENCES inscriptions(inscription_id),
    record_data       JSONB NOT NULL,          -- full routing record
    inscribed_by      TEXT NOT NULL,            -- address that inscribed this record
    block_height      BIGINT NOT NULL,
    is_current        BOOLEAN DEFAULT FALSE,   -- true for the latest valid record
    created_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE profile_records (
    id                SERIAL PRIMARY KEY,
    name_normalized   TEXT NOT NULL REFERENCES names(name_normalized),
    inscription_id    TEXT NOT NULL REFERENCES inscriptions(inscription_id),
    record_data       JSONB NOT NULL,
    inscribed_by      TEXT NOT NULL,
    block_height      BIGINT NOT NULL,
    is_current        BOOLEAN DEFAULT FALSE,
    created_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE primary_names (
    address           TEXT PRIMARY KEY,         -- wallet address
    name_normalized   TEXT NOT NULL,            -- primary name
    method            TEXT NOT NULL,            -- "solution_a" or "solution_b"
    block_height      BIGINT NOT NULL,          -- block where this became primary
    inscription_id    TEXT,                     -- for Solution A: the primary-name inscription
    transfer_txid     TEXT,                     -- for Solution B: the self-transfer tx
    verified          BOOLEAN DEFAULT FALSE,    -- forward/reverse match verified
    updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE bitmap_records (
    id                SERIAL PRIMARY KEY,
    name_normalized   TEXT NOT NULL REFERENCES names(name_normalized),
    inscription_id    TEXT NOT NULL REFERENCES inscriptions(inscription_id),
    district          TEXT,
    parcel            TEXT,
    record_data       JSONB NOT NULL,
    block_height      BIGINT NOT NULL,
    is_current        BOOLEAN DEFAULT FALSE,
    created_at        TIMESTAMP DEFAULT NOW()
);

-- Indexes for efficient resolution
CREATE INDEX idx_names_owner ON names(current_owner);
CREATE INDEX idx_routing_name ON routing_records(name_normalized, is_current);
CREATE INDEX idx_profile_name ON profile_records(name_normalized, is_current);
CREATE INDEX idx_primary_name ON primary_names(address);
CREATE INDEX idx_primary_name_reverse ON primary_names(name_normalized);
CREATE INDEX idx_inscriptions_owner ON inscriptions(current_owner);
CREATE INDEX idx_inscriptions_block ON inscriptions(reveal_block);
```

### C.3 Indexer Sync Algorithm

```pseudocode
function syncIndexer():
    lastProcessedBlock = db.getLastProcessedBlock()
    currentTip = bitcoinRPC.getBestBlockHash()
    currentHeight = bitcoinRPC.getBlockCount()

    // Check for reorgs
    for height = lastProcessedBlock down to max(lastProcessedBlock - 10, 0):
        storedHash = db.getBlockHash(height)
        chainHash = bitcoinRPC.getBlockHash(height)
        if storedHash != chainHash:
            log("Reorg detected at height " + height)
            db.rollbackToBlock(height - 1)
            lastProcessedBlock = height - 1
            break

    // Process new blocks
    for height = lastProcessedBlock + 1 to currentHeight:
        block = bitcoinRPC.getBlock(height, verbosity=2)

        for tx in block.transactions:
            // Check for inscription reveals
            inscriptions = parseInscriptions(tx)
            for insc in inscriptions:
                processInscription(insc, height, block.time)

            // Check for inscription transfers (UTXO movements)
            transfers = detectInscriptionTransfers(tx)
            for transfer in transfers:
                processTransfer(transfer, height)

        // Update confirmation counts
        db.incrementConfirmations(height)

        // Activate records that just reached 3 confirmations
        newlyConfirmed = db.getRecordsAtConfirmation(3, height - 2)
        for record in newlyConfirmed:
            activateRecord(record)

        db.setLastProcessedBlock(height)

function processInscription(insc, blockHeight, blockTime):
    // Store raw inscription
    db.insertInscription(
        inscription_id = insc.txid + "i" + insc.index,
        content_type = insc.contentType,
        content = insc.content,
        reveal_txid = insc.txid,
        reveal_block = blockHeight,
        reveal_timestamp = blockTime,
        minter_address = insc.minterAddress,
        current_utxo = insc.txid + ":" + insc.outputIndex,
        current_owner = insc.minterAddress,
        confirmations = 0
    )

    // Attempt BNRP record parsing
    if insc.contentType in ["text/plain", "application/json"]:
        try:
            json = parseJSON(insc.content)
            if json.p in ["bnrp", "btcname"]:
                validateAndStoreRecord(json, insc, blockHeight)
        catch:
            // Not a valid BNRP record — ignore
            pass

function processTransfer(transfer, blockHeight):
    // Update inscription ownership
    db.updateInscriptionOwner(
        inscription_id = transfer.inscriptionId,
        new_utxo = transfer.newUtxo,
        new_owner = transfer.newOwnerAddress
    )

    // Check for Solution B primary name (self-transfer)
    if transfer.fromAddress == transfer.toAddress:
        if blockHeight >= 840000:
            name = db.getNameByInscriptionId(transfer.inscriptionId)
            if name != null and len(name) <= 128:
                if isValidAddressType(transfer.toAddress):  // bc1q or bc1p
                    db.setPrimaryName(
                        address = transfer.toAddress,
                        name = name,
                        method = "solution_b",
                        block_height = blockHeight,
                        transfer_txid = transfer.txid
                    )

    // Check if this transfer affects any routing records
    db.revalidateRoutingRecords(transfer.inscriptionId)
```

### C.4 Testing Recommendations

A conformant BNRP implementation SHOULD include the following test suites:

1. **Name Normalization Tests** — verify all normalization rules from Section 8.1, including Unicode edge cases, maximum length enforcement, and reserved label detection.

2. **Schema Validation Tests** — verify that all six JSON schemas (Section 6) correctly accept valid records and reject invalid records. Test boundary conditions (maximum field lengths, missing required fields, invalid URI schemes).

3. **Resolution Tests** — verify forward resolution, reverse resolution, avatar resolution, content resolution, and bitmap resolution against known test vectors. Include tests for the anti-spoofing guarantee (Section 9.1).

4. **Ownership Tests** — verify that routing records inscribed by non-owners are rejected, that ownership changes correctly invalidate stale records, and that the "Last is New" rule is applied correctly.

5. **Reorg Tests** — simulate a chain reorganization and verify that the indexer correctly rolls back and re-processes affected records.

6. **Confirmation Tests** — verify that records with fewer than 3 confirmations are not returned by the resolver API, and that they become active exactly at 3 confirmations.

7. **Backward Compatibility Tests** — verify that existing `"p": "btcname"` inscriptions are correctly indexed and resolved alongside `"p": "bnrp"` inscriptions.

### C.5 Performance Targets

A production BNRP resolver SHOULD meet the following performance targets:

| Operation | Target Latency (p99) | Target Throughput |
|-----------|----------------------|-------------------|
| Forward resolution (`/resolve/{name}`) | < 50ms | 1000 req/s |
| Reverse resolution (`/reverse/{address}`) | < 50ms | 1000 req/s |
| Avatar resolution (`/avatar/{name}`) | < 100ms | 500 req/s |
| Content resolution (`/content/{name}`) | < 50ms | 1000 req/s |
| Bitmap resolution (`/bitmap/{name}`) | < 50ms | 1000 req/s |
| Health check (`/health`) | < 10ms | 5000 req/s |
| Full indexer sync (mainnet) | < 48 hours | N/A |
| Incremental sync (per block) | < 5 seconds | N/A |

---

*End of Document*

*BTC Name Resolution Protocol (BNRP) — Version 0.1 Draft Specification*
*Copyright and related rights waived via CC0 1.0 Universal.*
*This document is published as an open standard. Contributions, feedback, and independent implementations are welcome.*
*Repository: https://github.com/bnrp/bnrp-spec (proposed)*
*Discussion: https://discord.gg/bnrp (proposed)*
