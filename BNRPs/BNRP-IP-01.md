# BNRP-IP-01: Name Normalization

| Field | Value |
|-------|-------|
| **BNRP-IP** | 01 |
| **Title** | Name Normalization |
| **Author(s)** | Open Protocol Working Group |
| **Status** | Draft |
| **Created** | 2026-03-26 |
| **Requires** | — |
| **Replaces** | — |

---

## Abstract

This proposal defines the canonical normalization rules that MUST be applied to every `.btc` name before any resolution, indexing, or validation step. Normalization ensures that `Alice.BTC`, `alice.btc`, and `ALICE.BTC` all resolve identically, and that homoglyph attacks are detectable.

---

## Motivation

Without a normalization standard, different indexers and resolvers may treat the same name differently. A name entered as `Alice.BTC` in one wallet and `alice.btc` in another could resolve to different owners. This is a critical correctness and security requirement that must be specified before any other resolution logic.

---

## Specification

### 1. Input Processing Order

Every BNRP implementation MUST apply normalization steps in the following order:

1. Unicode NFC normalization
2. Lowercase conversion (Unicode case folding)
3. Label splitting on `.`
4. Per-label validation
5. Reconstruction

### 2. Unicode Normalization

The input string MUST be normalized to Unicode NFC (Canonical Decomposition followed by Canonical Composition) before any other processing.

Implementations MUST use a standard Unicode library for this step. Manual string manipulation is not acceptable.

### 3. Case Folding

After NFC normalization, the entire string MUST be converted to lowercase using Unicode full case folding (as defined in Unicode Standard Annex #21). ASCII characters follow standard ASCII lowercasing. Non-ASCII characters follow Unicode case folding tables.

### 4. Label Splitting

The normalized, lowercased string MUST be split on the `.` character (U+002E) to produce labels.

A valid `.btc` name consists of exactly two labels: `[name, "btc"]`. The TLD label MUST be `btc`. Multi-level names (e.g., `sub.alice.btc`) are not supported in this version of the spec.

### 5. Per-Label Validation

Each label (excluding the TLD `btc`) MUST satisfy all of the following:

#### 5.1 Allowed Characters

Labels MAY contain:
- ASCII lowercase letters: `a–z`
- ASCII digits: `0–9`
- Hyphens: `-` (U+002D)
- Unicode letters and digits in NFC form (for internationalized names)

Labels MUST NOT:
- Begin or end with a hyphen
- Contain two consecutive hyphens in positions 3 and 4 (i.e., `xn--` prefix is reserved for Punycode IDN; other double-hyphen uses are prohibited)
- Contain whitespace, null bytes, or control characters
- Be empty

#### 5.2 Length Constraints

- Minimum label length: 1 character
- Maximum label length: 63 characters (per DNS label convention)
- Maximum total name length (including `.btc`): 128 characters (per BtcName Primary Name max-length rule from Solution B)

Names exceeding 128 characters MUST NOT be used as primary names. They MAY be registered as name inscriptions but will not qualify for primary name resolution.

#### 5.3 Reserved Labels

The following labels are reserved and MUST NOT be registered or resolved as user names:

| Label | Reason |
|-------|--------|
| `btc` | TLD |
| `bitmap` | Reserved for bitmap/metaverse integration |
| `ord` | Reserved for Ordinals protocol use |
| `sat` | Reserved for satoshi-level addressing |
| `bnrp` | Reserved for protocol use |
| `resolver` | Reserved for resolver infrastructure |
| `_bnrp` | Reserved for DNS bridge TXT records |

### 6. Internationalized Names (IDN)

Names containing non-ASCII Unicode characters are valid under BNRP, subject to the following:

- The label MUST be valid under UTS#46 (Unicode IDNA Compatibility Processing) with `UseSTD3ASCIIRules=true` and `CheckHyphens=true`.
- Wallets and resolvers MUST display the Unicode form to users but MUST ALSO show the Punycode equivalent (prefixed `xn--`) in a tooltip or secondary display to guard against homoglyph phishing.
- Implementations MUST flag names containing characters from mixed scripts (e.g., Latin + Cyrillic in the same label) as "potentially deceptive" in their UI. This is a SHOULD for resolvers and a MUST for wallet display contexts.

### 7. Normalization Output

The canonical form of a name is the result of applying all steps above. This canonical form MUST be used:
- As the key for all indexer lookups
- As the value stored in the `name` field of inscriptions
- In all API responses
- In all comparisons between names

### 8. Normalization Function (Pseudocode)

```pseudocode
function normalizeName(input: string) -> string | error:
  // Step 1: NFC
  s = unicode_nfc(input)

  // Step 2: Case fold
  s = unicode_casefold(s)

  // Step 3: Split
  labels = s.split(".")
  if len(labels) != 2:
    return error("INVALID_LABEL_COUNT")
  if labels[1] != "btc":
    return error("INVALID_TLD")

  name_label = labels[0]

  // Step 4: Length
  if len(name_label) < 1:
    return error("LABEL_TOO_SHORT")
  if len(name_label) > 63:
    return error("LABEL_TOO_LONG")
  if len(s) > 128:
    return error("NAME_TOO_LONG")

  // Step 5: Reserved
  if name_label in RESERVED_LABELS:
    return error("RESERVED_LABEL")

  // Step 6: Character validation
  if name_label.startswith("-") or name_label.endswith("-"):
    return error("INVALID_HYPHEN_POSITION")
  if name_label[2:4] == "--" and name_label[0:2] != "xn":
    return error("INVALID_DOUBLE_HYPHEN")
  for char in name_label:
    if not isAllowedCharacter(char):
      return error("INVALID_CHARACTER: " + char)

  return s  // e.g. "alice.btc"
```

---

## JSON Schema Impact

The `name` field in all BNRP inscription types MUST contain the normalized form of the name. Indexers MUST normalize the `name` field value before storing or comparing it, even if the inscription contains a non-normalized form.

---

## Examples

### Valid Names

| Input | Normalized Output | Notes |
|-------|-----------------|-------|
| `Alice.BTC` | `alice.btc` | Case fold applied |
| `alice.btc` | `alice.btc` | Already normalized |
| `ALICE.BTC` | `alice.btc` | Case fold applied |
| `test-name.btc` | `test-name.btc` | Hyphen in middle is valid |
| `840000.btc` | `840000.btc` | All-numeric valid |
| `café.btc` | `café.btc` | Unicode NFC, valid IDN |

### Invalid Names

| Input | Error | Reason |
|-------|-------|--------|
| `-alice.btc` | `INVALID_HYPHEN_POSITION` | Starts with hyphen |
| `alice-.btc` | `INVALID_HYPHEN_POSITION` | Ends with hyphen |
| `al--ice.btc` | `INVALID_DOUBLE_HYPHEN` | Double hyphen not in xn-- position |
| `alice.eth` | `INVALID_TLD` | Wrong TLD |
| `alice` | `INVALID_LABEL_COUNT` | No TLD |
| `btc.btc` | `RESERVED_LABEL` | Reserved label |
| ` alice.btc` | `INVALID_CHARACTER` | Leading space |
| (128+ char name) | `NAME_TOO_LONG` | Exceeds max length |

---

## Backward Compatibility

All existing BtcName inscriptions that used lowercase ASCII names (the vast majority) are unaffected. Existing inscriptions with mixed-case names will normalize to their lowercase equivalent. Indexers MUST normalize stored names when re-indexing.

---

## Security Considerations

**Homoglyph attacks:** The Unicode case-folding and IDN rules reduce but do not eliminate homoglyph risk. Implementations MUST display Punycode alongside Unicode for non-ASCII names. See BNRP-IP-00 Section 15 for the full threat model.

**Normalization inconsistency:** If different implementations use different Unicode library versions, rare edge cases may normalize differently. Implementations SHOULD document their Unicode library version.

---

## Test Vectors

| Input | Expected Output | Should Pass? |
|-------|----------------|-------------|
| `alice.btc` | `alice.btc` | Yes |
| `Alice.BTC` | `alice.btc` | Yes |
| `840000.btc` | `840000.btc` | Yes |
| `-bad.btc` | error | No |
| `bad-.btc` | error | No |
| `alice.eth` | error | No |
| `btc.btc` | error | No |
| `xn--n3h.btc` | `xn--n3h.btc` | Yes (Punycode IDN) |

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE).
