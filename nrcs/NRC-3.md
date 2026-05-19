# NRC-3: Wallet Derivation and Recovery Standard

**Status:** Draft  
**Category:** Wallet UX  

## Abstract

Defines how NOVA wallets derive keys and accounts from seed phrases and other recovery mechanisms.

## Motivation

Users need a standard way to:

- Generate NOVA accounts from mnemonic phrases
- Derive classical and post-quantum key pairs
- Recover accounts when keys are lost
- Distinguish between recovery credentials and the account itself

## Specification

### Seed Derivation

NOVA is compatible with BIP-39 mnemonic generation:

```text
entropy -> BIP-39 mnemonic -> BIP-39 seed
```

### Key Derivation Path

```text
m / purpose' / coin_type' / account' / change / address_index

purpose:    44' (BIP-44) or 101010' (NOVA-specific, TBD)
coin_type:  TBD (SLIP-44 registration required)
```

### Key Branches

```typescript
struct DerivedKeys {
    classical: ClassicalKeys;
    postQuantum: option<PostQuantumKeys>;
    hybrid: option<HybridKeys>;
}

struct ClassicalKeys {
    schemeId: u16;       // e.g. 0x01 for secp256k1
    publicKey: bytes;
    privateKey: bytes;
}
```

### Recovery Flow

```text
1. User enters mnemonic
2. Wallet derives seed
3. Wallet generates key pairs for supported schemes
4. Wallet computes or recovers account address
5. Account = smart account address, NOT public key
```

### Critical Distinction

```text
Mnemonic = recovery / auth credential
Account  = smart account address

One mnemonic can authorize multiple accounts.
One account can accept multiple authorization methods.
```

## Rationale

Separating recovery credentials from account identity enables key rotation, social recovery, and multi-device support without changing the account address.

## Security Considerations

- Post-quantum key material is larger; wallets must handle storage
- Recovery phrases must never be transmitted or stored unencrypted
- Wallets should warn users about quantum-vulnerable schemes
