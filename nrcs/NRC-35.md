# NRC-35: Frontend Integrity Registry

**Status:** Draft  
**Category:** Wallet UX  

## Abstract

Extends NRC-23 with a registry for frontend signing key lifecycle management. Frontend signing keys are a security surface: rotation policy, revocation paths, and emergency authorities must be declared. A stolen frontend signing key should not be a silent compromise.

## Motivation

NOVA v0.5 section 36 establishes that frontend manifests declare signing keys. However, key management is a lifecycle: keys rotate, get compromised, and need revocation. Without a registry, a compromised frontend key can sign malicious manifests silently. Key rotation events must be visible in manifest history, signed by a higher-authority key, so that wallets can detect anomalous rotations.

A frontend signing key is not just an authentication mechanism. It is the trust anchor between a user's wallet and the application they interact with. If that anchor silently changes, the user has no way to know.

## Specification

### FrontendKeyRecord

```typescript
enum KeyStatus {
    Active,
    Revoked,
    Rotating,
}

struct FrontendKeyRecord {
    appId: string;
    signingKey: bytes;
    registeredAt: u64;
    rotationPolicy: string;
    revocationAuthority: address;
    requiresTimelock: bool;
    status: KeyStatus;
    higherAuthorityKey: bytes;
}
```

### Key Rotation Event

```typescript
struct KeyRotationEvent {
    appId: string;
    oldKey: bytes;
    newKey: bytes;
    rotatedAt: u64;
    rotatedBy: address;
    signature: bytes;
    reason: string;
    timelockExpiresAt: option<u64>;
}
```

### Emergency Revocation

```typescript
struct EmergencyRevocation {
    appId: string;
    revokedKey: bytes;
    revokedAt: u64;
    authority: address;
    authoritySignature: bytes;
    reason: string;
    replacementKey: option<bytes>;
}
```

### Frontend Key Registry Interface

```typescript
trait FrontendKeyRegistry {
    fn registerKey(record: FrontendKeyRecord);
    fn getActiveKey(appId: string) -> option<FrontendKeyRecord>;
    fn getKeyHistory(appId: string) -> vec<FrontendKeyRecord>;
    fn initiateRotation(event: KeyRotationEvent);
    fn emergencyRevoke(revocation: EmergencyRevocation);
    fn getRotationHistory(appId: string) -> vec<KeyRotationEvent>;
}
```

### Rotation Validation

```typescript
fn validateRotation(event: KeyRotationEvent) -> bool {
    let record = getActiveKey(event.appId);
    if record.isNone() {
        return false;
    }
    let current = record.unwrap();
    if current.higherAuthorityKey != verifySignature(event.signature) {
        return false;
    }
    if current.requiresTimelock && event.timelockExpiresAt.isSome() {
        let now = currentTimestamp();
        if now < event.timelockExpiresAt.unwrap() {
            return false;
        }
    }
    return true;
}
```

### Wallet Display

Wallets must display key status and rotation history for every app interaction:

```text
App: exchange.nova
Frontend key: Active (registered 2025-06-01)
Key rotations: 3 (last: 2025-11-15)
Revocation authority: nova1auth...
Timelock on rotation: Yes (48h)

Rotation History:
  2025-06-01  Key registered    by nova1deploy...
  2025-09-10  Key rotated       by nova1auth...  (scheduled)
  2025-11-15  Key rotated       by nova1auth...  (scheduled)
```

### Anomalous Rotation Detection

```typescript
fn detectAnomalousRotation(appId: string) -> option<string> {
    let history = getRotationHistory(appId);
    let recentRotations = history.filter(|e| e.rotatedAt > currentTimestamp() - 86400);
    if recentRotations.len() > 1 {
        return some("Multiple key rotations within 24 hours");
    }
    let lastRotation = history.last();
    if lastRotation.isSome() && lastRotation.unwrap().timelockExpiresAt.isNone() {
        let record = getActiveKey(appId);
        if record.isSome() && record.unwrap().requiresTimelock {
            return some("Key rotation without required timelock");
        }
    }
    return none;
}
```

## Rationale

Higher-authority keys provide a chain of trust: the rotating key is not the same as the key authorizing rotation. Timelock requirements on rotation give users a window to detect and respond to unauthorized rotation attempts. Emergency revocation by a separate authority ensures that compromise does not require waiting for a timelock to expire.

## Security Considerations

- The higher-authority key must be stored more securely than the frontend signing key. If both are compromised, the rotation mechanism provides no protection.
- Emergency revocation bypasses timelock. The revocation authority must be a multisig or governance-controlled address.
- Wallets should alert users on any key rotation, not just anomalous ones. Silent rotation is the primary threat model.
- Key history must be append-only. Deletion of rotation events is a red flag.
