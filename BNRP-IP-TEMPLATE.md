# BNRP-IP-XX: Title

| Field | Value |
|-------|-------|
| **BNRP-IP** | XX |
| **Title** | Short descriptive title |
| **Author(s)** | Your Name (GitHub handle or contact) |
| **Status** | Draft |
| **Created** | YYYY-MM-DD |
| **Requires** | BNRP-IP-YY (if any) |
| **Replaces** | BNRP-IP-ZZ (if any) |

---

## Abstract

One paragraph. What does this proposal do and why does it exist?

---

## Motivation

Why is this change needed? What problem does it solve? What is currently broken or missing? Reference any prior discussions, GitHub issues, or Discord threads.

---

## Specification

The technical specification. This section is normative. Use RFC 2119 key words:
- **MUST** / **MUST NOT** — absolute requirements
- **SHOULD** / **SHOULD NOT** — strong recommendations with allowed exceptions
- **MAY** — optional

### Sub-section 1

### Sub-section 2

---

## JSON Schema

If this proposal defines a record format, include the full JSON Schema Draft-07 definition here.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "RecordName",
  "type": "object",
  "required": [],
  "properties": {}
}
```

---

## Examples

At least one complete, filled-in JSON example. Not a template — actual data.

### Example 1: [Description]

```json
{
  "p": "bnrp",
  "op": "...",
  "name": "example.btc"
}
```

---

## Backward Compatibility

How does this proposal affect existing `"p": "btcname"` inscriptions? Are any existing BNRP-IPs affected? What do existing indexers need to change, if anything?

---

## Security Considerations

What are the security implications of this proposal? Address at minimum:
- Can this be used to spoof ownership?
- Can this be used to point users to malicious content?
- Is there an indexer disagreement risk?
- Are there replay or revocation issues?

---

## Test Vectors

Provide at least three test cases: one valid, one invalid (should be rejected), and one edge case.

| Input | Expected Output | Notes |
|-------|----------------|-------|
| | | |

---

## Reference Implementation

Link to or include a reference implementation (code) if available.

---

## Copyright

This BNRP-IP is released under [CC0 1.0 Universal](../LICENSE). No rights reserved.
