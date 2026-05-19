# NRC-5: Session Keys and Scoped Permissions

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines temporary keys and limited permissions for NOVA smart accounts.

## Motivation

DApps and protocols frequently request broad, permanent access to user funds. This creates persistent attack surfaces. NOVA session keys provide scoped, temporary, revocable permissions.

## Specification

### Session Key

```typescript
struct SessionKey {
    publicKey: bytes;
    permissions: vec<Permission>;
    expiresAt: u64;
    spendingLimit: option<SpendingLimit>;
    allowedContracts: option<vec<address>>;
    nonce: bytes32;
    revoked: bool;
}
```

### Permission Scope

```typescript
struct Permission {
    contract: option<address>;
    selector: option<bytes4>;
    maxCalls: option<u64>;
    maxSpend: option<u256>;
}
```

### Spending Limit

```typescript
struct SpendingLimit {
    token: address;
    amount: u256;
    period: u64;       // time window in seconds
    spent: u256;       // amount used in current period
    periodStart: u64;  // start of current period
}
```

### Core Rule

> Every delegated permission needs a cap, scope, and expiry.

### Session Key Lifecycle

```text
1. Account owner creates session key with permissions
2. Session key is authorized on-chain
3. DApp / protocol uses session key within constraints
4. Session key expires or is revoked
5. No persistent access remains
```

### Revocation

Session keys can be revoked at any time by the account owner:

```typescript
fn revokeSessionKey(publicKey: bytes) {
    if msg.sender != self.owner { revert NotOwner; }
    self.sessionKeys[publicKey].revoked = true;
}
```

## Rationale

Session keys replace the pattern of approving unlimited token allowances to smart contracts. A DEX session key might allow swapping up to 5 NOVA for 1 hour, after which it is automatically invalid.

## Security Considerations

- Session key permissions must be enforced by the account validation logic
- Spending limits prevent session keys from draining accounts
- Expiry prevents stale permissions from persisting
