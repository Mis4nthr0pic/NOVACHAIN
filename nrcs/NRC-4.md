# NRC-4: Guardian Recovery

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines social recovery and guardian-set behavior for NOVA smart accounts.

## Motivation

Lost keys should not mean lost funds. NOVA accounts support guardian-based social recovery, allowing trusted contacts or institutions to help restore access.

## Specification

### Guardian Set

```typescript
struct GuardianSet {
    guardians: vec<address>;
    threshold: u64;
    timelock: u64;
    active: bool;
}
```

### Recovery Process

```text
1. Guardian submits recovery request
2. Enough guardians confirm (>= threshold)
3. Recovery enters timelock period
4. Account owner can cancel during timelock
5. After timelock, new auth root is activated
```

### Recovery Request

```typescript
struct RecoveryRequest {
    account: address;
    newAuthRoot: bytes32;
    guardiansApproved: vec<address>;
    submittedAt: u64;
    executeAt: u64;
    cancelled: bool;
}
```

### Rules

- Guardian threshold must be > 50% of guardian count
- Minimum timelock: 24 hours for standard accounts
- High-value accounts may require longer timelocks (configurable)
- Account owner can cancel recovery during timelock
- Guardian sets can be rotated by the account owner
- A guardian cannot be the account itself

### Guardian Rotation

```typescript
fn rotateGuardians(newGuardians: vec<address>, newThreshold: u64) {
    if msg.sender != self.owner { revert NotOwner; }
    if newThreshold <= newGuardians.len() / 2 { revert ThresholdTooLow; }
    self.guardians = GuardianSet {
        guardians: newGuardians,
        threshold: newThreshold,
        timelock: self.guardians.timelock,
        active: true,
    };
}
```

## Rationale

Social recovery avoids the single-point-of-failure of seed phrases while keeping the account address stable.

## Security Considerations

- Guardians colluding could hijack an account; threshold design mitigates this
- Timelock gives the legitimate owner time to cancel malicious recovery
- Guardians should not know each other to prevent collusion
