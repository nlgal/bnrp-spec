# Proposal: BNRP × BtcName Collaboration / Merger

**Date:** April 17, 2026  
**Author:** [@nlgal](https://github.com/nlgal)  
**Status:** Open for Discussion  
**Cross-posted:** [BtcName/BtcName](https://github.com/BtcName/BtcName) — see linked issue/discussion

---

## Summary

BNRP and BtcName are solving adjacent problems in the same space. BtcName established the foundational layer — name ownership via Ordinals, routing inscriptions, and the draft primary name proposals. BNRP formalizes and extends that foundation into a complete open standard: avatar verification, multi-TLD resolution, open indexer and REST API specs, cross-chain address routing, and a live reference implementation.

This proposal is a formal invitation to the BtcName team to align these two efforts — whether as a full merger, a co-authorship arrangement, or a ratified compatibility agreement — so that wallets, explorers, and builders have one clear standard to implement.

---

## Why This Makes Sense

### We Are Not Competitors

BtcName is a registrar and rules layer. BNRP is a resolution and identity protocol. These are different layers of the same stack — the same relationship as a DNS registrar vs. the DNS resolution standard.

Specifically:

| Layer | BtcName | BNRP |
|-------|---------|------|
| Name issuance + ownership | ✅ | Defers to BtcName |
| Forward resolution | ✅ | ✅ (extended, open spec) |
| Reverse resolution | Draft | ✅ (standardized + anti-spoofing) |
| Primary name | Draft (2 proposals) | ✅ (Solution B canonical) |
| Avatar (multi-scheme) | Inscription ID only | ✅ ord:, ipfs:, ipns:, https:, data: |
| Cross-chain (SLIP-44) | ❌ | ✅ |
| Profile records | ❌ | ✅ (ENSIP-5/18 equivalent) |
| Web/content resolution | ❌ | ✅ IPFS, IPNS, Arweave, HTTP |
| Bitmap/metaverse | ❌ | ✅ |
| Open indexer spec | ❌ | ✅ |
| Open REST API spec | ❌ | ✅ |
| Multi-TLD (.sats, .x, .unisat, .xbt, .sat) | ❌ | ✅ |

### BNRP Is Fully Backward-Compatible

Every existing `"p": "btcname"` inscription is valid under BNRP. No re-inscription required. No migration cost. BtcName's existing user base inherits full BNRP functionality automatically.

### One Standard Is Stronger Than Two

The Bitcoin name space is fragmented. Wallets are reluctant to implement resolution because there is no canonical spec to implement. Every parallel standard makes that problem worse. A joint standard — backed by both teams — removes the "which one do we build for?" question entirely.

---

## What We Are Proposing

Three options, in order of preference:

### Option A — Full Merger (Preferred)

BtcName and BNRP merge into a single specification under a jointly governed open standard. The BNRP-IP series becomes the canonical technical spec. BtcName retains its identity as the name registry layer. Both teams co-author the standard and jointly publish updates.

Governance model: open, CC0, PR-based — same as BNRP today.

### Option B — Co-Authorship / Formal Endorsement

BtcName formally endorses BNRP as the resolution standard for `.btc` names. BtcName team members are invited as co-authors on the BNRP-IP documents. BNRP links back to BtcName as the canonical name registry. Both repos cross-reference.

### Option C — Compatibility Agreement

Both projects remain independent but publish a formal compatibility document confirming that:
- All `"p": "btcname"` records are valid BNRP records
- Both specs commit to non-breaking changes only on the shared record schema
- A joint test suite is maintained for resolver implementations

---

## What BNRP Has Today

- **Live resolver and demo:** [bnrp.name](https://bnrp.name) — resolves .btc, .sats, .unisat, .x, .xbt, .sat
- **Full spec:** 7 BNRP-IP documents (normalization, routing format, primary name, reverse resolution, avatar URI, REST API)
- **Genesis inscription:** `d2650a7c0caed740c29e033ad404ec41432042650696855627acd10031dd11d5i0` — April 15, 2026
- **CC0 license:** Public domain, no governance token, no protocol fee
- **Reference implementation:** JavaScript/TypeScript SDK scaffold
- **Substack publication:** [ordinalpunk72.substack.com](https://substack.com/pub/ordinalpunk72)

---

## Next Steps

If the BtcName team is open to this conversation:

1. Open a corresponding Discussion in the BtcName repo
2. Schedule a working session between both teams
3. Publish a joint statement signaling alignment to the community
4. Produce a merged BNRP-IP-00 that credits both teams

---

## Contact

- GitHub: [@nlgal](https://github.com/nlgal)
- Twitter/X: see profile
- GitHub Discussions: [nlgal/bnrp-spec/discussions](https://github.com/nlgal/bnrp-spec/discussions)
