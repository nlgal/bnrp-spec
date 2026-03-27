# Contributing to BNRP

BNRP is an open standard. Anyone can propose changes, new features, or corrections.

---

## BNRP Improvement Proposal (BNRP-IP) Process

All changes to the BNRP specification are made via BNRP Improvement Proposals (BNRP-IPs).

### Proposal Status Levels

| Status | Meaning |
|--------|---------|
| `Idea` | Informal discussion in Discord or GitHub Discussions before a PR |
| `Draft` | PR open, spec written, community review in progress |
| `Review` | Core maintainers have reviewed; final comment period (14 days) |
| `Final` | Accepted into the canonical spec |
| `Superseded` | Replaced by a newer BNRP-IP |
| `Withdrawn` | Author withdrew the proposal |
| `Obsolete` | No longer applicable |

### How to Submit a BNRP-IP

1. **Discuss first.** Open a GitHub Discussion or post in the BtcName Discord `#bnrp` channel. Get early feedback before writing the full spec. This saves everyone time.

2. **Fork the repo.**
   ```bash
   git clone https://github.com/bnrp/bnrp-spec
   git checkout -b BNRP-IP-XX-short-title
   ```

3. **Write your proposal.** Copy [BNRP-IP-TEMPLATE.md](./BNRP-IP-TEMPLATE.md) to `BNRPs/BNRP-IP-XX.md` (use the next available number). Fill in all required sections.

4. **Add examples and test vectors.** Every BNRP-IP that defines a data format MUST include at least one complete JSON example. Proposals that define resolution behavior MUST include pseudocode or test vectors.

5. **Open a pull request** against `main`. Use the title format: `BNRP-IP-XX: Short Title`.

6. **Community review.** Respond to comments. The proposal will be discussed in the Discord `#bnrp` channel. Substantive objections must be resolved before the proposal advances.

7. **Final status.** After the 14-day final comment period with no unresolved objections, a maintainer will merge the PR and mark the BNRP-IP as `Final`.

---

## What Makes a Good BNRP-IP

- **Concrete, not vague.** Every normative statement uses MUST, SHOULD, MAY, MUST NOT, or SHOULD NOT (per RFC 2119).
- **Includes complete examples.** Schemas must have filled-in JSON examples, not just empty templates.
- **Addresses backward compatibility.** Explain how existing BtcName inscriptions are affected (they should not be broken).
- **Considers the security model.** Every proposal that touches name resolution, ownership, or records must include a threat analysis section.
- **Scoped appropriately.** One concern per BNRP-IP. If your proposal covers two unrelated topics, split it.

---

## Non-Proposal Contributions

Bug fixes, typos, and clarifications to existing BNRP-IPs can be submitted as a standard PR without the full BNRP-IP process. Use the PR title format: `Fix: short description`.

---

## Code of Conduct

This is a technical standards community. Contributions are evaluated on their technical merit. Be direct, be specific, be respectful.

---

## Maintainers

BNRP is currently maintained by the open protocol working group. There is no single controlling entity. Maintainer status is earned through sustained, high-quality contributions to the spec.
