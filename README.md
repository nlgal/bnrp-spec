# BTC Name Resolution Protocol (BNRP)

> An open protocol for Bitcoin-native identity, name resolution, and cross-chain addressing using Ordinals inscriptions.

[![Status: Draft](https://img.shields.io/badge/status-draft-yellow)](./BNRPs/BNRP-IP-00.md)
[![License: CC0-1.0](https://img.shields.io/badge/license-CC0--1.0-lightgrey)](./LICENSE)
[![Discussions](https://img.shields.io/badge/discord-join-5865F2)](https://discord.gg/eNERqJU85x)

---

## What is BNRP?

BNRP is an open standard that makes `.btc` names function as a Bitcoin-native identity layer — equivalent to what ENS provides for Ethereum, but rooted entirely in Bitcoin and Ordinals.

A single `.btc` name can serve as:

- A **primary identity** displayed in wallets and explorers in place of a raw Bitcoin address
- A **reverse-resolved name** for any wallet that owns a `.btc` inscription
- A **cross-chain address pointer** (BTC, ETH, SOL, Base, Arbitrum, Polygon, and any SLIP-44 chain)
- A **Bitcoin-native avatar** via Ordinals inscription ID, IPFS, or HTTPS
- A **web/content domain** resolving to IPFS, IPNS, Arweave, or HTTP
- A **bitmap/metaverse pointer** linking to Bitcoin block districts and parcels
- A **profile identity** with display name, bio, and social handles

BNRP is fully backward-compatible with existing [BtcName](https://docs.btcname.id) inscriptions. All current `"p": "btcname"` records remain valid.

---

## Specification

| Document | Title | Status |
|----------|-------|--------|
| [BNRP-IP-00](./BNRPs/BNRP-IP-00.md) | Master Specification | Draft |
| [BNRP-IP-01](./BNRPs/BNRP-IP-01.md) | Name Normalization | Draft |
| [BNRP-IP-02](./BNRPs/BNRP-IP-02.md) | Routing Inscription Format | Draft |
| [BNRP-IP-03](./BNRPs/BNRP-IP-03.md) | Primary Name Mechanism | Draft |
| [BNRP-IP-04](./BNRPs/BNRP-IP-04.md) | Reverse Resolution & Anti-Spoofing | Draft |
| [BNRP-IP-05](./BNRPs/BNRP-IP-05.md) | Avatar URI Schemes | Draft |
| [BNRP-IP-06](./BNRPs/BNRP-IP-06.md) | REST API Specification v1 | Draft |

---

## Quick Example

A complete `alice.btc` routing inscription under BNRP:

```json
{
  "p": "bnrp",
  "op": "routing",
  "name": "alice.btc",
  "btc_taproot": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u",
  "btc_segwit": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
  "btc_lightning": "lnbc1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfsp5zyg3zyg3",
  "eth": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "base": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "arb": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "sol": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "avatar": "ord:9ba5066ee169289087167b963145901340e295a829fc9bda93ea4df45150a641i0",
  "content": "ipfs://QmNtHN7WE6gdkPfC4GKNkEBEbHEJjVMYLHTMkuqnGziqio",
  "bitmap_district": "840000"
}
```

Forward resolve `alice.btc` → taproot address:
```
GET https://resolver.bnrp.io/v1/address/alice.btc?chain=0
→ { "address": "bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u" }
```

Reverse resolve a wallet address → primary name:
```
GET https://resolver.bnrp.io/v1/reverse/bc1p4x9kv3qzx2mhjymfe7rl5xd6afgz8s3n9zkcwlvfszqd2r9j0qss4ef3u
→ { "name": "alice.btc", "verified": true }
```

---

## Design Principles

1. **Bitcoin is root of truth.** Name ownership is determined by UTXO ownership of inscription sats. No registry contract, no trusted database.
2. **Open and permissionless.** Anyone can run a BNRP indexer or resolver. The spec is governed as an open standard.
3. **No vendor lock-in.** No single company controls resolution. Multiple independent implementations are expected and encouraged.
4. **Backward compatible.** All existing `"p": "btcname"` inscriptions are valid BNRP records.
5. **Incrementally adoptable.** Wallets can implement primary name display (BNRP-IP-03) without supporting the full spec.

---

## For Wallets & Developers

### Minimum viable integration (1 day)

Implement primary name display using Solution B (already deployed on Bitcoin since block 840000):

```
GET https://resolver.bnrp.io/v1/reverse/{btc_address}
```

If it returns a name, display it. That's it.

### Full SDK (JavaScript/TypeScript)

```typescript
import { BNRPClient } from 'bnrp-sdk';

const bnrp = new BNRPClient({ resolver: 'https://resolver.bnrp.io' });

// Forward: name → address
const address = await bnrp.getAddress('alice.btc', 0); // coinType 0 = BTC

// Reverse: address → name
const name = await bnrp.reverseResolve('bc1p4x9kv...');

// Avatar
const avatarUrl = await bnrp.getAvatar('alice.btc');

// Full profile
const profile = await bnrp.getProfile('alice.btc');
```

Full SDK reference: [`/sdk-reference`](./sdk-reference/)

---

## Relationship to BtcName

BNRP is built on top of [BtcName (btcname.id)](https://docs.btcname.id), not in competition with it. BtcName established the foundational layer — name ownership via Ordinals, routing inscriptions, and the draft primary name proposals. BNRP formalizes and extends that foundation into a complete open standard.

| Layer | BtcName | BNRP |
|-------|---------|------|
| Name ownership | ✅ | ✅ (same) |
| Forward resolution | ✅ | ✅ (extended) |
| Reverse resolution | ✅ (draft) | ✅ (standardized + anti-spoofing) |
| Primary name | ✅ (draft, 2 proposals) | ✅ (Solution B canonical, Solution A extended) |
| Avatar | Inscription ID only | ✅ ord:, ipfs:, ipns:, https:, data: |
| Cross-chain (EVM L2) | ❌ | ✅ SLIP-44 coinType schema |
| Profile records | ❌ | ✅ ENSIP-5/18 equivalent |
| Web/content resolution | ❌ | ✅ IPFS, IPNS, Arweave, HTTP |
| Bitmap/metaverse | ❌ | ✅ |
| Open indexer spec | ❌ | ✅ |
| Open REST API spec | ❌ | ✅ |
| SDK | ❌ | ✅ (reference) |

---

## Contributing

BNRP is an open standard. Contributions, corrections, and new proposals are welcome.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full process.

**Quick version:**
1. Fork this repo
2. Write your proposal using [BNRP-IP-TEMPLATE.md](./BNRP-IP-TEMPLATE.md)
3. Save it as `BNRPs/BNRP-IP-XX.md` (next available number)
4. Open a pull request
5. Community review in Discord before merge

---

## Community

- **Discord:** [BtcName Discord](https://discord.gg/eNERqJU85x) — `#bnrp` channel (request creation) or `#primary-name`
- **GitHub Discussions:** Use the Discussions tab on this repo
- **Twitter/X:** Tag `#BNRP` and `#BtcName`

---

## License

[CC0 1.0 Universal](./LICENSE) — public domain. No rights reserved. Build freely.
