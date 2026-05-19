# NRC-2: Signature Scheme Registry

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines the registry of supported signature schemes for NOVA accounts. Makes NOVA crypto-agile from genesis.

## Motivation

Blockchains that hardcode a single signature scheme cannot adapt to cryptographic breakthroughs. Quantum computing threatens classical schemes. New post-quantum schemes are larger and more expensive.

NOVA needs a registry that supports classical, post-quantum, and hybrid schemes simultaneously.

## Specification

### Scheme IDs

```text
0x01  secp256k1           (ECDSA, 65-byte signatures)
0x02  Ed25519             (EdDSA, 64-byte signatures)
0x03  P-256               (ECDSA, 64-byte signatures)
0x10  ML-DSA-44           (post-quantum, dilithium)
0x11  ML-DSA-65           (post-quantum, dilithium)
0x12  ML-DSA-87           (post-quantum, dilithium)
0x20  SLH-DSA-SHA2-128s   (post-quantum, sphincs)
0x21  SLH-DSA-SHA2-128f   (post-quantum, sphincs)
0x30  Hybrid-secp256k1-ML-DSA-65
0x31  Hybrid-P256-ML-DSA-65
```

### Registry Entry

```typescript
struct SignatureScheme {
    id: u16;
    name: string;
    publicKeySize: u32;
    signatureSize: u32;
    verifyGas: GasCost;
    active: bool;
}
```

### Migration Path

```text
classical -> hybrid -> PQ recommended -> PQ required for high-value accounts
```

### Rules

- New schemes are added through NRC-9 runtime upgrades
- Schemes are never removed, only deprecated
- Deprecated schemes may be disallowed for new accounts
- Existing accounts may continue using deprecated schemes until a hard fork

## Rationale

The range-based ID system (0x01-0x0F classical, 0x10-0x1F ML-DSA, 0x20-0x2F SLH-DSA, 0x30-0x3F hybrid) makes the registry self-organizing.

## Security Considerations

- Hybrid schemes provide defense in depth during transition
- Public keys remain hidden until first on-chain use
- Signature verification gas costs must reflect actual compute
