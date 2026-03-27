# BNRP-IP-06: REST API Specification v1

| Field | Value |
|-------|-------|
| **BNRP-IP** | 06 |
| **Title** | REST API Specification v1 |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | BNRP-IP-01, BNRP-IP-02, BNRP-IP-03, BNRP-IP-04, BNRP-IP-05 |
| **Replaces** | — |

---

## Abstract

This proposal defines the canonical HTTP REST API that any BNRP-compliant resolver MUST implement. Any party — a wallet team, an indexer operator, a blockchain explorer, or an individual developer — can run a compliant resolver by implementing this spec. Multiple independent resolvers can coexist, and clients MAY query more than one for cross-verification.

---

## Motivation

BtcName currently exposes a private API (e.g., `wallet-api.unisat.io/v5/address/search?domain=`) that is undocumented, single-vendor, and not officially public. An open API standard enables ecosystem-wide interoperability without dependence on any single provider.

---

## Specification

### 1. Base URL

Each resolver exposes its API at a base URL. The path prefix for all v1 endpoints is `/v1/`.

Example: `https://resolver.bnrp.io/v1/`

Resolvers MUST include a `BNRP-Resolver-Version` response header with the value `1` for all endpoints.

### 2. Common Response Fields

All successful responses MUST include:

| Field | Type | Description |
|-------|------|-------------|
| `resolved_at` | ISO8601 string | UTC timestamp when resolution was performed |
| `name` | string | The normalized name this response is for (where applicable) |

All error responses MUST use standard HTTP status codes and include:

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description"
}
```

### 3. Endpoints

---

#### `GET /v1/resolve/{name}`

Resolve a `.btc` name to its full record.

**Parameters:**
- `name` (path): The `.btc` name to resolve (e.g., `alice.btc`)

**Behavior:**
1. Normalize `name` per BNRP-IP-01
2. Verify inscription ownership (>= 3 confirmations)
3. Fetch most recent valid routing/profile inscriptions
4. Return all fields

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "owner": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "inscription_id": "9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "addresses": {
    "btc_taproot": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
    "btc_segwit":  "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
    "btc_lightning": "lnurlp1qqpsqqqqqqqqqqqqqqqqqqqqqqqqqqqqxkx9h2",
    "eth":   "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "base":  "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "arb":   "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "sol":   "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU"
  },
  "profile": {
    "display":      "Alice",
    "description":  "Bitcoin artist and builder.",
    "avatar":       "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
    "avatar_verified": true,
    "url":          "https://alice.xyz",
    "com.twitter":  "aliceonbtc",
    "com.github":   "alice-btc"
  },
  "content": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "bitmap": {
    "district": "840000",
    "parcel":   null,
    "world":    null,
    "coordinates": null
  },
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

**Error Responses:**

| Status | Error Code | Meaning |
|--------|-----------|---------|
| 400 | `INVALID_NAME` | Name fails normalization |
| 404 | `NAME_NOT_FOUND` | No inscription found for this name |
| 422 | `INSUFFICIENT_CONFIRMATIONS` | Inscription exists but < 3 confirmations |

---

#### `GET /v1/reverse/{address}`

Reverse resolve a Bitcoin address to its primary `.btc` name.

**Parameters:**
- `address` (path): Bitcoin address (`bc1p...`, `bc1q...`, `1...`, `3...`)

**Success Response: 200 OK (name found)**

```json
{
  "address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "name": "alice.btc",
  "verified": true,
  "mechanism": "solution_b",
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

**Success Response: 200 OK (no primary name)**

```json
{
  "address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "name": null,
  "verified": false,
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

Note: Returns `200` (not `404`) when no name is found — this is a valid query result, not an error.

**Error Responses:**

| Status | Error Code | Meaning |
|--------|-----------|---------|
| 400 | `INVALID_ADDRESS` | Address format unrecognized |

---

#### `GET /v1/address/{name}`

Get a single chain address for a name.

**Parameters:**
- `name` (path): The `.btc` name
- `chain` (query, optional): SLIP-44 coinType as integer. Defaults to `0` (BTC Taproot).

**Examples:**
- `/v1/address/alice.btc` → BTC taproot address
- `/v1/address/alice.btc?chain=60` → ETH address
- `/v1/address/alice.btc?chain=2155872277` → Base address

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "chain": 0,
  "address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

**Success Response: 200 OK (chain not set)**

```json
{
  "name": "alice.btc",
  "chain": 60,
  "address": null,
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

---

#### `GET /v1/profile/{name}`

Get profile/metadata fields for a name.

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "display":     "Alice",
  "description": "Bitcoin artist and builder.",
  "avatar":      "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "avatar_verified": true,
  "url":         "https://alice.xyz",
  "email":       "alice@alice.xyz",
  "com.twitter": "aliceonbtc",
  "com.github":  "alice-btc",
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

---

#### `GET /v1/avatar/{name}`

Get the resolved avatar image URL for a name (ready to use in an `<img>` tag).

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "avatar_uri": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "avatar_url": "https://ordinals.com/content/9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "mime_type": "image/png",
  "verified": true,
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

- `avatar_uri`: the raw field value from the inscription
- `avatar_url`: the resolved, directly-fetchable image URL (ready for `<img src>`)
- `verified`: whether inscription owner == name owner

**Success Response: 200 OK (no avatar)**

```json
{
  "name": "alice.btc",
  "avatar_uri": null,
  "avatar_url": null,
  "mime_type": null,
  "verified": false,
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

---

#### `GET /v1/content/{name}`

Get the web/content target for a name (IPFS, IPNS, Arweave, or HTTP URL).

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "content_uri": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "content_url": "https://ipfs.io/ipfs/QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

---

#### `GET /v1/bitmap/{name}`

Get bitmap/metaverse record for a name.

**Success Response: 200 OK**

```json
{
  "name": "alice.btc",
  "bitmap": {
    "district":    "840000",
    "parcel":      "42",
    "world":       null,
    "coordinates": "x:100,y:200",
    "avatar":      null,
    "profile_url": null
  },
  "resolved_at": "2026-03-26T20:00:00Z"
}
```

---

#### `GET /v1/health`

Resolver health and status check.

**Success Response: 200 OK**

```json
{
  "status": "ok",
  "version": "1",
  "indexer_block_height": 895432,
  "indexer_lag_blocks": 0,
  "uptime_seconds": 86400,
  "resolver_id": "resolver.bnrp.io",
  "timestamp": "2026-03-26T20:00:00Z"
}
```

`indexer_lag_blocks` indicates how many blocks behind the indexer is relative to Bitcoin tip. Values > 6 indicate the resolver may be returning stale data; clients SHOULD warn users.

---

### 4. Rate Limiting

Resolvers SHOULD implement rate limiting. They MUST return `429 Too Many Requests` with a `Retry-After` header when rate limits are exceeded.

### 5. CORS

Resolvers MUST include `Access-Control-Allow-Origin: *` to allow browser-based clients.

### 6. Caching Headers

Resolvers SHOULD include `Cache-Control: max-age=600` (10 minutes) on resolution endpoints. The `/v1/health` endpoint SHOULD use `Cache-Control: no-cache`.

### 7. Multi-Resolver Usage (Client Pattern)

Clients MAY query multiple resolvers for cross-verification. The recommended pattern for high-value operations (e.g., sending funds):

```typescript
async function verifiedResolve(name: string): Promise<string | null> {
  const resolvers = [
    'https://resolver1.bnrp.io',
    'https://resolver2.bnrp.io'
  ];

  const results = await Promise.allSettled(
    resolvers.map(r => fetch(`${r}/v1/address/${name}`).then(r => r.json()))
  );

  const addresses = results
    .filter(r => r.status === 'fulfilled')
    .map(r => (r as any).value.address)
    .filter(Boolean);

  // Require agreement from all available resolvers
  const unique = [...new Set(addresses)];
  if (unique.length === 1) return unique[0];

  // Disagreement — warn user, do not auto-send
  throw new Error('Resolver disagreement: ' + unique.join(' vs '));
}
```

---

## SDK Reference (TypeScript)

The following TypeScript interfaces define the full BNRP client SDK. A reference implementation is in `/sdk-reference/`.

```typescript
// Types
interface BNRPAddresses {
  btc_taproot?: string;
  btc_segwit?: string;
  btc_p2sh?: string;
  btc_legacy?: string;
  btc_lightning?: string;
  eth?: string;
  base?: string;
  arb?: string;
  sol?: string;
  matic?: string;
  [key: string]: string | undefined; // chain_{coinType}
}

interface BNRPProfile {
  display?: string;
  description?: string;
  avatar?: string;
  avatar_verified?: boolean;
  url?: string;
  email?: string;
  'com.twitter'?: string;
  'com.github'?: string;
}

interface BNRPBitmapRecord {
  district?: string;
  parcel?: string;
  world?: string;
  coordinates?: string;
  avatar?: string;
  profile_url?: string;
}

interface BNRPRecord {
  name: string;
  owner: string;
  inscription_id: string;
  addresses: BNRPAddresses;
  profile: BNRPProfile;
  content?: string;
  bitmap?: BNRPBitmapRecord;
  resolved_at: string;
}

interface BNRPReverseResult {
  address: string;
  name: string | null;
  verified: boolean;
  mechanism?: 'solution_a' | 'solution_b' | 'explicit';
  resolved_at: string;
}

interface BNRPAvatarResult {
  avatar_uri: string | null;
  avatar_url: string | null;
  mime_type: string | null;
  verified: boolean;
}

// Client interface
interface BNRPClient {
  // Full record resolution
  resolve(name: string): Promise<BNRPRecord>;

  // Reverse resolution
  reverseResolve(address: string): Promise<BNRPReverseResult>;

  // Single chain address
  getAddress(name: string, coinType?: number): Promise<string | null>;

  // Profile/metadata
  getProfile(name: string): Promise<BNRPProfile>;

  // Avatar (returns display-ready URL)
  getAvatar(name: string): Promise<BNRPAvatarResult>;

  // Web/content target
  getContent(name: string): Promise<string | null>;

  // Bitmap/metaverse record
  getBitmap(name: string): Promise<BNRPBitmapRecord | null>;

  // Resolver health
  health(): Promise<{ status: string; indexer_block_height: number }>;
}

// Factory
function createBNRPClient(config: {
  resolver: string | string[]; // single URL or array for multi-resolver
  timeout?: number;            // ms, default 5000
  cache?: boolean;             // default true (10-min TTL)
}): BNRPClient;
```

---

## Backward Compatibility

This API has no predecessor (it is new). However, the UniSat internal endpoint `wallet-api.unisat.io/v5/address/search?domain=` performs a subset of what `/v1/address/{name}` defines. Wallets currently using that endpoint can migrate to BNRP-IP-06 resolvers without changing their core name-to-address logic.

---

## Security Considerations

**Resolver trust:** Any party can run a resolver. A malicious resolver could return false addresses, leading users to send funds to the wrong address. Clients SHOULD use the multi-resolver pattern (Section 7) for high-value operations.

**HTTPS only:** Resolvers MUST be served over HTTPS. Clients MUST refuse to use resolvers over plain HTTP.

**Response signing (future):** A future BNRP-IP may define an optional response signing scheme where resolvers sign responses with a known public key, allowing clients to verify authenticity without trusting TLS alone.

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
