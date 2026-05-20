# NRC-21: Custody Account Standard

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines a canonical custody account type for exchanges, treasuries, bridges, and DAOs with separate operational roles, spending limits, anomaly detection, and pause capabilities.

## Motivation

Custody accounts manage funds on behalf of others. Historically, custody failures stem from:

- Single-key control over large treasuries
- No separation between operational and policy roles
- No automated anomaly detection
- No structured pause mechanism
- Delay-free withdrawals to new addresses

NOVA defines a custody account standard that enforces separation of powers, rate limits, and anomaly response.

## Specification

### Custody Configuration

```typescript
struct CustodyConfig {
    dailyLimit: u256;
    weeklyLimit: u256;
    newRecipientCooldown: u64;
    largeTransferDelay: u64;
    operationalKeys: vec<address>;
    policyKeys: vec<address>;
    upgradeKeys: vec<address>;
    withdrawalKeys: vec<address>;
    recoveryKeys: vec<address>;
    pauseOnAnomaly: bool;
    signerRotationDelay: u64;
    hardwareSignerRequired: bool;
}
```

### Separate Powers Model

```text
operationalKeys:  submit transactions, queue transfers
policyKeys:       approve limits, manage recipients, configure rules
upgradeKeys:      modify custody account code
withdrawalKeys:   authorize outgoing transfers above threshold
recoveryKeys:     initiate account recovery, override pause

No key set may overlap with another.
```

```typescript
fn validateKeySeparation(config: CustodyConfig) -> bool {
    let allKeys: vec<address> = [];
    for key in config.operationalKeys { allKeys.push(key); }
    for key in config.policyKeys { allKeys.push(key); }
    for key in config.upgradeKeys { allKeys.push(key); }
    for key in config.withdrawalKeys { allKeys.push(key); }
    for key in config.recoveryKeys { allKeys.push(key); }
    return allKeys.length == unique(allKeys).length;
}
```

### Pause Rule

```typescript
struct PauseRule {
    trigger: PauseTrigger;
    autoPause: bool;
    notificationEndpoint: string;
    cooldown: u64;
}
```

```typescript
enum PauseTrigger {
    DailyLimitExceeded,
    UnauthorizedTransferAttempt,
    KeyCompromise,
    UnusualVolume,
    NewRecipientLargeTransfer,
    SimulationMismatch,
    GovernanceOverride,
}
```

### Pause Mechanism

```typescript
struct CustodyPause {
    active: bool;
    triggeredBy: address;
    trigger: PauseTrigger;
    triggeredAt: u64;
    cooldownEnd: u64;
}
```

```typescript
fn pauseCustody(trigger: PauseTrigger, config: CustodyConfig) -> CustodyPause {
    let matchingRule = config.pauseRules.find(|r| r.trigger == trigger);
    if matchingRule.is_some() && matchingRule.unwrap().autoPause {
        return CustodyPause {
            active: true,
            triggeredBy: msg.sender,
            trigger: trigger,
            triggeredAt: now,
            cooldownEnd: now + matchingRule.unwrap().cooldown,
        };
    }
    return CustodyPause { active: false, ... };
}
```

### Anomaly Detection Triggers

```typescript
struct AnomalyDetector {
    dailyVolume: u256;
    dailyTransferCount: u32;
    newRecipientCount: u32;
    failedAuthAttempts: u32;
    lastReset: u64,
}
```

An anomaly is flagged when any of these conditions are true:

```text
1. dailyVolume > config.dailyLimit
2. dailyTransferCount > 1000
3. newRecipientCount > 50 in a single day
4. failedAuthAttempts > 10 in a single hour
5. Transfer to a sanctioned or flagged address
6. Transfer pattern matches known drain signatures
```

### Spending Limits

```typescript
fn checkSpendingLimits(
    transferAmount: u256,
    detector: AnomalyDetector,
    config: CustodyConfig,
) -> bool {
    if detector.dailyVolume + transferAmount > config.dailyLimit {
        return false;
    }
    if detector.weeklyVolume + transferAmount > config.weeklyLimit {
        return false;
    }
    return true;
}
```

### Signer Rotation

When rotating any key set, the `signerRotationDelay` must elapse before the new key becomes active:

```text
1. policyKey proposes rotation
2. Rotation is queued with activation timestamp = now + signerRotationDelay
3. During delay, existing keys remain active
4. Any recoveryKey can cancel the rotation during the delay
5. After delay, new keys become active
```

## Rationale

Separation of powers ensures no single role can unilaterally drain funds. Operational keys can submit but not approve large transfers. Policy keys configure rules but cannot move funds. Withdrawal keys approve transfers but cannot change rules. This prevents a compromise of any single role from resulting in fund loss.

## Security Considerations

- Key set overlap must be rejected at configuration time
- `pauseOnAnomaly` must be true for L4 custody accounts
- Recovery keys must be stored in hardware security modules
- All configuration changes must be delayed per NRC-20 rules
- The anomaly detector must reset daily to prevent permanent lockout
- Pause cooldown must have a maximum to prevent indefinite freezing
