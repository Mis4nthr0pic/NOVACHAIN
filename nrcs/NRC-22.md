# NRC-22: Application Contract Upgrade Standard

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines a visible, structured upgrade flow for high-value application contracts with risk classification, diff exposure, and enforced delay periods.

## Motivation

Application contract upgrades affect all users who interact with the contract. Without a standard:

- Users cannot see what changed
- Developers can instantly replace trusted code with malicious code
- No mechanism exists for users to withdraw before an upgrade takes effect
- Risk is opaque because diffs are not exposed on-chain

NOVA requires that upgrades to high-value contracts follow a visible flow with explicit diffs, risk classification, and time for users to respond.

## Specification

### Upgrade Flow

```typescript
enum UpgradeStep {
    Proposed,
    DiffPublished,
    RiskClassified,
    DelayStarted,
    AttestationsCollected,
    Activated,
    RolledBack,
    Cancelled,
}
```

```text
Flow:
1. Proposed:        upgradeAuthority submits upgrade proposal
2. DiffPublished:   full UpgradeDiff is published on-chain
3. RiskClassified:  risk class is computed from the diff
4. DelayStarted:    mandatory delay begins based on risk class
5. AttestationsCollected: simulation attestations gathered (if required)
6. Activated:       new code becomes active after delay
7. RolledBack:      upgrade reverted if critical issue found during delay
8. Cancelled:       upgradeAuthority cancels before activation
```

### Upgrade Risk Class

```typescript
enum UpgradeRiskClass {
    Low,
    Medium,
    High,
    Critical,
}
```

```text
Suggested delays:
Low:      1 hour
Medium:   24 hours
High:     72 hours
Critical: 168 hours (7 days)
```

### Upgrade Diff

```typescript
struct UpgradeDiff {
    codeChanges: vec<CodeChange>;
    permissionChanges: vec<PermissionChange>;
    storageChanges: vec<StorageLayoutChange>;
    assetMovements: vec<ExpectedAssetMovement>;
    affectedUsers: u64;
    withdrawalBehaviorChanges: bool;
    adminPowerChanges: bool;
    oracleLogicChanges: bool;
    bridgeLogicChanges: bool;
}
```

### Diff Components

```typescript
struct CodeChange {
    functionSelector: bytes4;
    oldBytecodeHash: bytes32;
    newBytecodeHash: bytes32;
    changeDescription: string;
}

struct PermissionChange {
    role: bytes32;
    oldPermissions: vec<bytes4>;
    newPermissions: vec<bytes4>;
}

struct StorageLayoutChange {
    slot: bytes32;
    oldType: string;
    newType: string;
    migrationRequired: bool;
}

struct ExpectedAssetMovement {
    asset: address;
    from: address;
    to: address;
    maxAmount: u256;
}

struct WithdrawalBehaviorChange {
    oldWithdrawalConditions: bytes;
    newWithdrawalConditions: bytes;
    breaking: bool;
}
```

### Risk Classification

```typescript
fn classifyUpgradeRisk(diff: UpgradeDiff) -> UpgradeRiskClass {
    let score: u32 = 0;

    if diff.withdrawalBehaviorChanges { score += 4; }
    if diff.adminPowerChanges { score += 3; }
    if diff.bridgeLogicChanges { score += 3; }
    if diff.oracleLogicChanges { score += 2; }
    if diff.storageChanges.length > 0 { score += 2; }
    if diff.assetMovements.length > 0 { score += 2; }
    if diff.permissionChanges.length > 0 { score += 1; }
    if diff.affectedUsers > 10000 { score += 2; }

    if score >= 8 { return UpgradeRiskClass.Critical; }
    if score >= 5 { return UpgradeRiskClass.High; }
    if score >= 3 { return UpgradeRiskClass.Medium; }
    return UpgradeRiskClass.Low;
}
```

### Upgrade Label

```typescript
enum UpgradeLabel {
    Immutable,
    UpgradeableWithDelay,
    UpgradeableAfterVerification,
    UpgradeableInstantlyByMultisig,
    UpgradeableInstantlyBySingleKey,
    UnknownUpgradePath,
}
```

```text
Immutable:                      no upgrade mechanism exists
UpgradeableWithDelay:           upgrade requires a time delay
UpgradeableAfterVerification:   upgrade requires simulation attestation + delay
UpgradeableInstantlyByMultisig: upgrade by multisig with no delay (minimum N-of-M required)
UpgradeableInstantlyBySingleKey: upgrade by single key with no delay (highest risk)
UnknownUpgradePath:             upgrade mechanism cannot be determined
```

### User Notification

Contracts with `affectedUsers > 0` must emit:

```typescript
struct UpgradeNotification {
    contract: address;
    upgradeId: bytes32;
    riskClass: UpgradeRiskClass;
    activationAt: u64;
    diff: UpgradeDiff;
    rollbackAvailable: bool;
}
```

### Emergency Rollback

```typescript
fn emergencyRollback(upgradeId: bytes32, reason: string) -> bool {
    let upgrade = upgrades[upgradeId];
    require(msg.sender == upgrade.upgradeAuthority);
    require(upgrade.step == UpgradeStep.Activated);
    require(now < upgrade.activatedAt + ROLLBACK_WINDOW);

    code_storage[upgrade.contract] = upgrade.oldCodeHash;
    upgrade.step = UpgradeStep.RolledBack;

    emit UpgradeRolledBack {
        upgradeId: upgradeId,
        reason: reason,
        rolledBackAt: now,
    };

    return true;
}
```

## Rationale

Visible upgrade flows give users time to assess risk and withdraw if necessary. The scoring system provides objective risk classification. Emergency rollback provides a safety net for upgrades that introduce critical bugs.

## Security Considerations

- `UpgradeableInstantlyBySingleKey` should be flagged as highest risk by all wallets
- `UnknownUpgradePath` should be treated as equivalent to `UpgradeableInstantlyBySingleKey`
- Rollback window must be bounded to prevent indefinite rollback capability
- `affectedUsers` should be computed on-chain or verified through simulation attestation
- Storage layout changes that are not backward-compatible can brick existing data
- Asset movements in the diff allow users to verify no unauthorized transfers occur during upgrade
