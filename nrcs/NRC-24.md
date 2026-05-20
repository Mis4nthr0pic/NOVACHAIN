# NRC-24: Hardware Signer Attestation

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines a standard for hardware-backed signing attestation, allowing smart accounts to require proof that high-value actions were signed by a registered hardware device.

## Motivation

Software keys can be exfiltrated by malware, clipboard attackers, or browser exploits. For high-value actions, smart accounts need cryptographic proof that the signing operation occurred inside a hardware security module. NOVA v0.5 section 29.5 specifies that smart accounts can require hardware-backed signing for sensitive operations.

## Specification

### HardwareAttestation

```typescript
struct HardwareAttestation {
    deviceId: bytes32;
    attestationCertificate: bytes;
    firmwareVersion: string;
    manufacturer: HardwareManufacturer;
    timestamp: u64;
    challengeNonce: bytes32;
    signature: bytes;
    attestationType: AttestationType;
}

enum HardwareManufacturer {
    Ledger,
    Trezor,
    YubiKey,
    Keystone,
    GridPlus,
    Custom,
}

enum AttestationType {
    WebAuthn,
    PlatformAuthenticator,
    FidoU2F,
    TpmDirect,
    SecureEnclave,
}
```

### AttestationPolicy

```typescript
struct AttestationPolicy {
    registeredDevices: vec<DeviceRegistration>;
    approvedFirmwareVersions: vec<FirmwareVersion>;
    requireAttestationForTransfersAbove: u256;
    requireAttestationForUpgrades: bool;
    requireAttestationForNewSigners: bool;
    requireAttestationForGuardianChanges: bool;
    rejectBrowserWalletSignatures: bool;
    maxAttestationAge: u64;
    requireFreshAttestation: bool;
}

struct DeviceRegistration {
    deviceId: bytes32;
    manufacturer: HardwareManufacturer;
    model: string;
    registeredAt: u64;
    publicKey: bytes;
    schemeId: u16;
    label: string;
    active: bool;
}

struct FirmwareVersion {
    manufacturer: HardwareManufacturer;
    model: string;
    minVersion: string,
    maxVersion: option<string>,
    approvedAt: u64,
    knownVulnerabilities: vec<string>,
}
```

### Device Registration

```typescript
fn registerDevice(account: address, device: DeviceRegistration, attestation: HardwareAttestation, existingSignature: Signature) -> Result {
    verifyAccountAuthority(existingSignature)?;

    if !verifyAttestation(attestation, device) {
        return Err::InvalidAttestation;
    }

    if getDeviceRegistration(account, device.deviceId).is_some() {
        return Err::DeviceAlreadyRegistered;
    }

    let policy = getAttestationPolicy(account);
    let maxDevices = 5;

    if countActiveDevices(account) >= maxDevices {
        return Err::DeviceLimitReached;
    }

    storeDeviceRegistration(account, device);
    emitEvent(DeviceRegistered { account, deviceId: device.deviceId, manufacturer: device.manufacturer });
    Ok
}
```

### Firmware Approval

```typescript
fn approveFirmwareVersion(version: FirmwareVersion, approver: address, signatures: vec<Signature>) -> Result {
    verifyProtocolGovernance(signatures)?;

    if version.knownVulnerabilities.len() > 0 {
        emitEvent(FirmwareVersionApprovedWithVulnerabilities { version, approver });
    }

    storeFirmwareApproval(version);
    emitEvent(FirmwareVersionApproved { version, approver });
    Ok
}

fn isFirmwareApproved(manufacturer: HardwareManufacturer, model: string, firmwareVersion: string) -> bool {
    let approved = getApprovedFirmware(manufacturer, model);
    let meetsMin = semverCompare(firmwareVersion, approved.minVersion) >= 0;
    let meetsMax = match approved.maxVersion {
        Some(max) => semverCompare(firmwareVersion, max) <= 0,
        None => true,
    };
    meetsMin && meetsMax
}
```

### Attestation Verification

```typescript
fn requireHardwareAttestation(account: address, action: Action, attestation: HardwareAttestation) -> Result {
    let policy = getAttestationPolicy(account);

    let requiresAttestation = match action {
        Action::Transfer(amount) => amount >= policy.requireAttestationForTransfersAbove,
        Action::Upgrade => policy.requireAttestationForUpgrades,
        Action::AddSigner => policy.requireAttestationForNewSigners,
        Action::ChangeGuardian => policy.requireAttestationForGuardianChanges,
        _ => false,
    };

    if !requiresAttestation {
        return Ok;
    }

    let device = getDeviceRegistration(account, attestation.deviceId);
    if device.is_none() || !device.unwrap().active {
        return Err::DeviceNotRegistered;
    }

    if !isFirmwareApproved(attestation.manufacturer, device.unwrap().model, attestation.firmwareVersion) {
        return Err::FirmwareNotApproved;
    }

    if now() - attestation.timestamp > policy.maxAttestationAge {
        return Err::AttestationExpired;
    }

    verifyAttestationSignature(attestation, device.unwrap().publicKey)?;
    Ok
}
```

## Rationale

Hardware attestation provides a cryptographic guarantee that the private key never existed in software-accessible memory. By requiring it for high-value actions, even a fully compromised computer cannot authorize a large transfer without physical access to the hardware device.

## Security Considerations

- Attestation certificates must be validated against manufacturer root CAs
- Firmware version checks prevent attacks using devices with known vulnerabilities
- Fresh attestation requirements (challenge-response) prevent replay attacks
- Browser wallet rejection prevents software wallets from impersonating hardware signers
- Device limits prevent an attacker from registering many devices to dilute security
