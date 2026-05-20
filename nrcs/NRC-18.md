# NRC-18: Independent Simulation Attestation

**Status:** Draft  
**Category:** Account Security  

## Abstract

Requires high-value accounts to obtain simulation results from independent sources before executing transactions. Simulators attest to the expected outcome, state changes, and risk flags.

## Motivation

Even with canonical intent, users may not fully understand the consequences of complex transactions. Simulation provides a preview, but relying on a single simulator creates a trust bottleneck. Independent simulation attestation distributes trust across multiple sources:

- A compromised wallet could simulate incorrect results
- A single RPC node could lie about state changes
- Complex contract interactions may have non-obvious effects

Multiple independent simulators make collusion economically infeasible.

## Specification

### Simulation Attestation

```typescript
struct SimulationAttestation {
    canonicalIntentHash: bytes32;
    simulator: address;
    resultHash: bytes32;
    stateDiff: vec<StateChange>;
    assetDiff: vec<AssetMovement>;
    riskFlags: vec<RiskFlag>;
    timestamp: u64;
    simulatorSignature: bytes;
}
```

### State Change

```typescript
struct StateChange {
    contract: address;
    slot: bytes32;
    oldValue: bytes;
    newValue: bytes;
}
```

### Asset Movement

```typescript
struct AssetMovement {
    asset: address;
    from: address;
    to: address;
    amount: u256;
}
```

### Risk Flag

```typescript
enum RiskFlag {
    UnexpectedAssetMovement,
    PermissionChange,
    CodeModification,
    StorageOverwrite,
    ExternalCallToNewContract,
    RecursiveCall,
    StateExpansionBeyondThreshold,
    FeeExceedsLimit,
    InteractionWithUnauditedContract,
    BridgeToUnrecognizedChain,
}
```

### Attestation Policy

```typescript
struct AttestationPolicy {
    minimumAttestations: u32;
    trustedSimulators: vec<address>;
    requireHardwareAttestation: bool;
}
```

### Validation Rules

```typescript
fn validateAttestations(
    intentHash: bytes32,
    attestations: vec<SimulationAttestation>,
    policy: AttestationPolicy,
) -> bool {
    if attestations.length < policy.minimumAttestations {
        return false;
    }

    let verified: u32 = 0;
    for attestation in attestations {
        if !policy.trustedSimulators.contains(attestation.simulator) {
            continue;
        }
        if attestation.canonicalIntentHash != intentHash {
            continue;
        }
        if !verifySignature(
            attestation.simulator,
            attestation.resultHash,
            attestation.simulatorSignature,
        ) {
            continue;
        }
        if attestation.timestamp < now - ATTESTATION_MAX_AGE {
            continue;
        }
        verified += 1;
    }

    return verified >= policy.minimumAttestations;
}
```

### Trust Model

```text
Trust levels:
1. Self-simulation: account runs its own simulation (no trust)
2. Single trusted simulator: relies on one external source
3. Multi-simulator: requires agreement across N independent sources
4. Hardware-attested: simulator provides TEE-based attestation

High-value accounts must use level 3 or 4.
```

### Result Agreement

```typescript
fn checkResultAgreement(attestations: vec<SimulationAttestation>) -> bool {
    let referenceHash = attestations[0].resultHash;
    for attestation in attestations {
        if attestation.resultHash != referenceHash {
            return false;
        }
    }
    return true;
}
```

All attestations must produce the same `resultHash`. Divergent results indicate either a bug or a compromised simulator.

### Hardware Attestation

When `requireHardwareAttestation` is true, each simulator must provide a TEE attestation quote proving:

- The simulation ran in a trusted execution environment
- The code hash of the simulator binary matches the expected version
- The output was not tampered with

## Rationale

Independent simulation attestation makes it economically infeasible for a single party to mislead a user about transaction outcomes. Requiring agreement on `resultHash` ensures all simulators agree on the outcome.

## Security Considerations

- Simulator identities must be verified through a registry or on-chain attestation
- Attestations must include a timestamp to prevent replay of old simulations
- The `resultHash` must cover all state diffs and asset movements to prevent partial agreement
- Hardware attestation adds a hardware trust root but introduces TEE supply chain considerations
- The trusted simulator list must require multisig governance to update
