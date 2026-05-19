# NRC-6: Native Naming Standard

**Status:** Draft  
**Category:** Wallet UX  

## Abstract

Defines NOVA's human-readable name system for accounts.

## Motivation

Raw Bech32m addresses are not human-readable. Users need recognizable names for sending, receiving, and verifying identity on-chain.

## Specification

### Name Format

```text
<name>.nova

Examples:
alex.nova
cryptolar.nova
opensense.nova
vault.cryptolar.nova
dao.opensense.nova
```

### Name Rules

- Top-level names: `[a-z0-9-]{3,64}.nova`
- Subdomain names: `[a-z0-9-]{1,64}.<parent>.nova`
- Names are case-insensitive (stored lowercase)
- Names cannot start or end with a hyphen
- No consecutive hyphens
- Reserved names: `nova`, `www`, `api`, `rpc`, `node`, `admin`, `system`, `nova-staking`, `bridge`

### Name Record

```typescript
struct NameRecord {
    name: string;
    owner: address;
    resolver: address;
    ttl: u64;
    registeredAt: u64;
    expiresAt: u64;
}
```

### Resolver Interface

```typescript
interface NameResolver {
    fn resolve(name: string) -> option<address>;
    fn reverseResolve(address: address) -> option<string>;
    fn setText(name: string, key: string, value: string);
    fn getText(name: string, key: string) -> option<string>;
}
```

### Deterministic Alias

Every account also has a deterministic human-readable alias derived from its address:

```text
nova1z4m8...c3s5a
= funny-horse-idaho-king
```

This alias is not a global username. It is a human-readable fingerprint that always maps to one address.

### Wallet Display Priority

```text
1. Global name (alex.nova)       — if registered
2. Deterministic alias            — always available
3. Abbreviated address            — always available
```

## Rationale

Layered naming (global names, deterministic aliases, raw addresses) gives wallets flexibility to display the most recognizable identifier.

## Security Considerations

- Name registration requires NOVA (anti-spam)
- Names expire and can be re-registered after expiry
- Name transfers are on-chain transactions
- Deterministic aliases cannot be spoofed (cryptographically derived)
