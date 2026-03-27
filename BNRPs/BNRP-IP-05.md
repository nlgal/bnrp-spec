# BNRP-IP-05: Avatar URI Schemes

| Field | Value |
|-------|-------|
| **BNRP-IP** | 05 |
| **Title** | Avatar URI Schemes |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | BNRP-IP-01, BNRP-IP-02 |
| **Replaces** | — |

---

## Abstract

This proposal defines the canonical avatar field format for `.btc` names, the set of supported URI schemes, resolution behavior for each scheme, Ordinals-native ownership verification, image safety rules, fallback behavior, and wallet display guidelines. BNRP avatars prefer Ordinals-native images when available.

---

## Motivation

BtcName's existing avatar field accepts only a raw inscription ID (e.g., `"xxxi0"`) with no URI scheme, no fallback support, no ownership verification, and no safety rules. ENS's ENSIP-12 defines a mature avatar standard including multiple URI schemes, NFT ownership verification, and fallback behavior. BNRP-IP-05 brings equivalent maturity to the Bitcoin ecosystem while making Ordinals inscriptions the preferred avatar source.

---

## Specification

### 1. Avatar Field Location

The `avatar` field MAY appear in:
- A BNRP routing inscription (`"op": "routing"`)
- A BNRP primary-name inscription (`"op": "primary-name"`)
- A BNRP profile inscription (`"op": "profile"`, defined in BNRP-IP-07)

When multiple records contain an `avatar` field for the same name, precedence is:
1. Profile inscription (most specific)
2. Primary-name inscription
3. Routing inscription

Within each record type, the most recent (highest block height) valid inscription wins.

### 2. Supported URI Schemes

All BNRP-compliant resolvers and wallets MUST support the following URI schemes:

| Scheme | Format | Example | Resolution |
|--------|--------|---------|------------|
| `ord:` | `ord:{inscriptionId}` | `ord:9ba506...641i0` | Fetch from Ordinals indexer; verify MIME is image/* |
| `ord-num:` | `ord-num:{number}` | `ord-num:35648` | Resolve via inscription number (see Section 4) |
| `ipfs:` | `ipfs://{CIDv0orv1}` | `ipfs://QmNtHN7...` | Resolve via IPFS gateway |
| `ipns:` | `ipns://{key}` | `ipns://k51qzi5...` | Resolve via IPNS |
| `https:` | `https://{url}` | `https://example.com/avatar.png` | Direct HTTP(S) fetch |
| `data:` | `data:{mime};base64,{data}` | `data:image/png;base64,...` | Inline image |

Implementations MAY support additional schemes. Unknown schemes MUST be ignored and treated as if no avatar were set (triggering fallback behavior).

### 3. Ordinals Avatar: `ord:` Scheme

The `ord:` scheme is the preferred avatar source in BNRP. It references an Ordinals inscription by its inscription ID.

**Format:** `ord:{txid}i{index}`
**Example:** `ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0`

#### 3.1 Resolution Procedure

```pseudocode
function resolveOrdAvatar(inscriptionId: string) -> imageURL | null:
  // Step 1: Query Ordinals indexer for inscription metadata
  meta = ordinals.getInscription(inscriptionId)
  if meta == null:
    return null  // Inscription not found

  // Step 2: Verify content type is an image
  if not meta.contentType.startsWith("image/"):
    return null  // Not an image — fallback

  // Step 3: Check supported MIME types
  if meta.contentType not in SUPPORTED_MIME_TYPES:
    return null  // Unsupported image type — fallback

  // Step 4: For SVG — sanitize before rendering
  if meta.contentType == "image/svg+xml":
    return sanitizedSVGUrl(inscriptionId)  // Serve from sandboxed origin

  // Step 5: Return content URL
  return ordinals.getContentUrl(inscriptionId)
  // e.g., https://ordinals.com/content/{inscriptionId}
```

#### 3.2 Ownership Verification

When an `ord:` avatar is used in a routing or profile inscription:

- Resolvers MUST check that the inscription owner of the avatar == the current owner of the `.btc` name inscription.
- If ownership does not match, the avatar is treated as **unverified**.
- Resolvers MUST include an `avatar_verified: false` flag in their response when ownership cannot be confirmed.
- The `avatar` field MAY include the suffix `#unverified` to explicitly opt out of ownership verification:
  - `"avatar": "ord:...i0#unverified"` — display without ownership check, show "unverified" badge

Rationale: This prevents a name owner from falsely claiming another person's valuable Ordinal as their avatar.

#### 3.3 Backward Compatibility: Raw Inscription ID

BtcName routing inscriptions contain `"avatar": "xxxi0"` (a raw inscription ID without the `ord:` scheme prefix). Resolvers MUST treat a bare inscription ID in the avatar field as an `ord:` URI:

```
"avatar": "9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
→ treated as →
"avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
```

### 4. Inscription Number Avatar: `ord-num:` Scheme

`ord-num:{number}` references an inscription by its sequential inscription number (e.g., `ord-num:35648`).

**Warning:** Inscription numbers are not permanently stable. The original `ord` software developer has proposed making inscription numbers unstable (reprioritizing the ordering system). Resolvers MUST warn clients when returning avatars resolved via `ord-num:` that the reference may break in the future.

Resolvers SHOULD resolve `ord-num:` to an inscription ID internally and cache the `ord:` equivalent. The `ord-num:` scheme is DEPRECATED for new inscriptions; use `ord:` instead.

### 5. IPFS and IPNS Avatars

For `ipfs://{CID}` and `ipns://{key}`:

- Resolvers without native IPFS support MAY rewrite to a public IPFS gateway (e.g., `https://ipfs.io/ipfs/{CID}` or `https://cloudflare-ipfs.com/ipfs/{CID}`)
- Resolvers SHOULD support at least two gateway fallbacks
- The content at the CID MUST be a direct image file (not HTML, not metadata JSON)
- MIME type verification applies (see Section 7)

### 6. HTTPS Avatars

For `https://{url}`:
- The URL MUST resolve directly to an image file
- It MUST NOT resolve to an HTML page containing an image
- Redirect chains (HTTP 301/302) are allowed, up to a maximum of 3 redirects
- The final response MUST have a `Content-Type` header matching a supported image MIME type

### 7. Supported MIME Types

| MIME Type | Support Level | Notes |
|-----------|--------------|-------|
| `image/jpeg` | MUST | |
| `image/png` | MUST | |
| `image/gif` | MUST | Animated GIF allowed |
| `image/webp` | MUST | |
| `image/svg+xml` | SHOULD | MUST be sanitized before rendering; serve from sandboxed origin |
| `image/avif` | MAY | |
| `image/bmp` | MAY | |

All other MIME types MUST be rejected. An avatar field pointing to HTML, JavaScript, or any non-image content MUST be ignored and treated as if no avatar were set.

### 8. Fallback Chain

When avatar resolution fails at any step, the following fallback chain MUST be applied in order:

1. **Primary scheme** (e.g., `ord:`) — try first
2. **Secondary scheme** (if `avatar_fallback` field is present in the inscription) — try second
3. **Generated avatar** — derive a deterministic color and initials-based avatar from the name (e.g., `A` for `alice.btc`, background color derived from hash of name). This is always available.

The `avatar_fallback` field is an optional extension in routing/profile inscriptions:

```json
{
  "avatar": "ord:9ba506...i0",
  "avatar_fallback": "ipfs://QmNtHN7..."
}
```

### 9. Image Safety Rules

Implementations MUST enforce the following:

1. **SVG sanitization:** SVG content MUST have script tags, event handlers (`onload`, `onclick`, etc.), and external resource references (`href`, `src` pointing to external domains) stripped before rendering. Implementations SHOULD serve SVGs from a separate sandboxed origin.

2. **Content Security Policy:** Avatars served via gateway MUST be served with `Content-Security-Policy: default-src 'none'` (or equivalent sandbox headers).

3. **Size limits:** Implementations SHOULD reject images larger than 5MB for HTTPS/IPFS avatars. Ordinals inscriptions are inherently small (Bitcoin block size constraint) and exempt.

4. **No redirect to HTML:** Any redirect chain that results in an HTML page MUST be treated as a resolution failure.

5. **Timeout:** Avatar HTTP requests MUST time out after 5 seconds. Treat timeout as resolution failure.

### 10. Display Guidelines for Wallets

| Context | Recommended Size |
|---------|-----------------|
| Compact (address list) | 32×32px |
| Standard (send/receive dialog) | 64×64px |
| Profile view | 128×128px or larger |
| Full profile page | 400×400px max |

Display MUST be:
- **1:1 aspect ratio** (square crop from center if source is non-square)
- No visible blank space
- Rounded corners (circular or rounded-square) are recommended
- Show an "unverified" badge when `avatar_verified: false`
- On hover/tap, show inscription ID or source URL

### 11. Full Examples

#### Example 1: Ordinal inscription avatar (ownership verified)

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0"
}
```

Resolution: fetch `https://ordinals.com/content/9ba5066...641i0`, verify `image/png`, verify alice.btc inscription owner == avatar inscription owner.

#### Example 2: IPFS fallback avatar

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "avatar": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio"
}
```

Resolution: `https://ipfs.io/ipfs/QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio`, no ownership verification required for IPFS avatars.

#### Example 3: Ordinal avatar with IPFS fallback

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "avatar_fallback": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio"
}
```

---

## Backward Compatibility

BtcName inscriptions with `"avatar": "{inscriptionId}"` (bare inscription ID, no scheme prefix) are treated as `ord:{inscriptionId}`. No migration needed.

---

## Security Considerations

**Avatar abuse:** A malicious actor may point an avatar field at harmful, NSFW, or legally problematic content. Implementations MUST enforce MIME type gating (Section 7) and image size limits (Section 9). Platforms MAY implement additional content moderation layers.

**SVG XSS:** SVG files can contain JavaScript and external resource references. Implementations MUST sanitize SVG content before rendering (Section 9.1).

**Ownership impersonation:** Without ownership verification, a user could claim another person's valuable Ordinal as their avatar. The mandatory ownership check for `ord:` avatars (Section 3.2) mitigates this.

**HTTPS avatar phishing:** An `https:` avatar could be hosted at a domain that looks legitimate but serves malicious content. Implementations SHOULD warn users when an avatar is served from an HTTPS URL (as opposed to a verifiable Ordinals inscription or IPFS CID).

---

## Test Vectors

| Input | Expected behavior |
|-------|-------------------|
| `"avatar": "ord:abc...i0"` + inscription owner == name owner | Resolved, `avatar_verified: true` |
| `"avatar": "ord:abc...i0"` + inscription owner != name owner | Resolved, `avatar_verified: false` |
| `"avatar": "ipfs://QmXxx..."` resolving to `image/png` | Resolved successfully |
| `"avatar": "https://example.com/avatar.html"` (HTML page) | Rejected, fallback applied |
| `"avatar": "ord:abc...i0"` + inscription MIME is `text/html` | Rejected, fallback applied |
| `"avatar": "ord:abc...i0"` + inscription not found | Rejected, fallback applied |
| Bare inscription ID (legacy BtcName) | Treated as `ord:{id}`, resolved per `ord:` rules |

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
