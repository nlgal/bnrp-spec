# BNRP SDK Reference

This directory contains the reference SDK specification and example implementation for the BTC Name Resolution Protocol.

## Status

Reference implementation: **planned**. The TypeScript interfaces are defined in [BNRP-IP-06](../BNRPs/BNRP-IP-06.md).

## Quickstart (once published to npm)

```bash
npm install bnrp-sdk
```

```typescript
import { createBNRPClient } from 'bnrp-sdk';

const bnrp = createBNRPClient({
  resolver: 'https://resolver.bnrp.io',
});

// Forward: name → BTC address
const address = await bnrp.getAddress('alice.btc');

// Reverse: BTC address → name
const result = await bnrp.reverseResolve('bc1p4x9kv...');
if (result.name && result.verified) {
  console.log(`Primary name: ${result.name}`);
}

// Avatar
const avatar = await bnrp.getAvatar('alice.btc');
console.log(avatar.avatar_url); // Ready for <img src>
```

## Interfaces

All TypeScript interfaces are defined in [BNRP-IP-06 Section: SDK Reference](../BNRPs/BNRP-IP-06.md).

## Contributing an Implementation

To contribute a reference implementation:
1. Implement the `BNRPClient` interface from BNRP-IP-06
2. Add a `package.json` with `"name": "bnrp-sdk"`
3. Include tests covering all BNRP-IP-01 through BNRP-IP-06 test vectors
4. Open a PR
