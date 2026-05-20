# NRC-43: Shielded Intent Privacy (Optional)

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines an optional privacy layer using hybrid zero-knowledge proofs for intent shielding. NOT mandatory. Transparent mode remains the default. Shielded intents use zero-knowledge proofs to hide sender, receiver, and amount while maintaining compliance with NOVA's typed intent model. Auditors and regulators can be granted viewing keys. Shielded intents still go through typed intent validation and risk labels.

## Motivation

NOVA v0.5 section 45 addresses optional privacy as a protocol extension. Today, all on-chain transactions are fully transparent. This is appropriate for most use cases but creates problems for commercial activity where transaction details are competitively sensitive, and for individual privacy where financial history should not be publicly inspectable.

Privacy must be optional, not mandatory. Transparent mode is the default and must always be available. Shielded mode uses zero-knowledge proofs to hide transaction details while preserving the typed intent model, risk labeling, and compliance infrastructure. Viewing keys allow authorized auditors and regulators to inspect shielded transactions without breaking the privacy guarantee for other observers.

## Specification

### ShieldedConfig

```typescript
struct ShieldedConfig {
    enabled: bool;
    proofSystem: ProofSystem;
    privacyLevel: PrivacyLevel;
    viewingKeys: vec<ViewingKeyGrant>,
}

enum ProofSystem {
    Groth16,
    PLONK,
}

enum PrivacyLevel {
    Transparent,
    ShieldedTransfers,
    FullShield,
}
```

### ViewingKeyGrant

```typescript
struct ViewingKeyGrant {
    grantee: address;
    granteeType: GranteeType;
    scope: ViewingScope;
    grantedAt: u64;
    expiresAt: u64,
}

enum GranteeType {
    Auditor,
    Regulator,
    ComplianceOfficer,
}

enum ViewingScope {
    AllTransactions,
    TransactionsInRange { start: u64, end: u64 },
    SpecificTokens { tokens: vec<address> },
}
```

### ShieldedIntent

```typescript
struct ShieldedIntent {
    commitmentHash: bytes32;
    nullifier: bytes32;
    zkProof: bytes;
    encryptedPayload: bytes;
    proofSystem: ProofSystem;
    privacyLevel: PrivacyLevel;
    intentTypeHash: bytes32,
}
```

### Shielded Transfer Flow

```typescript
fn createShieldedTransfer(
    sender: address,
    receiver: address,
    amount: u256,
    token: address,
    config: ShieldedConfig,
) -> ShieldedIntent {
    let commitment = computeCommitment(receiver, amount, token, generateNonce());
    let nullifier = computeNullifier(sender, commitment);
    let zkProof = generateZKProof({
        sender: sender,
        receiver: receiver,
        amount: amount,
        token: token,
        commitment: commitment,
        nullifier: nullifier,
        merkleRoot: getMerkleRoot(),
        proofSystem: config.proofSystem,
    });
    let payload = encryptPayload({
        sender: sender,
        receiver: receiver,
        amount: amount,
        token: token,
    }, getViewingPublicKeys(config));
    return {
        commitmentHash: commitment,
        nullifier: nullifier,
        zkProof: zkProof,
        encryptedPayload: payload,
        proofSystem: config.proofSystem,
        privacyLevel: config.privacyLevel,
        intentTypeHash: INTENT_TYPE_HASH,
    };
}
```

### Proof Verification

```typescript
fn verifyShieldedIntent(intent: ShieldedIntent) -> ShieldedVerificationResult {
    if !verifyZKProof(intent.zkProof, intent.proofSystem) {
        return { valid: false, reason: some("Invalid zero-knowledge proof") };
    }
    if isNullifierSpent(intent.nullifier) {
        return { valid: false, reason: some("Nullifier already spent") };
    }
    let typeCheck = verifyIntentType(intent.intentTypeHash);
    if !typeCheck {
        return { valid: false, reason: some("Intent type hash mismatch") };
    }
    return { valid: true, reason: none };
}

struct ShieldedVerificationResult {
    valid: bool;
    reason: option<string>,
}
```

### Nullifier Management

```typescript
struct NullifierRecord {
    nullifier: bytes32;
    spentAt: u64;
    spentBy: bytes32,
}

trait NullifierStore {
    fn isNullifierSpent(nullifier: bytes32) -> bool;
    fn markNullifierSpent(record: NullifierRecord);
    fn getNullifierHistory(nullifier: bytes32) -> option<NullifierRecord>;
}
```

### Viewing Key Decryption

```typescript
fn decryptShieldedTransaction(
    intent: ShieldedIntent,
    viewingKey: bytes,
    grant: ViewingKeyGrant,
) -> Option<ShieldedTransactionDetail> {
    if !validateViewingKeyGrant(grant) {
        return none;
    }
    let decrypted = decryptPayload(intent.encryptedPayload, viewingKey);
    if decrypted.isNone() {
        return none;
    }
    return some(parseTransactionDetail(decrypted.unwrap()));
}

struct ShieldedTransactionDetail {
    sender: address;
    receiver: address;
    amount: u256;
    token: address;
    timestamp: u64,
}
```

### Compliance Integration

Shielded intents must still pass typed intent validation and risk labeling:

```typescript
fn processShieldedIntent(intent: ShieldedIntent, riskLabels: vec<RiskLabel>) -> ProcessingResult {
    let verification = verifyShieldedIntent(intent);
    if !verification.valid {
        return ProcessingResult::Rejected(verification.reason.unwrap());
    }
    let riskCheck = evaluateRiskLabels(riskLabels);
    if riskCheck.requiresReview {
        emitComplianceEvent({
            intentHash: intent.commitmentHash,
            riskLabels: riskLabels,
            action: "review_required",
        });
        return ProcessingResult::PendingReview;
    }
    markNullifierSpent({
        nullifier: intent.nullifier,
        spentAt: currentTimestamp(),
        spentBy: intent.commitmentHash,
    });
    return ProcessingResult::Accepted;
}

enum ProcessingResult {
    Accepted,
    PendingReview,
    Rejected(string),
}
```

### Privacy Level Details

```text
Transparent:       No shielding applied. Standard typed intents.
                   All data visible on-chain. Default mode.

ShieldedTransfers: Transfer amounts and counterparties hidden.
                   Intent type still visible. Risk labels still applied.
                   Suitable for commercial privacy.

FullShield:        Intent type, amounts, and counterparties hidden.
                   Only commitment hash and nullifier visible.
                   Risk labels applied to encrypted payload.
                   Viewing keys grant selective disclosure.
```

### Wallet Display

```text
Shielded Transfer

  Mode:           Shielded Transfers (Groth16)
  Commitment:     0x4a7f...e2c1
  Nullifier:      0x8b3d...f1a9
  Proof:          Valid

  Visible to:     You, 2 auditors, 1 regulator
  Viewing keys:   Active (expires 2026-06-01)

  Risk assessment: Passed (no labels triggered)
  Compliance:      Standard review

  [Decrypt Details] [Revoke Viewing Key]
```

## Rationale

Privacy is optional because not every user or application needs it, and mandatory privacy creates friction and regulatory risk. Transparent mode remains the default. Shielded modes use zero-knowledge proofs to hide transaction details while preserving the typed intent model, which ensures that privacy does not bypass risk assessment or compliance.

Viewing keys allow selective disclosure. Auditors and regulators can inspect shielded transactions without breaking privacy for the broader public. This balances privacy with accountability. The two shielded levels (ShieldedTransfers and FullShield) let users choose how much privacy they need.

## Security Considerations

- Zero-knowledge proof systems have trusted setup requirements (Groth16) or larger proof sizes (PLONK). The choice of proof system affects both security and performance.
- Nullifier reuse detection is critical. A double-spend in a shielded context would bypass the privacy guarantees. The nullifier store must be append-only and consensus-enforced.
- Viewing key grants create a disclosure surface. Compromised viewing keys break privacy for all granted transactions. Key management for grantees must follow strict security practices.
- Shielded intents that bypass risk labeling create a compliance gap. The enforcement of risk labels on shielded intents must be verified at the protocol level, not trusted to client-side code.
- The encrypted payload size depends on the number of viewing key grantees. Large grantee lists increase transaction size and gas cost.
- FullShield mode hides the intent type, which limits the ability of on-chain monitors (NRC-41) to detect anomalies. Monitoring hooks for shielded transactions must work with commitment hashes and nullifiers rather than transaction content.
