# NRC-9: On-Chain Runtime Upgrades

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines forkless runtime upgrades for NOVA. The node host is stable; runtime logic is upgradeable on-chain through a governed process with mandatory challenge windows.

## Motivation

Hard forks are socially expensive and technically risky. Runtime upgrades allow the protocol to evolve without requiring every node to manually update software.

However, upgrades must be conservative. Not everything should be upgradeable.

## Specification

### Core Principle

> Forkless upgrades, filtered through Bitcoin-grade paranoia.

### Upgrade Package

```typescript
struct RuntimeUpgradePackage {
    runtimeVersion: u32;
    targetActivationEpoch: u64;

    wasmHash: bytes32;
    sourceRoot: bytes32;
    compilerId: bytes32;
    compilerVersion: string;
    buildConfigHash: bytes32;

    migrationHash: bytes32;
    specHash: bytes32;
    auditRoot: bytes32;

    safetyProfile: RuntimeSafetyProfile;
}
```

### Upgrade Classes

| Class | Type | Example | Minimum Delay |
|---|---|---|---|
| A | Parameter upgrade | Fee coefficients, oracle thresholds | 24h-72h |
| B | Runtime module upgrade | Account logic, name service | 7 days |
| C | VM/precompile upgrade | New precompile, host function | 14-30 days |
| D | Consensus/security upgrade | PoW, DAG, supply, state root | 30+ days / social consensus |

### Upgrade Flow

```text
1.  Proposal submitted
2.  Runtime package published
3.  Source and build metadata attached (NRC-8)
4.  Deterministic build reproduced by independent verifiers
5.  Simulation against shadow state
6.  Public challenge period (class-dependent duration)
7.  Governance/finality approval
8.  Activation scheduled at epoch N
9.  Nodes automatically switch runtime at epoch N
10. Post-upgrade monitoring window
```

### Safe Upgrade Candidates

- Fee rules
- Gas schedule
- Precompile pricing
- Oracle registry parameters
- Name service rules
- Account standard versions
- Pulsar stdlib system modules
- Runtime host function table
- Aurora committee parameters
- Storage rent parameters
- Transaction type registry
- Signature scheme registry

### Dangerous Upgrade Candidates

These require stricter constitutional hard fork or social consensus:

- PoW validity rules
- BlockDAG ordering rules
- Supply schedule
- Emission cap
- Historical state transition rules
- Address format
- Core hashing commitments

### Safety Profile

```typescript
struct RuntimeSafetyProfile {
    upgradeClass: UpgradeClass;
    affectsConsensus: bool;
    affectsSupply: bool;
    affectsStateRoot: bool;
    requiresMigration: bool;
    challengePeriodDays: u32;
    activationDelayEpochs: u32;
}
```

## Rationale

Class-based upgrade delays ensure that parameter tweaks can happen quickly while consensus-level changes require extended public scrutiny.

## Security Considerations

- All upgrades must have a challenge period
- Simulation must cover shadow state testing
- Post-upgrade monitoring window allows emergency rollback
- Class D upgrades should require social consensus, not just governance vote
