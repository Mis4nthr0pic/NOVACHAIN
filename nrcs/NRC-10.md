# NRC-10: Token Capability Standard

**Status:** Draft  
**Category:** Account Security  

## Abstract

Replaces ERC20-style infinite approvals with bounded, scoped, expiring, revocable token permissions.

## Motivation

ERC20's `approve(spender, MAX_UINT256)` pattern gives permanent, unlimited spending power to smart contracts. When any intermediate contract is compromised, all approved tokens are at risk.

NOVA eliminates this pattern by design.

## Specification

### Core Rule

> No infinite approvals. No permanent router power. Every spend permission has a cap, purpose, recipient, and expiry.

### Spend Authorization (One-Time)

```typescript
struct SpendAuth {
    owner: address;
    spender: address;
    token: address;
    amount: u256;
    recipient: address;
    purpose: bytes32;
    validAfter: u64;
    validUntil: u64;
    nonce: bytes32;
}
```

Usage:

```typescript
token.transferWithAuth(auth, signature);
```

After use:

```text
nonce consumed
authorization dead
cannot be reused
cannot be increased
cannot become infinite
```

### TokenSpend Capability (Recurring, Bounded)

```typescript
capability TokenSpend {
    token: address;
    spender: address;
    maxAmount: u256;
    remainingAmount: u256;
    expiresAt: u64;
    allowedRecipient: option<address>;
    allowedFunction: option<selector>;
    revocable: bool;
}
```

### Comparison: Ethereum vs NOVA

```text
Ethereum:  approve(router, infinite)
           swap later
           hope nothing is compromised

NOVA:      authorize exact amount, exact recipient, exact purpose, exact window
           use once
           authorization is dead
```

### Capability Lifecycle

```text
1. Owner grants TokenSpend capability
2. Spender uses capability within constraints
3. Capability tracks remaining amount
4. Capability expires or is revoked
5. No residual permission exists
```

### Revocation

Capabilities marked `revocable: true` can be cancelled by the owner at any time:

```typescript
fn revokeCapability(capabilityId: bytes32) {
    if msg.sender != self.owner { revert NotOwner; }
    self.capabilities[capabilityId].revoked = true;
}
```

### Default Behavior

- Approvals are one-time by default
- Amount is bounded by the exact transaction amount
- Expiry is short (minutes to hours)
- Recipient is specified
- Purpose can be tagged

## Rationale

Every major DeFi hack involving token approvals exploited the infinite approval pattern. Removing it structurally eliminates the attack class.

## Security Considerations

- One-time approvals prevent replay attacks
- Expiry prevents stale permissions from persisting
- Bounded amounts prevent draining more than intended
- Purpose tagging enables wallet risk analysis
