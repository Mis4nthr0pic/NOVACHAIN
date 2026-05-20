# NRC-23: Verified App Manifest

**Status:** Draft  
**Category:** Wallet UX  

## Abstract

Defines a standard for verified application manifests that allow smart accounts to validate frontend integrity before authorizing transactions. A verified contract is insufficient if users interact with a compromised frontend.

## Motivation

Most wallet-draining attacks do not exploit smart contract vulnerabilities. They exploit the frontend. A phishing site can present a legitimate-looking interface while routing funds to an attacker. Contract verification alone does not protect users from this class of attack.

NOVA requires that the smart account can verify the intent originated from an official, unmodified frontend before execution.

## Specification

### AppManifest

```typescript
struct AppManifest {
    appId: bytes32;
    name: string;
    officialDomains: vec<string>;
    frontendBuildHashes: vec<bytes32>;
    ipfsHashes: vec<bytes32>;
    contractAddresses: vec<address>;
    allowedMethods: vec<string>;
    expectedIntentTypes: vec<IntentType>;
    riskPolicy: RiskPolicy;
    frontendSigningKey: FrontendSigningKey;
    registeredAt: u64;
    updatedBy: address;
}
```

### FrontendSigningKey

```typescript
struct FrontendSigningKey {
    keyId: bytes32;
    publicKey: bytes;
    schemeId: u16;
    algorithm: string;
    rotatedAt: u64;
    expiresAt: u64;
    rotationPolicy: KeyRotationPolicy;
    emergencyRevocation: EmergencyRevocation;
}

struct KeyRotationPolicy {
    minRotationInterval: u64;
    maxKeyAge: u64;
    requireTimelockForRotation: bool;
    rotationTimelockDuration: u64;
    previousKeysRetention: u32;
}

struct EmergencyRevocation {
    emergencyRevokers: vec<address>;
    timelockOverrideAllowed: bool;
    revocationCooldown: u64;
    requireMultiSigForEmergency: bool;
    emergencyThreshold: u64;
}
```

### RiskPolicy

```typescript
struct RiskPolicy {
    maxTransferWithoutConfirmation: u256;
    requireExplicitApprovalForNewContracts: bool;
    requireExplicitApprovalForNewMethods: bool;
    warnOnUnverifiedDomain: bool;
    blockOnHashMismatch: bool;
    maxSessionDuration: u64;
}
```

### Intent Validation

```typescript
fn validateIntent(intent: Intent, manifest: AppManifest) -> ValidationResult {
    if !manifest.officialDomains.contains(intent.originDomain) {
        return ValidationResult::UnofficialDomain;
    }

    if !manifest.frontendBuildHashes.contains(intent.frontendHash) {
        return ValidationResult::FrontendHashMismatch;
    }

    if !manifest.contractAddresses.contains(intent.targetContract) {
        return ValidationResult::UnregisteredContract;
    }

    if !manifest.allowedMethods.contains(intent.method) {
        return ValidationResult::UnexpectedMethod;
    }

    if !manifest.expectedIntentTypes.contains(intent.intentType) {
        return ValidationResult::UnexpectedIntentType;
    }

    return verifyFrontendSignature(intent, manifest.frontendSigningKey);
}
```

### Frontend Key Rotation

```typescript
fn rotateFrontendKey(manifest: AppManifest, newKey: FrontendSigningKey, signature: Signature) -> Result {
    let policy = manifest.frontendSigningKey.rotationPolicy;

    if newKey.rotatedAt - manifest.frontendSigningKey.rotatedAt < policy.minRotationInterval {
        return Err::RotationTooFrequent;
    }

    if policy.requireTimelockForRotation {
        scheduleTimelock(policy.rotationTimelockDuration, Action::KeyRotation(newKey));
    }

    if policy.previousKeysRetention > 0 {
        retainPreviousKey(manifest.frontendSigningKey, policy.previousKeysRetention);
    }

    verify(rotationAuthority, signature)?;
    applyKeyRotation(manifest.appId, newKey);
    Ok
}
```

### Emergency Revocation

```typescript
fn emergencyRevokeKey(appId: bytes32, revoker: address, signatures: vec<Signature>) -> Result {
    let manifest = getManifest(appId);
    let revocation = manifest.frontendSigningKey.emergencyRevocation;

    if !revocation.emergencyRevokers.contains(revoker) {
        return Err::NotAuthorizedRevoker;
    }

    if revocation.requireMultiSigForEmergency {
        if signatures.len() < revocation.emergencyThreshold {
            return Err::InsufficientRevocationSignatures;
        }
    }

    revokeKey(appId);
    emitEvent(AppManifestKeyRevoked { appId, revoker, timestamp: now() });
    Ok
}
```

### Wallet Warning Triggers

```typescript
enum WalletWarning {
    DomainNotInManifest,
    FrontendHashMismatch,
    ContractNotRegistered,
    MethodNotAllowed,
    IntentTypeUnexpected,
    SigningKeyExpired,
    SigningKeyRevoked,
    ManifestStale,
    RiskPolicyExceeded,
}

struct WalletWarningDisplay {
    warning: WalletWarning,
    severity: WarningSeverity,
    message: string,
    recommendedAction: string,
}

enum WarningSeverity {
    Info,
    Caution,
    Danger,
    Critical,
}
```

Wallets MUST display warnings before the user signs. Critical-severity warnings MUST require explicit user override.

## Rationale

Contract verification is necessary but not sufficient. The entire trust chain from frontend build to signed intent must be verifiable. Key rotation with timelocks prevents both stale keys and hasty compromises.

## Security Considerations

- Frontend signing keys are high-value targets; rotation policies must be enforced on-chain
- Emergency revocation must be faster than the timelock for key rotation, otherwise a compromised key cannot be stopped quickly enough
- Previous key retention allows grace periods during rotation without creating permanent downgrade attacks
- Manifest staleness (e.g., not updated in 90+ days) should trigger wallet warnings even if hashes technically match
