# NRC-16: Canonical Intent Hash

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines a canonical intent object for every high-risk transaction. The intent describes what the transaction means in human-readable terms, is hashed with domain-separated BLAKE3-256, and signed alongside the transaction. If transaction bytes and human-readable intent diverge, the account rejects the signature.

## Motivation

Raw transaction bytes are opaque to users and wallets. A user who signs a blob cannot verify what it actually does. This enables:

- Blind-signing attacks where malware substitutes transaction content
- Phishing wallets that display one action and execute another
- Multisig participants who approve transactions they do not understand

A canonical intent hashes the meaning of the transaction so that signatures cover both the execution bytes and the semantic intent.

## Specification

### Intent Type

```typescript
enum IntentType {
    Transfer,
    ContractCall,
    ContractDeploy,
    AccountUpdate,
    Upgrade,
    Governance,
    Bridge,
    Batch,
}
```

### Canonical Intent

```typescript
struct CanonicalIntent {
    intentType: IntentType;
    from: address;
    to: address;
    asset: address;
    amount: u256;
    codeChanges: bool;
    newPermissions: bool;
    upgradeEffects: bool;
    oracleChanges: bool;
    bridgeEffects: bool;
    feeLimit: u256;
    validUntil: u64;
    nonce: u64;
}
```

### Intent Hash Computation

```text
intent_hash = BLAKE3-256("NOVA_INTENT_V1" || serialize(canonical_intent))
```

Serialization is canonical field-order deterministic encoding. All fields are encoded in the order defined in the struct. Variable-length fields are length-prefixed. Booleans encode as single bytes.

### Intent Signing

```typescript
struct IntentSignature {
    intentHash: bytes32;
    transactionHash: bytes32;
    signature: bytes;
}
```

The signer signs the concatenation:

```text
signed_payload = intent_hash || transaction_hash
```

Both hashes must be covered by a single signature.

### Validation Rules

1. The account must recompute `intent_hash` from the submitted `CanonicalIntent`
2. The recomputed hash must match `intentHash` in the signature
3. The `transactionHash` must match the actual transaction hash
4. The signature must be valid over `intent_hash || transaction_hash`
5. If any check fails, the account must reject with `IntentMismatch`
6. `validUntil` must not be in the past
7. `nonce` must match the account's expected nonce

### High-Risk Transaction Definition

The following transaction types require canonical intent:

- Any transfer above the account's declared threshold
- Contract deployment
- Account update (key rotation, permission change)
- Upgrade transactions
- Governance votes on protocol changes
- Bridge operations
- Batch transactions containing any of the above

Simple transfers below the account threshold are exempt.

## Rationale

Domain separation ("NOVA_INTENT_V1") prevents cross-protocol replay. Requiring the signature to cover both intent and transaction hashes makes blind-signing attacks infeasible because the attacker would need to produce a valid intent that matches both the malicious transaction and the victim's expectations.

## Security Considerations

- Wallets must display the full canonical intent before signing
- Intent hashes must be compared by the account contract, not trusted off-chain
- The `validUntil` field prevents intent replay across time
- The `nonce` field prevents intent replay within the same account
- If a wallet skips intent verification, the on-chain check still protects the user
