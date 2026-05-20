# NRC-27: Bridge Risk Registry

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines a standard for bridge trust model declarations. Every bridge on NOVA must register its trust assumptions so that wallets, explorers, and smart accounts can make informed decisions about cross-chain risk.

## Motivation

Bridges are among the most exploited systems in blockchain. Not all bridges carry equal risk. A light-client bridge has fundamentally different trust assumptions than a custodial bridge, yet wallets display them identically. NOVA v0.5 section 35 requires bridges to declare their trust model explicitly so risk can be surfaced to users.

## Specification

### BridgeType

```typescript
enum BridgeType {
    LightClient,
    Optimistic,
    ZK,
    Multisig,
    MPC,
    Custodial,
}
```

### BridgeRegistration

```typescript
struct BridgeRegistration {
    bridgeId: bytes32;
    name: string;
    bridgeType: BridgeType;
    sourceChain: ChainId;
    destinationChain: ChainId;
    proofModel: ProofModel;
    signerSet: option<SignerSet>,
    upgradeAuthority: AuthorityConfig;
    mintAuthority: AuthorityConfig;
    pauseAuthority: AuthorityConfig;
    dailyCap: u256;
    transactionCap: u256;
    challengePeriod: option<u64>,
    exitPath: ExitPathConfig;
    knownAssumptions: vec<string>;
    registeredAt: u64;
    riskScore: BridgeRisk;
}

struct ProofModel {
    proofType: BridgeType,
    verifierCount: u32,
    threshold: u32,
    challengeMechanism: option<ChallengeMechanism>,
    proofLatency: u64,
}

struct SignerSet {
    signers: vec<address>;
    threshold: u32;
    rotationPolicy: SignerRotationPolicy;
}

struct SignerRotationPolicy {
    timelockDuration: u64;
    requireGovernanceApproval: bool;
    maxRotationFrequency: u64;
}

struct AuthorityConfig {
    authType: AuthorityType,
    addresses: vec<address>,
    threshold: u32,
    timelockDuration: u64;
}

enum AuthorityType {
    SingleSigner,
    MultiSig,
    Governance,
    TimelockedMultiSig,
}

struct ExitPathConfig {
    slowExitAvailable: bool,
    slowExitDelay: u64,
    fastExitAvailable: bool,
    fastExitFeeBps: u32,
    forcedExitAvailable: bool,
    forcedExitDelay: u64,
}

struct ChallengeMechanism {
    bondRequired: u256,
    challengeWindow: u64,
    resolverCount: u32,
}
```

### BridgeRisk

```typescript
enum BridgeRisk {
    Low,
    Medium,
    High,
    Critical,
}

fn computeBridgeRisk(registration: BridgeRegistration) -> BridgeRisk {
    match registration.bridgeType {
        BridgeType::LightClient => BridgeRisk::Low,
        BridgeType::ZK => BridgeRisk::Low,
        BridgeType::Optimistic => BridgeRisk::Medium,
        BridgeType::Multisig => {
            let set = registration.signerSet.unwrap();
            if set.signers.len() <= 3 {
                BridgeRisk::Critical;
            } else if set.threshold < set.signers.len() * 2 / 3 {
                BridgeRisk::High;
            } else {
                BridgeRisk::Medium;
            }
        }
        BridgeType::MPC => BridgeRisk::High,
        BridgeType::Custodial => BridgeRisk::Critical,
    }
}
```

### Wallet Risk Display

```typescript
fn formatBridgeRiskDisplay(registration: BridgeRegistration) -> BridgeRiskDisplay {
    BridgeRiskDisplay {
        name: registration.name,
        bridgeType: registration.bridgeType,
        risk: registration.riskScore,
        trustAssumptions: registration.knownAssumptions,
        dailyCap: registration.dailyCap,
        challengePeriod: registration.challengePeriod,
        exitPath: registration.exitPath,
        signerInfo: match registration.signerSet {
            Some(set) => format(
                "{} of {} signers required",
                set.threshold,
                set.signers.len(),
            ),
            None => "No external signers".to_string(),
        },
    }
}

struct BridgeRiskDisplay {
    name: string;
    bridgeType: BridgeType;
    risk: BridgeRisk;
    trustAssumptions: vec<string>;
    dailyCap: u256;
    challengePeriod: option<u64>,
    exitPath: ExitPathConfig;
    signerInfo: string;
}
```

### Display Requirements

Wallets MUST display bridge risk before any cross-chain transfer:

```text
Bridge Transfer Risk Assessment

Bridge: NovaLight Bridge
Type: Light Client
Risk Level: LOW

Trust assumptions:
- Relies on source chain light client proofs
- No external signer trust required
- 7-day challenge period for disputes

Exit options:
- Slow exit: 7 days, no fee
- Fast exit: 1 hour, 10 bps fee
- Forced exit: 3 days, no fee

Daily cap: 1,000,000 NOVA
```

## Rationale

Different bridge architectures have fundamentally different threat models. A custodial bridge where one entity controls all funds should not appear as safe as a ZK bridge with on-chain verification. The registry makes this distinction explicit and machine-readable.

## Security Considerations

- Bridge risk scores are initial assessments; real-world incident history should be incorporated over time
- Signer set transparency enables community monitoring of bridge operator changes
- Exit path availability determines whether users can recover funds during a bridge failure
- Daily caps limit maximum possible damage from a single bridge compromise
- Custodial and small-multisig bridges should trigger explicit user warnings
