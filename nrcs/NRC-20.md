# NRC-20: Account Security Levels

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines five security levels (L0 through L4) that accounts declare to control transaction risk thresholds, required attestations, delay periods, and key management requirements.

## Motivation

A single security policy does not fit all accounts. A user with 100 NOVA has different needs than a DAO treasury with 10 million NOVA. Forcing all accounts through the same friction either under-protects high-value accounts or over-burdens low-value accounts.

Security levels let accounts declare their risk tolerance. Higher levels add progressively stricter controls.

## Specification

### Security Level

```typescript
enum SecurityLevel {
    L0,
    L1,
    L2,
    L3,
    L4,
}
```

```text
L0: Personal accounts with low balances
L1: Personal accounts with moderate balances
L2: Business accounts, high-value personal accounts
L3: Exchange hot wallets, treasury operations, DAOs
L4: Custody accounts, bridge operators, protocol treasuries
```

### Security Policy

```typescript
struct SecurityPolicy {
    level: SecurityLevel;
    maxTransferWithoutDelay: u256;
    multisigThreshold: u32;
    requiredSigners: u32;
    newRecipientCooldown: u64;
    largeTransferDelay: u64;
    upgradeDelay: u64;
    requiresHardwareAttestation: bool;
    requiresSimulationAttestation: bool;
    recoveryDelay: u64;
}
```

### Default Policies

```typescript
let L0_POLICY: SecurityPolicy = {
    level: SecurityLevel.L0,
    maxTransferWithoutDelay: 1000_00000000,
    multisigThreshold: 1,
    requiredSigners: 1,
    newRecipientCooldown: 0,
    largeTransferDelay: 0,
    upgradeDelay: 3600,
    requiresHardwareAttestation: false,
    requiresSimulationAttestation: false,
    recoveryDelay: 86400,
};

let L1_POLICY: SecurityPolicy = {
    level: SecurityLevel.L1,
    maxTransferWithoutDelay: 10000_00000000,
    multisigThreshold: 1,
    requiredSigners: 1,
    newRecipientCooldown: 300,
    largeTransferDelay: 3600,
    upgradeDelay: 86400,
    requiresHardwareAttestation: false,
    requiresSimulationAttestation: false,
    recoveryDelay: 172800,
};

let L2_POLICY: SecurityPolicy = {
    level: SecurityLevel.L2,
    maxTransferWithoutDelay: 100000_00000000,
    multisigThreshold: 2,
    requiredSigners: 3,
    newRecipientCooldown: 3600,
    largeTransferDelay: 14400,
    upgradeDelay: 172800,
    requiresHardwareAttestation: false,
    requiresSimulationAttestation: true,
    recoveryDelay: 604800,
};

let L3_POLICY: SecurityPolicy = {
    level: SecurityLevel.L3,
    maxTransferWithoutDelay: 500000_00000000,
    multisigThreshold: 3,
    requiredSigners: 5,
    newRecipientCooldown: 86400,
    largeTransferDelay: 86400,
    upgradeDelay: 604800,
    requiresHardwareAttestation: true,
    requiresSimulationAttestation: true,
    recoveryDelay: 1209600,
};

let L4_POLICY: SecurityPolicy = {
    level: SecurityLevel.L4,
    maxTransferWithoutDelay: 0,
    multisigThreshold: 4,
    requiredSigners: 7,
    newRecipientCooldown: 604800,
    largeTransferDelay: 172800,
    upgradeDelay: 1209600,
    requiresHardwareAttestation: true,
    requiresSimulationAttestation: true,
    recoveryDelay: 2592000,
};
```

### Level Upgrade Rules

```typescript
fn upgradeLevel(current: SecurityLevel, target: SecurityLevel) -> UpgradeResult {
    if target as u32 <= current as u32 {
        return UpgradeResult.InvalidTarget;
    }

    let delay: u64;
    match target {
        SecurityLevel.L1 => { delay = 0; },
        SecurityLevel.L2 => { delay = 86400; },
        SecurityLevel.L3 => { delay = 604800; },
        SecurityLevel.L4 => { delay = 1209600; },
        _ => { return UpgradeResult.InvalidTarget; },
    }

    return UpgradeResult.Pending(delay);
}
```

Upgrading to a higher level requires a delay proportional to the target level. During the delay, the account operates at the current level.

### Level Downgrade Rules

```typescript
fn downgradeLevel(current: SecurityLevel, target: SecurityLevel) -> UpgradeResult {
    if target as u32 >= current as u32 {
        return UpgradeResult.InvalidTarget;
    }

    let delay: u64;
    match target {
        SecurityLevel.L0 => { delay = 604800; },
        SecurityLevel.L1 => { delay = 604800; },
        SecurityLevel.L2 => { delay = 2592000; },
        SecurityLevel.L3 => { delay = 2592000; },
        _ => { return UpgradeResult.InvalidTarget; },
    }

    return UpgradeResult.Pending(delay);
}
```

Downgrading requires a longer delay than upgrading. Downgrade requests must be broadcast to all registered notification endpoints.

### New Recipient Cooldown

When `newRecipientCooldown > 0`, transfers to a previously unseen recipient address are delayed:

```text
1. First transfer to new recipient is queued
2. Queued transfer becomes executable after newRecipientCooldown
3. Subsequent transfers to the same recipient execute immediately
4. Cooldown resets if the recipient is not used for 30 days
```

### Capability Warm-Up Period

New capabilities granted to an account are subject to a warm-up period:

```typescript
fn capabilityWarmUp(level: SecurityLevel) -> u64 {
    match level {
        SecurityLevel.L0 => 0,
        SecurityLevel.L1 => 300,
        SecurityLevel.L2 => 3600,
        SecurityLevel.L3 => 86400,
        SecurityLevel.L4 => 172800,
    }
}
```

During the warm-up period, the capability exists but cannot be exercised.

## Rationale

Progressive security levels let users start with low friction and add protection as their account value grows. The asymmetric delay (fast upgrade, slow downgrade) ensures accounts move to higher security quickly but cannot rashly lower protections.

## Security Considerations

- Level downgrade notifications must be sent to all registered endpoints to detect unauthorized downgrade attempts
- The `maxTransferWithoutDelay` of 0 for L4 means every transfer requires a delay
- Capability warm-up prevents granting and immediately exercising dangerous permissions
- Recovery delays must be longer than any other delay to give guardians time to intervene
