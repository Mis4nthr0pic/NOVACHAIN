# NRC-42: Post-Quantum Migration Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines mandatory migration timelines from classical to post-quantum cryptographic signatures. Establishes phased deadlines, on-chain quantum readiness scoring, and migration tooling requirements. L3+ accounts must rotate to hybrid or post-quantum signatures within defined windows. Classical-only signatures are downgraded (not rejected) after the deadline. On-chain readiness scores are visible to wallets and counterparties.

## Motivation

NOVA v0.5 section 44 addresses quantum readiness as a protocol-level concern. Classical signature schemes (Ed25519, ECDSA) are vulnerable to quantum attacks via Shor's algorithm. While large-scale quantum computers do not yet exist, the "harvest now, decrypt later" threat means that transactions signed today could be forged retroactively. Migration must begin before quantum computers arrive, not after.

A phased migration allows the ecosystem to transition gradually. Hybrid signatures (classical + post-quantum) provide security during the transition. The quantum readiness score gives wallets and counterparties a way to assess the cryptographic posture of any account. Downgrading classical-only signatures after the deadline incentivizes migration without breaking existing accounts.

## Specification

### MigrationTimeline

```typescript
enum MigrationPhase {
    ClassicalOnly,
    HybridRecommended,
    HybridRequired,
    PQRequired,
}

struct MigrationPhaseConfig {
    phase: MigrationPhase;
    deadline: u64;
    affectedAccountLevels: vec<AccountLevel>;
    description: string,
}

enum AccountLevel {
    L1,
    L2,
    L3,
    L4,
}
```

### Phase Schedule

```typescript
let migrationSchedule: vec<MigrationPhaseConfig> = [
    {
        phase: MigrationPhase::ClassicalOnly,
        deadline: 0,
        affectedAccountLevels: [AccountLevel::L1, AccountLevel::L2, AccountLevel::L3, AccountLevel::L4],
        description: "Classical signatures accepted. Post-quantum signatures also accepted but not required.",
    },
    {
        phase: MigrationPhase::HybridRecommended,
        deadline: 1735689600,
        affectedAccountLevels: [AccountLevel::L3, AccountLevel::L4],
        description: "Hybrid signatures recommended for L3+ accounts. Classical-only accepted with warnings.",
    },
    {
        phase: MigrationPhase::HybridRequired,
        deadline: 1767225600,
        affectedAccountLevels: [AccountLevel::L3, AccountLevel::L4],
        description: "Hybrid signatures required for L3+ accounts. Classical-only downgraded.",
    },
    {
        phase: MigrationPhase::PQRequired,
        deadline: 1798761600,
        affectedAccountLevels: [AccountLevel::L2, AccountLevel::L3, AccountLevel::L4],
        description: "Post-quantum or hybrid signatures required for L2+. Classical-only downgraded.",
    },
];
```

### QuantumReadinessScore

```typescript
struct QuantumReadinessScore {
    account: address;
    currentScheme: SignatureScheme;
    migrationStatus: MigrationStatus;
    score: u8;
    lastChecked: u64,
}

enum SignatureScheme {
    Ed25519,
    ECDSA,
    MLDSA65,
    MLDSA87,
    HybridEd25519MLDSA65,
    HybridECDSAMLDSA65,
    HybridEd25519MLDSA87,
}

enum MigrationStatus {
    NotStarted,
    HybridReady,
    HybridActive,
    PQReady,
    PQActive,
    Migrated,
}
```

### Score Computation

```typescript
fn computeQuantumReadiness(account: address) -> QuantumReadinessScore {
    let scheme = getAccountSignatureScheme(account);
    let score: u8 = match scheme {
        SignatureScheme::Ed25519 => 20,
        SignatureScheme::ECDSA => 20,
        SignatureScheme::MLDSA65 => 90,
        SignatureScheme::MLDSA87 => 100,
        SignatureScheme::HybridEd25519MLDSA65 => 75,
        SignatureScheme::HybridECDSAMLDSA65 => 75,
        SignatureScheme::HybridEd25519MLDSA87 => 85,
    };
    let status = deriveMigrationStatus(scheme);
    return {
        account: account,
        currentScheme: scheme,
        migrationStatus: status,
        score: score,
        lastChecked: currentTimestamp(),
    };
}

fn deriveMigrationStatus(scheme: SignatureScheme) -> MigrationStatus {
    match scheme {
        SignatureScheme::Ed25519 => MigrationStatus::NotStarted,
        SignatureScheme::ECDSA => MigrationStatus::NotStarted,
        SignatureScheme::MLDSA65 => MigrationStatus::PQActive,
        SignatureScheme::MLDSA87 => MigrationStatus::Migrated,
        SignatureScheme::HybridEd25519MLDSA65 => MigrationStatus::HybridActive,
        SignatureScheme::HybridECDSAMLDSA65 => MigrationStatus::HybridActive,
        SignatureScheme::HybridEd25519MLDSA87 => MigrationStatus::HybridActive,
    }
}
```

### Downgrade Behavior

```typescript
struct DowngradeResult {
    account: address;
    previousLevel: AccountLevel;
    newLevel: AccountLevel;
    reason: string,
}

fn applyDowngrade(account: address) -> option<DowngradeResult> {
    let readiness = computeQuantumReadiness(account);
    let currentPhase = getCurrentMigrationPhase();
    let accountLevel = getAccountLevel(account);
    if !currentPhase.affectedAccountLevels.contains(accountLevel) {
        return none;
    }
    if readiness.score >= 75 {
        return none;
    }
    let previousLevel = accountLevel;
    let newLevel = match previousLevel {
        AccountLevel::L4 => AccountLevel::L2,
        AccountLevel::L3 => AccountLevel::L2,
        AccountLevel::L2 => AccountLevel::L1,
        AccountLevel::L1 => AccountLevel::L1,
    };
    setAccountLevel(account, newLevel);
    return some({
        account: account,
        previousLevel: previousLevel,
        newLevel: newLevel,
        reason: "Classical-only signature after migration deadline",
    });
}
```

### Migration Tooling

```typescript
struct MigrationToolConfig {
    targetScheme: SignatureScheme;
    accountId: address;
    newPublicKey: bytes;
    signature: bytes;
    backupKey: option<bytes>,
}

fn executeMigration(config: MigrationToolConfig) -> MigrationResult {
    let account = getAccount(config.accountId);
    let verifyResult = verifyMigrationSignature(config);
    if !verifyResult {
        return MigrationResult::Err("Migration signature verification failed");
    }
    setAccountSignatureScheme(config.accountId, config.targetScheme);
    setAccountPublicKey(config.accountId, config.newPublicKey);
    if config.backupKey.isSome() {
        setAccountBackupKey(config.accountId, config.backupKey.unwrap());
    }
    let readiness = computeQuantumReadiness(config.accountId);
    return MigrationResult::Ok(readiness);
}

enum MigrationResult {
    Ok(QuantumReadinessScore),
    Err(string),
}
```

### Wallet Display

```text
Account: nova1abc...
Quantum Readiness: 75/100

  Current scheme:  Hybrid (Ed25519 + ML-DSA-65)
  Migration status: Hybrid Active
  Last checked:    2025-12-01

  Phase: Hybrid Recommended (deadline: 2026-06-01)
  Your account meets current requirements.

  Next phase: Hybrid Required (deadline: 2027-06-01)
  Recommendation: Rotate to ML-DSA-87 for full readiness.

  [Rotate to ML-DSA-87] [View Migration Guide]
```

### Readiness Registry

```typescript
trait QuantumReadinessRegistry {
    fn getScore(account: address) -> QuantumReadinessScore;
    fn getScores(accounts: vec<address>) -> vec<QuantumReadinessScore>;
    fn getMigrationStatus(account: address) -> MigrationStatus;
    fn getPhaseSchedule() -> vec<MigrationPhaseConfig>;
    fn getCurrentPhase() -> MigrationPhaseConfig;
}
```

## Rationale

Four phases provide a gradual transition from classical to post-quantum. Hybrid signatures during the transition ensure both classical and quantum security. Downgrading rather than rejecting classical-only signatures after deadlines avoids breaking accounts while incentivizing migration. The on-chain readiness score creates transparency: counterparties can assess the quantum risk of any account before transacting.

L3+ accounts are targeted first because they manage higher value and have greater security requirements. The score system is simple (0-100) so that it can be displayed in any wallet interface without specialized knowledge.

## Security Considerations

- Post-quantum signatures (ML-DSA) are larger than classical signatures, increasing transaction size and gas cost. Protocol parameters must account for this.
- Hybrid signatures double the verification cost. The transition period should be as short as practical.
- Downgrade does not revoke access; it reduces account level. Downgraded accounts can still transact but with reduced capabilities and limits.
- The migration deadline assumes that quantum computers will not break classical signatures before the final phase. If quantum computing advances faster than expected, the timeline must be accelerated.
- Migration tooling must handle key generation for post-quantum schemes securely. Poor randomness in PQ key generation is a new attack surface.
- The readiness score is a snapshot, not a guarantee. An account that migrates and then reverts would have a stale score. Scores should be recomputed on every transaction.
