# NRC-45: Compiler Soundness & Supply Chain Security

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines mandatory security requirements for the Pulsar compiler pipeline, the NovaVM optimizer, and the Rust dependency supply chain. Addresses the critical trust boundary where language-level safety guarantees must survive compilation to WASM bytecode.

## Motivation

Historical compiler bugs prove that language-level safety guarantees are illusions if the compiler does not preserve them at the execution level.

Precedents:

* **Vyper `@nonreentrant` bug (2023)**: Compiler failed to generate correct reentrancy locks, resulting in multi-million dollar exploits on Curve Finance.
* **Solidity Yul optimizer bugs**: Optimizer incorrectly removed storage writes and memory safety checks, silently weakening contract security.
* **Circom under-constrained circuits**: Compiler omitted mathematical constraints, allowing invalid proofs to pass verification.

NOVA must not repeat these patterns. Every safety guarantee that exists at the Pulsar source level must be verified at the NovaVM bytecode level.

## Specification

### Compiler Verification Pipeline

```typescript
struct CompilerVerification {
    compilerVersion: string;
    sourceHash: bytes32;
    bytecodeHash: bytes32;
    referenceInterpreterResult: option<ExecutionTrace>;
    bytecodeVerificationResult: BytecodeVerificationResult;
    differentialFuzzResult: option<FuzzComparison>;
    optimizerPassesEnabled: vec<string>;
    timestamp: u64;
}

enum BytecodeVerificationResult {
    Passed,
    ExternalBarrierMissing,
    CheckedArithmeticMissing,
    InvariantCheckMissing,
    UnknownViolation,
}
```

### Reference Interpreter

A verified reference interpreter must exist for the Pulsar language.

* The reference interpreter is the ground truth for Pulsar semantics.
* Every compiler release is tested against the reference interpreter using differential fuzzing.
* If the reference interpreter and compiled WASM disagree on any execution trace, the contract is rejected at deployment.
* The reference interpreter itself is formally verified for correctness against the Pulsar specification.

### Bytecode Verification

Before deployment, NovaVM validates compiled bytecode against structural safety rules:

```typescript
struct BytecodeSafetyChecks {
    noStateWriteAfterExternalCall: bool;
    allArithmeticIsChecked: bool;
    invariantChecksPresent: bool;
    noDirectMemoryAccess: bool;
    noUnsafeControlFlow: bool;
    externalBlocksProperlyDelimited: bool;
}
```

If any check fails, the contract is rejected regardless of whether the source code looks correct.

### Optimizer Safety Rules

```typescript
struct OptimizerRule {
    ruleId: string;
    description: string;
    safetyRelevant: bool;
    formallyProvenSound: bool;
    enabled: bool;
}
```

Mandatory rules:

* Checked arithmetic operations cannot be replaced with unchecked equivalents.
* Instructions cannot be reordered across `external { }` call boundaries.
* `@invariant` checks cannot be eliminated.
* No optimization pass may change the observable behavior of any safety-critical operation.
* Every optimization pass must be tested by comparing optimized and unoptimized output against the reference interpreter.

### Dependency Supply Chain Security

```typescript
struct DependencyAudit {
    crateName: string;
    version: string;
    auditDate: u64;
    auditor: string;
    result: AuditResult;
    cveExceptions: vec<string>;
}

enum AuditResult {
    Clean,
    AcceptedRisk,
    Rejected,
}
```

Requirements:

* All production builds use strict `Cargo.lock` pinning.
* Every dependency must have a recorded audit before inclusion in a release build.
* Full dependency tree must build deterministically across independent machines.
* Automated vulnerability scanning runs continuously against all dependencies.
* Dependencies with known critical vulnerabilities are patched within 24 hours or the crate is disabled.
* Security-critical paths (cryptography, consensus, VM) minimize external dependencies and prefer in-house implementations.

### WASM Runtime Requirements

```typescript
struct RuntimeVerification {
    runtimeId: string;
    runtimeVersion: string;
    noJitCompilation: bool;
    interpreterFallbackAvailable: bool;
    continuousFuzzingEnabled: bool;
    addressSanitizerEnabled: bool;
    memorySanitizerEnabled: bool;
    multiRuntimeAgreement: bool;
}
```

* No JIT compilation in production. JIT introduces speculative execution and code generation attack surfaces.
* Interpreted-only fallback must be available if the AOT path shows anomalies.
* At least two independent WASM runtime implementations must agree on every execution result.

## Rationale

The compiler is the most dangerous component in a "safe by construction" system. If developers trust the language but the compiler breaks the guarantees, the safety model collapses silently. The Vyper and Solidity precedents demonstrate that this is not theoretical. Supply chain attacks on dependencies are a proven attack vector in the broader software ecosystem.

## Security Considerations

* Reference interpreter bugs could themselves produce false negatives. The interpreter is a small, auditable codebase with formal verification targets.
* Bytecode verification adds gas cost to deployment. This is acceptable because deployment is infrequent.
* Optimizer restrictions may increase gas costs for contract execution. This is a deliberate trade-off: safety over performance.
* Dependency auditing is labor-intensive. Automated scanning covers known CVEs but cannot catch novel backdoors. Critical dependencies require human audit.
* Multi-runtime verification doubles execution cost for consensus-critical paths. This is acceptable because consensus correctness is non-negotiable.
