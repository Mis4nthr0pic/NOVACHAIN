# NRC-14: Address and Human Alias Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines raw addresses, Bech32m encoding, and deterministic human-readable aliases for NOVA accounts.

## Motivation

Users need to identify, verify, and share addresses. Raw 32-byte values are not practical for human communication.

## Specification

### Internal Address

```text
32 bytes
```

### Derivation

```text
address = BLAKE3-256(
    "NOVA_ACCOUNT_V1" ||
    account_code_hash ||
    salt ||
    initial_auth_root
)
```

### Bech32m Encoding

Canonical user-facing encoding uses Bech32m:

```text
nova1z4m8r7qk0n6v2x9h3c5twaepldjsyfgb1u84rmq2k0v6n9c3s5a
```

### Network Prefixes

```text
nova1...   mainnet
tnova1...  testnet
dnova1...  devnet
```

### Deterministic Human-Readable Alias

Every address maps to a deterministic word-sequence alias:

```text
nova1z4m8...c3s5a
= funny-horse-idaho-king
```

Properties:

- Deterministic: same address always produces the same alias
- Not a global username
- Cannot be registered or transferred
- Cryptographically derived from the address
- Used as a human-readable fingerprint

### Alias Generation

```text
alias = wordlist_encode(BLAKE3-256("NOVA_ALIAS_V1" || address)[0..8])
```

Four words from a fixed BIP-39-style wordlist, encoding the first 8 bytes of a domain-separated hash.

### Display Priority

Wallets should display:

```text
1. Global name (alex.nova)          — if registered (NRC-6)
2. Deterministic alias               — always available
3. Abbreviated Bech32m address       — always available
```

Full display for a registered account:

```text
alex.nova
funny-horse-idaho-king
nova1z4m8...c3s5a
```

Full display for an unregistered account:

```text
funny-horse-idaho-king
nova1z4m8...c3s5a
```

### Transaction Hash Format

Transaction hashes use a separate Bech32m prefix:

```text
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9
```

Internal:

```text
tx_hash = BLAKE3-256("NOVA_TX_V1" || canonical_transaction_bytes)
```

Developer hex format:

```text
0x8f42c1e0a99b7d4d6c2e53f83a5b88dd934fb1e2b92a7126cc6f2dd5a7c9310e
```

## Rationale

Bech32m is error-detecting, case-insensitive, and widely supported. Deterministic aliases provide an always-available human-readable fallback without requiring name registration.

## Security Considerations

- Bech32m provides built-in error detection for typos
- Aliases are fingerprints, not authentication mechanisms
- Never use aliases as sole verification for large transfers; always check the full address
