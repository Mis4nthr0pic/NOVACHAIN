# NRC-17: Multisig Intent Agreement

**Status:** Draft  
**Category:** Account Security  

## Abstract

Requires every signer in a multisig to sign the same canonical intent hash. If signer intent hashes diverge, the chain rejects the transaction.

## Motivation

In existing multisig implementations, participants sign raw transaction hashes without a shared understanding of what the transaction does. This creates risks:

- One signer approves a transfer while another approves a contract call with the same hash
- An attacker crafts transactions where different signers see different meanings
- Quorum is reached on bytes, not on intent

NOVA requires that all multisig signers agree on the canonical intent before a transaction is valid.

## Specification

### Multisig Intent

```typescript
struct MultisigIntent {
    canonicalIntentHash: bytes32;
    signers: vec<SignerIntent>;
    threshold: u32;
    submittedAt: u64;
}
```

### Signer Intent

```typescript
struct SignerIntent {
    signer: address;
    intentHash: bytes32;
    signature: bytes;
}
```

### Validation

```typescript
fn validateMultisigIntent(multisig: MultisigIntent) -> bool {
    if multisig.signers.length < multisig.threshold {
        return false;
    }

    for signer in multisig.signers {
        if signer.intentHash != multisig.canonicalIntentHash {
            return false;
        }
        if !verifySignature(signer.signer, multisig.canonicalIntentHash, signer.signature) {
            return false;
        }
    }

    return true;
}
```

All signers' `intentHash` fields must equal the `canonicalIntentHash`. Any divergence causes the entire transaction to be rejected.

### Partial Signing Flow

```text
1. Proposer creates CanonicalIntent and computes canonicalIntentHash
2. Proposer shares canonicalIntentHash with all signers
3. Each signer independently:
   a. Receives the canonicalIntentHash
   b. Recomputes the hash from their own CanonicalIntent view
   c. Verifies the hashes match
   d. Signs canonicalIntentHash || transactionHash
   e. Returns SignerIntent to proposer
4. Proposer collects SignerIntent entries until threshold is met
5. Proposer submits MultisigIntent on-chain
6. Chain validates all signer hashes match canonicalIntentHash
```

### Expiration Rules

```typescript
struct MultisigConfig {
    intentExpiry: u64;
    maxSignerWindow: u64;
}
```

- `intentExpiry`: maximum time (in seconds) from `submittedAt` until the multisig intent expires
- `maxSignerWindow`: maximum time between the first and last signature

If either window is exceeded, the multisig intent is rejected.

### Rejection States

```typescript
enum MultisigRejection {
    IntentHashMismatch,
    InsufficientSigners,
    InvalidSignature,
    IntentExpired,
    SignerWindowExceeded,
    DuplicateSigner,
    UnauthorizedSigner,
}
```

## Rationale

Requiring all signers to agree on the same canonical intent hash eliminates ambiguity attacks where different participants believe they are signing different operations. The partial signing flow ensures each signer independently verifies the intent before signing.

## Security Considerations

- Signers must verify the canonical intent hash independently, not trust the proposer
- The signer window prevents collecting stale signatures over long periods
- Duplicate signer addresses must be rejected to prevent counting the same signer twice
- The canonical intent hash must include all fields from NRC-16 to prevent partial intent manipulation
