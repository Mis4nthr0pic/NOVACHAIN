# NRC-32: Contract Safety Metadata

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines a standard for contracts to publish structured safety metadata covering verification status, testing, auditing, upgradeability, and known risks. Wallets and explorers render this metadata as a clear safety summary before every interaction.

## Motivation

NOVA v0.5 section 38 states that contracts should publish safety metadata. Today, users interact with contracts knowing nothing about their safety posture. Audit status, test coverage, formal verification, and upgrade mechanisms are all invisible. This NRC makes safety metadata machine-readable and user-visible.

A contract that has not been tested, audited, or formally verified should not present the same trust surface as one that has. Safety metadata does not guarantee safety, but its absence is itself a signal.

## Specification

### AuditReport

```typescript
struct AuditReport {
    auditor: string;
    reportHash: bytes32;
    reportUrl: string;
    completedAt: u64;
    severityFindings: vec<string>;
}
```

### BugBountyInfo

```typescript
struct BugBountyInfo {
    active: bool;
    platform: string;
    url: string;
    maxReward: u256;
    scope: string;
}
```

### ContractSafetyMetadata

```typescript
struct ContractSafetyMetadata {
    contractAddress: address;
    verifiedSource: bool;
    reproducibleBuild: bool;
    formalInvariants: bool;
    fuzzTests: bool;
    auditReports: vec<AuditReport>;
    bugBounty: BugBountyInfo;
    upgradeable: bool;
    upgradeDelay: u64;
    adminRoles: vec<string>;
    oracleDependencies: vec<string>;
    bridgeDependencies: vec<string>;
    knownRisks: vec<string>;
    publishedAt: u64;
    updatedAt: u64;
}
```

### Safety Metadata Interface

```typescript
trait SafetyMetadataStore {
    fn publish(metadata: ContractSafetyMetadata);
    fn getMetadata(contractAddress: address) -> option<ContractSafetyMetadata>;
    fn updateMetadata(metadata: ContractSafetyMetadata);
}
```

### Display Format

Wallets and explorers must render safety metadata as a structured summary:

```text
Contract Safety: nova1abc...

  Source verified:    Yes
  Reproducible build: Yes
  Fuzz testing:       Yes
  Formal invariants:  No
  Audited:            Yes (SigmaPrime, 2025-03-14)
  Bug bounty:         Yes (up to 50,000 NOVA)
  Upgradeable:        Yes (48h delay)
  Oracle deps:        Pyth, Chainlink
  Bridge deps:        None
  Known risks:        2 listed

  Tests | Fuzz | Formal | Audit | Bounty | Risks
   Yes  | Yes  |   No   |  Yes  |  Yes   |  2
```

### Compact Display

For transaction confirmation screens where space is limited:

```text
Safety: Verified | Fuzzed | Audited | Bounty | Upgradable (48h)
Known risks: 2
```

### Missing Metadata

When no safety metadata is published:

```text
Contract Safety: UNKNOWN

  This contract has not published safety metadata.
  Interact with caution.
```

## Rationale

Each field answers a specific question a user or tool might have before interacting. Verified source and reproducible builds establish code integrity. Formal invariants and fuzz tests signal testing depth. Audit reports provide third-party assessment. Bug bounty programs create ongoing incentive for vulnerability discovery. Upgradeability and delay parameters affect ongoing trust. Dependencies on oracles and bridges surface external risk vectors. Known risks let developers be honest about limitations.

## Security Considerations

- Safety metadata is self-reported. Verified source and reproducible build fields should be validated by infrastructure, not taken on trust.
- Audit reports should be verifiable via hash against the auditor's published report.
- Missing metadata must be displayed as a warning, not silently omitted.
- Known risks that are empty is different from safety metadata that is absent. An empty known-risks list means the developer reviewed and found none. Absent metadata means no review occurred.
