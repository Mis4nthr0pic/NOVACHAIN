# NRC-8: On-Chain Contract Verification

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines native contract source verification for NOVA. Verification is decentralized and reproducible, not dependent on a centralized block explorer.

## Motivation

Ethereum relies on Etherscan for source verification. This is:

- Centralized
- Trust-dependent on one service
- Not enforceable by the protocol
- Vulnerable to manipulation

NOVA embeds verification into the protocol layer.

## Specification

### Verification Record

```typescript
struct VerificationRecord {
    contract: address;
    sourceRoot: bytes32;
    abiRoot: bytes32;
    compilerId: bytes32;
    compilerVersion: string;
    stdlibHash: bytes32;
    buildConfigHash: bytes32;
    bytecodeHash: bytes32;
    status: VerificationStatus;
    verifiedAt: option<u64>;
    verifiedBy: option<vec<address>>;
}
```

### Verification Status

```typescript
enum VerificationStatus {
    Unverified,
    SourceCommitted,
    ReproducibleBuildVerified,
    FormallyVerified,
}
```

### Verification Flow

```text
1. Developer writes Pulsar source
2. pulsarc compiles deterministically
3. Developer deploys bytecode with source/build commitments
4. Independent verifiers reproduce build
5. Output bytecode hash must match deployed bytecode hash
6. Verification status is written to NRC-8 registry
```

### Deterministic Build Requirements

- Pinned compiler version
- Pinned stdlib version
- Canonical build configuration
- No environment-dependent paths
- No floating dependencies

### Source Storage

Sources may be stored:

- On-chain compressed (NRC-8S)
- IPFS
- Arweave
- Git repositories
- NOVA source blob layer
- Any content-addressed storage

The chain stores the cryptographic commitment, not necessarily the full source.

### Verification vs Decompilation

```text
Decompiled view     = explanation, not proof
Verified source     = reproducible source-to-bytecode match
Formally verified   = mathematically proven properties
```

### Registry Interface

```typescript
interface VerificationRegistry {
    fn submitVerification(record: VerificationRecord);
    fn confirmVerification(contract: address, verifier: address);
    fn getStatus(contract: address) -> VerificationStatus;
    fn getRecord(contract: address) -> option<VerificationRecord>;
}
```

## Rationale

Native verification means users and wallets can check contract safety without trusting a third-party service.

## Security Considerations

- Compiler determinism is critical; any nondeterminism breaks verification
- Verification is only as strong as the compiler correctness
- Formal verification is optional but encouraged for high-value contracts
