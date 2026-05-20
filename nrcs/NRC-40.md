# NRC-40: Pulsar Prover & High-Assurance Certification

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines the Pulsar Prover toolchain for formal verification of Nova contracts. The prover generates verification conditions from Pulsar spec annotations (NRC-39) and NovaVM bytecode, then attempts proof via a Lean4 core with an AI-assisted backend. Defines three verification levels with increasing rigor and certification requirements. High-assurance certification is mandatory for contracts managing significant value, governance, bridges, and custody.

## Motivation

NOVA v0.5 section 42 establishes formal verification as part of the developer toolchain. Specification alone (NRC-39) is not enough. Without a prover, specs are documentation. Without verification levels, every contract gets the same level of scrutiny regardless of risk. Without certification requirements, there is no protocol-level guarantee that high-risk contracts have been verified.

The Pulsar Prover automates what it can and surfaces what it cannot. Lean4 provides a trusted proof core. AI assistance accelerates proof construction. Human review closes the gap. The three-level system matches verification effort to risk.

## Specification

### VerificationLevel

```typescript
enum VerificationLevel {
    Standard,
    FormallyVerified,
    HighAssurance,
}
```

### VerificationLevel Requirements

```text
Standard:
  - Basic type checking and lint
  - Bytecode validation
  - Gas estimation
  - No spec annotations required

FormallyVerified:
  - All Standard requirements
  - @requires and @ensures on all public functions
  - @invariant on all contract state
  - All verification conditions proven by Pulsar Prover
  - Proof artifacts published and hash-committed

HighAssurance:
  - All FormallyVerified requirements
  - Full spec coverage including @modifies
  - Independent audit of spec completeness
  - Reproducible build from source
  - Proof artifacts independently reproducible
  - Prover version and configuration pinned
```

### ProverOutput

```typescript
struct ProverOutput {
    status: ProverStatus;
    provenInvariants: vec<string>;
    failedConditions: vec<FailedCondition>;
    proofArtifactHash: bytes32;
    proverVersion: string;
    timestamp: u64;
    verificationLevel: VerificationLevel;
    contractAddress: address;
    bytecodeHash: bytes32;
}

enum ProverStatus {
    AllProven,
    PartiallyProven,
    Failed,
    Timeout,
    Error,
}

struct FailedCondition {
    conditionId: string;
    spec: string;
    reason: string;
    counterexample: option<string>;
}
```

### Prover Configuration

```typescript
struct ProverConfig {
    proverVersion: string;
    lean4CoreVersion: string;
    aiBackendModel: string;
    timeoutSeconds: u64;
    maxIterations: u32;
    proofStrategy: ProofStrategy;
    verificationLevel: VerificationLevel;
}

enum ProofStrategy {
    Auto,
    Induction,
    SymbolicExecution,
    ModelChecking,
}
```

### Certification Record

```typescript
struct CertificationRecord {
    contractAddress: address;
    verificationLevel: VerificationLevel;
    proverOutput: ProverOutput;
    auditorReports: vec<AuditorReport>;
    buildReproducibility: BuildReproducibility;
    certifiedAt: u64;
    expiresAt: u64;
    certifiedBy: address;
}

struct AuditorReport {
    auditor: address;
    reportHash: bytes32;
    findings: vec<string>;
    recommendation: string;
    reviewedAt: u64;
}

struct BuildReproducibility {
    sourceCommitHash: bytes32;
    bytecodeHash: bytes32;
    buildToolVersion: string;
    reproducibleBy: vec<address>;
    reproducedAt: vec<u64>,
}
```

### High-Assurance Requirements

```typescript
enum HighAssuranceCategory {
    ValueManagement,
    Governance,
    Bridge,
    Custody,
}

struct HighAssuranceTrigger {
    category: HighAssuranceCategory;
    threshold: ThresholdValue;
    description: string;
}

struct ThresholdValue {
    valueType: string;
    value: u256,
    unit: string,
}

let highAssuranceTriggers: vec<HighAssuranceTrigger> = [
    { category: HighAssuranceCategory::ValueManagement, threshold: { valueType: "TVL", value: 1000000, unit: "NOVA" }, description: "Contracts managing TVL above threshold" },
    { category: HighAssuranceCategory::Governance, threshold: { valueType: "Any", value: 0, unit: "NA" }, description: "All governance contracts" },
    { category: HighAssuranceCategory::Bridge, threshold: { valueType: "Any", value: 0, unit: "NA" }, description: "All bridge contracts" },
    { category: HighAssuranceCategory::Custody, threshold: { valueType: "Any", value: 0, unit: "NA" }, description: "All custody contracts" },
];
```

### Prover Pipeline

```typescript
fn runProver(contractSpec: ContractSpec, bytecode: bytes, config: ProverConfig) -> ProverOutput {
    let conditions = generateVerificationConditions(contractSpec);
    let proven: vec<string> = [];
    let failed: vec<FailedCondition> = [];
    for condition in conditions {
        let result = attemptProof(condition, bytecode, config);
        match result {
            ProverResult::Proven => { proven.push(condition.id); },
            ProverResult::Failed(reason, counterexample) => {
                failed.push({
                    conditionId: condition.id,
                    spec: condition.spec.expression,
                    reason: reason,
                    counterexample: counterexample,
                });
            },
        }
    }
    let status = if failed.len() == 0 { ProverStatus::AllProven }
                 else if proven.len() > 0 { ProverStatus::PartiallyProven }
                 else { ProverStatus::Failed };
    let artifactHash = hashProofArtifacts(proven, conditions);
    return {
        status: status,
        provenInvariants: proven,
        failedConditions: failed,
        proofArtifactHash: artifactHash,
        proverVersion: config.proverVersion,
        timestamp: currentTimestamp(),
        verificationLevel: config.verificationLevel,
        contractAddress: contractSpec.contractAddress,
        bytecodeHash: blake3_256(bytecode),
    };
}
```

### Certification Validation

```typescript
fn validateCertification(record: CertificationRecord) -> ValidationResult {
    if record.verificationLevel == VerificationLevel::HighAssurance {
        if record.auditorReports.len() < 1 {
            return ValidationResult::Err("High-assurance requires at least one independent audit");
        }
        if record.buildReproducibility.reproducibleBy.len() < 1 {
            return ValidationResult::Err("High-assurance requires reproducible build");
        }
    }
    if record.proverOutput.status != ProverStatus::AllProven {
        return ValidationResult::Err("All verification conditions must be proven");
    }
    if record.proverOutput.verificationLevel != record.verificationLevel {
        return ValidationResult::Err("Prover output level does not match certification level");
    }
    return ValidationResult::Ok;
}

enum ValidationResult {
    Ok,
    Err(string),
}
```

### Verification Level Display

```text
Contract: nova1lending...

Verification: High Assurance
  Prover: Pulsar v2.1.0 (Lean4 v4.3.1)
  Status: All conditions proven (47/47)
  Proof hash: 0x7f3a...c2d1
  Certified: 2025-06-15
  Expires: 2026-06-15
  Auditors: 2 independent reviews
  Build: Reproducible (confirmed by 3 parties)

Invariants proven:
  total_supply_non_negative
  balance_conservation
  no_unauthorized_mint
  collateral_ratio_maintained
  liquidation_ordering_correct
```

## Rationale

Three verification levels match rigor to risk. Standard is the baseline for all contracts. FormallyVerified adds machine-checked proofs. HighAssurance adds independent human review and reproducible builds. The Lean4 core provides a small trusted computing base for proofs. AI assistance in the backend accelerates proof search without compromising soundness. Failed conditions include counterexamples to help developers fix bugs rather than just reporting failure.

Certification records are on-chain so that wallets, explorers, and risk tools can verify the assurance level of any contract. Expiration ensures that certifications are renewed when contracts are updated.

## Security Considerations

- The prover's soundness depends on the Lean4 core and the verification condition generator. Bugs in either can produce false proofs. The Lean4 core should be independently audited.
- AI-assisted proof construction does not compromise soundness because the Lean4 checker validates all proofs regardless of how they were constructed. But AI may fail to find proofs, leading to false negatives.
- High-assurance certification requires independent audit, but auditor quality varies. Certification should track auditor reputation and history.
- Certification expiration prevents stale certifications from being trusted after contract upgrades. Upgrades should require re-certification.
- The prover version must be pinned and the proof reproducible. A prover upgrade that changes proof output invalidates the certification.
- Failed verification conditions with counterexamples may reveal contract vulnerabilities. Failed conditions should not be published publicly until the vulnerability is addressed.
