# NRC-25: Capability Warm-Up Periods

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines a warm-up mechanism for new spending capabilities. Newly authorized capabilities do not immediately reach full power, limiting damage from compromised or mistakenly granted permissions.

## Motivation

When a user grants a new spending capability, the recipient can immediately drain funds at full capacity. There is no grace period for the user to notice unexpected behavior or cancel the capability before significant damage occurs. NOVA v0.5 section 31.4 introduces warm-up periods so new capabilities ramp up gradually.

## Specification

### WarmUpPhase

```typescript
enum WarmUpPhase {
    Cold,
    Warming,
    Active,
}
```

### WarmUpConfig

```typescript
struct WarmUpConfig {
    capabilityId: bytes32;
    maxAmountDuringWarmUp: u256;
    warmUpDuration: u64;
    alertThreshold: u256;
    cancelable: bool;
    recipientCooldown: u64;
    maxNewRecipientsDuringWarmUp: u32;
}
```

### WarmUpState

```typescript
struct WarmUpState {
    capabilityId: bytes32;
    phase: WarmUpPhase;
    activatedAt: u64;
    warmedUpAt: option<u64>;
    amountUsedDuringWarmUp: u256;
    recipientsUsedDuringWarmUp: vec<address>;
    cancelled: bool;
    cancelledAt: option<u64>;
}
```

### Phase Transitions

```typescript
fn activateCapability(account: address, config: WarmUpConfig) -> WarmUpState {
    WarmUpState {
        capabilityId: config.capabilityId,
        phase: WarmUpPhase::Cold,
        activatedAt: now(),
        warmedUpAt: None,
        amountUsedDuringWarmUp: 0,
        recipientsUsedDuringWarmUp: vec![],
        cancelled: false,
        cancelledAt: None,
    }
}

fn transitionPhase(state: WarmUpState, config: WarmUpConfig) -> WarmUpPhase {
    if state.cancelled {
        return WarmUpPhase::Cold;
    }

    let elapsed = now() - state.activatedAt;

    if elapsed < 300 {
        return WarmUpPhase::Cold;
    }

    if elapsed < config.warmUpDuration {
        return WarmUpPhase::Warming;
    }

    WarmUpPhase::Active
}
```

### Spending Rules During Warm-Up

```typescript
fn validateWarmUpSpend(state: WarmUpState, config: WarmUpConfig, amount: u256, recipient: address) -> Result {
    if state.cancelled {
        return Err::CapabilityCancelled;
    }

    let phase = transitionPhase(state, config);

    match phase {
        WarmUpPhase::Cold => {
            return Err::CapabilityStillCold;
        }
        WarmUpPhase::Warming => {
            if amount > config.maxAmountDuringWarmUp {
                return Err::AmountExceedsWarmUpLimit;
            }

            if state.amountUsedDuringWarmUp + amount > config.alertThreshold {
                emitEvent(WarmUpAlertThresholdReached {
                    capabilityId: config.capabilityId,
                    used: state.amountUsedDuringWarmUp + amount,
                    threshold: config.alertThreshold,
                });
            }

            let isNewRecipient = !state.recipientsUsedDuringWarmUp.contains(recipient);
            if isNewRecipient {
                if state.recipientsUsedDuringWarmUp.len() >= config.maxNewRecipientsDuringWarmUp {
                    return Err::TooManyNewRecipientsDuringWarmUp;
                }
            }
        }
        WarmUpPhase::Active => {}
    }

    Ok
}
```

### Cancellation

```typescript
fn cancelCapability(state: WarmUpState, config: WarmUpConfig, canceller: address) -> Result {
    if !config.cancelable {
        return Err::CapabilityNotCancelable;
    }

    match state.phase {
        WarmUpPhase::Active => {
            return Err::CapabilityAlreadyActive;
        }
        _ => {}
    }

    WarmUpState {
        cancelled: true,
        cancelledAt: Some(now()),
        ..state
    };

    emitEvent(CapabilityCancelled { capabilityId: state.capabilityId, canceller, timestamp: now() });
    Ok
}
```

### New Recipient Cooldown

```typescript
fn validateNewRecipient(state: WarmUpState, config: WarmUpConfig, recipient: address, amount: u256) -> Result {
    let isNewRecipient = !state.recipientsUsedDuringWarmUp.contains(recipient);

    if isNewRecipient && state.phase == WarmUpPhase::Warming {
        if amount > config.maxAmountDuringWarmUp / 2 {
            return Err::HighValueTransferToNewRecipientBlocked;
        }

        emitEvent(NewRecipientDuringWarmUp {
            capabilityId: state.capabilityId,
            recipient,
            amount,
            timestamp: now(),
        });
    }

    Ok
}
```

## Rationale

Warm-up periods create a temporal safety net. If a capability is granted by mistake or through social engineering, the limited initial capacity gives the user time to notice and cancel. The Cold phase provides an immediate undo window, while the Warming phase allows low-value use to verify correct behavior.

## Security Considerations

- Warm-up duration must not be configurable to zero by the capability itself, only by the account owner
- Alert thresholds should trigger wallet notifications, not just on-chain events
- Cancellation must be free or nearly free to remove any cost barrier to revoking a malicious capability
- The Cold phase (first 5 minutes) should be treated as a mandatory undo window
