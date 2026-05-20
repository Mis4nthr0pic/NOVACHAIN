# NRC-33: Encrypted Mempool

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines the configuration and operational parameters for NOVA's encrypted mempool. Transactions are encrypted on submission and remain opaque until a designated reveal point, reducing sandwich attacks, copy-trading, liquidation sniping, governance sniping, and bridge withdrawal targeting.

## Motivation

NOVA v0.5 section 41 identifies that an encrypted mempool reduces MEV extraction by hiding transaction content during the ordering phase. However, encryption does not eliminate all MEV vectors: validator ordering power after reveal, censorship, cross-domain MEV, private orderflow abuse, and latency games remain. This NRC provides the design parameters and honest accounting of limitations.

The encrypted mempool must answer specific operational questions: who can decrypt, when transactions are revealed, whether miners can reorder after reveal, how failed decryptions are handled, how private orderflow is disclosed, what happens if the decryption committee censors, and what metadata remains visible before reveal.

## Specification

### EncryptedMempoolConfig

```typescript
struct EncryptedMempoolConfig {
    encryptionScheme: string;
    committeeSize: u64;
    decryptionThreshold: u64;
    revealEpochLength: u64;
    maxTxLifetime: u64;
    metadataVisibleBeforeReveal: vec<string>;
    postRevealReorderingAllowed: bool;
    failedDecryptionPolicy: FailedDecryptionPolicy;
    privateOrderflowDisclosure: PrivateOrderflowPolicy;
    censorshipResistanceMechanism: string;
}
```

### DecryptionCommittee

```typescript
struct DecryptionCommitteeMember {
    validatorId: bytes32;
    publicKey: bytes;
    registeredAt: u64;
    stake: u256;
    slashHistory: vec<SlashEvent>;
}

struct SlashEvent {
    epoch: u64;
    reason: string;
    penalty: u256;
}

struct DecryptionCommittee {
    members: vec<DecryptionCommitteeMember>;
    threshold: u64;
    rotationEpoch: u64;
}
```

### FailedDecryptionPolicy

```typescript
enum FailedDecryptionPolicy {
    Drop,
    RetryWithNextCommittee,
    RevealPartialAndHalt,
}
```

### PrivateOrderflowPolicy

```typescript
struct PrivateOrderflowPolicy {
    allowed: bool;
    disclosureRequired: bool;
    disclosureUrl: option<string>;
    maxPrivateOrderflowRatio: f64;
}
```

### Transaction Encryption

```typescript
struct EncryptedTransaction {
    ciphertext: bytes;
    ephemeralKey: bytes;
    nonce: bytes;
    commitmentHash: bytes32;
    submitter: option<address>;
    submittedAt: u64;
    revealAtEpoch: u64;
}
```

### Metadata Visible Before Reveal

The following metadata remains visible before decryption:

```typescript
struct PreRevealMetadata {
    txSize: u64;
    submitter: option<address>;
    submittedAt: u64;
    revealAtEpoch: u64;
    gasBid: u256;
    commitmentHash: bytes32;
}
```

### Operational Questions

| Question | Answer |
|---|---|
| Who can decrypt? | Decryption committee members (threshold of committee size) |
| When are transactions revealed? | At epoch boundary defined by revealEpochLength |
| Can miners reorder after reveal? | Governed by postRevealReorderingAllowed |
| How are failed decryptions handled? | Governed by failedDecryptionPolicy |
| How is private orderflow disclosed? | Governed by privateOrderflowPolicy |
| What if committee censors? | Censorship resistance via rotation and slashing |
| What metadata is visible before reveal? | Listed in PreRevealMetadata |

### Honest Limitations

The encrypted mempool does not eliminate:

- Validator ordering power after reveal
- Censorship of encrypted transactions
- Cross-domain MEV (CEX/DEX arbitrage)
- Private orderflow abuse
- Latency games at reveal boundary

## Rationale

Threshold encryption with a rotating committee balances privacy during ordering with accountability for validators. Pre-reveal metadata is minimized but cannot be zero: the network must route and schedule transactions. The failed decryption policy is configurable because different networks may prioritize liveness (retry) or safety (drop).

## Security Considerations

- Committee members who collude above the threshold can decrypt early. Stake and slashing provide economic disincentive.
- Commitment hashes allow submitters to prove their transaction was tampered with after decryption.
- Pre-reveal metadata (especially gasBid) can leak some information about transaction intent. Minimizing visible fields reduces but does not eliminate this.
- Private orderflow that bypasses the encrypted mempool undermines fairness. Disclosure requirements and ratio caps mitigate this.
- Censorship resistance requires committee rotation; static committees become centralization points.
