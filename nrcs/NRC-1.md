# NRC-1: Smart Account Interface

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines the standard interface for NOVA smart accounts. There is no permanent EOA/contract split. Every account is a smart account from genesis.

## Motivation

Ethereum's EOA model separates externally-owned accounts from contract accounts. This creates fundamental UX problems:

- EOAs cannot enforce spending policies
- EOAs cannot rotate keys
- EOAs cannot recover from lost keys
- EOAs cannot sponsor gas for others
- EOAs cannot require multisig for high-value transfers

NOVA eliminates this split. Every account is a smart account with configurable validation logic.

## Specification

### Account Interface

```typescript
interface SmartAccount {
    fn validateTransaction(tx: Transaction, context: ExecutionContext) -> bool;
    fn validateSignature(tx: Transaction, signature: Signature) -> bool;
    fn getNonce() -> u64;
    fn incrementNonce();
}
```

### Supported Features

- Transaction validation
- Signature verification (multiple schemes via NRC-2)
- Nonce handling
- Key rotation
- Session keys (via NRC-5)
- Guardian recovery (via NRC-4)
- Spending policies
- Gas sponsorship

### Account Deployment

Accounts are deployed with:

```typescript
struct AccountInit {
    code: bytes;
    salt: bytes32;
    initialAuthRoot: bytes32;
}
```

Address derivation:

```text
address = BLAKE3-256("NOVA_ACCOUNT_V1" || code_hash || salt || initial_authRoot)
```

### Core Principle

> The account is the on-chain identity. Keys and mnemonics are authorization methods.

## Rationale

Making every account a smart account from genesis avoids the complexity of ERC-4337-style overlay protocols. Account abstraction is not an add-on. It is the foundation.

## Backward Compatibility

N/A. This is a genesis design.

## Security Considerations

- Account code must be validated before deployment
- Auth root must cover all initial authorization methods
- Account code must handle fallback/receive for native transfers
