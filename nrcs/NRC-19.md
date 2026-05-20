# NRC-19: Code Change Intent Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Any transaction that changes executable code must be represented as a code-change event with full context about what is changing, why, and how to verify it.

## Motivation

Code changes are the highest-risk operations on any blockchain. A proxy upgrade, account rotation, or runtime swap can redirect funds, change permissions, or break invariants. Existing standards treat code changes as opaque byte replacements. Users and wallets cannot meaningfully assess risk.

NOVA requires that every code change carries structured metadata describing the old code, new code, change type, verification status, and rollback path.

## Specification

### Code Change Intent

```typescript
struct CodeChangeIntent {
    oldCodeHash: bytes32;
    newCodeHash: bytes32;
    contractAddress: address;
    changeType: CodeChangeType;
    verificationStatus: VerificationStatus;
    compilerVersion: string;
    sourceAvailable: bool;
    auditStatus: AuditStatus;
    activationDelay: u64;
    upgradeAuthority: address;
    rollbackPath: option<RollbackPath>;
}
```

### Change Type

```typescript
enum CodeChangeType {
    ProxyUpgrade,
    AccountRotation,
    RuntimeSwap,
    Migration,
    PermissionChange,
    StdlibReplacement,
}
```

### Verification Status

```typescript
enum VerificationStatus {
    Unverified,
    SourceVerified,
    BytecodeMatch,
    FormalProof,
    AuditVerified,
}
```

### Audit Status

```typescript
enum AuditStatus {
    None,
    Pending,
    Completed,
    CompletedWithFindings,
    Failed,
}
```

### Rollback Path

```typescript
struct RollbackPath {
    previousCodeHash: bytes32;
    rollbackDelay: u64;
    rollbackAuthority: address;
}
```

### Display Requirements

Wallets and interfaces must display the following for any code-change transaction:

```text
CODE CHANGE WARNING

Contract:
  <contract_address>

Change Type:
  ProxyUpgrade

Old Code Hash:
  0xabc123...

New Code Hash:
  0xdef456...

Verification:
  SourceVerified

Compiler:
  nova-c 0.5.2

Source Available:
  Yes

Audit Status:
  Completed

Activation Delay:
  48 hours

Upgrade Authority:
  <authority_address>

Rollback:
  Available via <rollback_authority>
  Rollback delay: 24 hours

Risk Level:
  <computed_risk_level>
```

### Risk Classification

```typescript
fn classifyRisk(intent: CodeChangeIntent) -> RiskLevel {
    if intent.changeType == StdlibReplacement {
        return RiskLevel.Critical;
    }
    if intent.changeType == RuntimeSwap {
        return RiskLevel.Critical;
    }
    if intent.changeType == AccountRotation && intent.auditStatus != AuditStatus.Completed {
        return RiskLevel.High;
    }
    if intent.changeType == Migration {
        return RiskLevel.High;
    }
    if intent.verificationStatus == Unverified {
        return RiskLevel.High;
    }
    if !intent.sourceAvailable {
        return RiskLevel.Medium;
    }
    return RiskLevel.Low;
}
```

```typescript
enum RiskLevel {
    Low,
    Medium,
    High,
    Critical,
}
```

### Mandatory Delay

High and Critical risk classifications require a minimum `activationDelay`:

```text
Low:      no mandatory delay
Medium:   minimum 1 hour
High:     minimum 24 hours
Critical: minimum 72 hours
```

## Rationale

Code changes are irreversible in effect even if technically reversible. Structured metadata allows wallets, explorers, and monitoring tools to surface meaningful information to users before they sign.

## Security Considerations

- `oldCodeHash` must match the current on-chain code hash or the transaction is rejected
- `activationDelay` prevents immediate execution of high-risk changes
- `rollbackPath` must reference a previously-deployed and audited code version
- Unverified code changes should require additional attestations per NRC-18
- `upgradeAuthority` must match the on-chain upgrade key or the transaction is rejected
