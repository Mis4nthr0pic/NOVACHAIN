# NRC-39: Pulsar Formal Specification Language

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines the specification language embedded in Pulsar for formal verification. Spec annotations attach behavioral contracts to functions and state: @invariant, @spec, @requires, @ensures, @modifies. Specs are optional for standard contracts, mandatory for high-assurance contracts. Integrates with the Pulsar Prover (NRC-40) for automated verification condition generation.

## Motivation

NOVA v0.5 section 42 introduces formal verification as a first-class toolchain concern. Today, contract behavior is tested but not specified. Tests sample behavior; specs define it. Without a specification language, formal verification has no input. Developers who want high-assurance contracts have no standardized way to express what their code should do.

Pulsar's spec language bridges the gap between informal documentation and machine-checkable proofs. Specs live alongside code, version with code, and can be verified automatically. They are optional for standard contracts but mandatory for contracts that manage user funds, governance, bridges, or custody.

## Specification

### SpecAnnotation

```typescript
enum SpecAnnotationKind {
    Invariant,
    Spec,
    Requires,
    Ensures,
    Modifies,
}

struct SpecAnnotation {
    kind: SpecAnnotationKind;
    expression: string;
    sourceLocation: SourceLocation;
}
```

### SourceLocation

```typescript
struct SourceLocation {
    file: string;
    line: u32;
    column: u32;
}
```

### Annotated Function

```typescript
struct AnnotatedFunction {
    name: string;
    params: vec<Parameter>;
    returnType: string;
    annotations: vec<SpecAnnotation>;
}

struct Parameter {
    name: string;
    type: string;
}
```

### Contract Specifications

```typescript
struct ContractSpec {
    contractAddress: address;
    invariants: vec<SpecAnnotation>;
    functionSpecs: map<string, vec<SpecAnnotation>>;
    stateVariables: vec<StateVariableSpec>;
    declaredAt: u64;
}

struct StateVariableSpec {
    name: string;
    type: string;
    modifies: vec<string>;
}
```

### @invariant Annotation

Invariants are properties that must hold across every state transition of the contract.

```typescript
let balanceInvariant: SpecAnnotation = {
    kind: SpecAnnotationKind::Invariant,
    expression: "forall u: User :: u.balance >= 0 && totalSupply == sum(u.balance)",
    sourceLocation: { file: "token.pulsar", line: 12, column: 1 },
};
```

### @requires Annotation

Preconditions that must hold when a function is called.

```typescript
let transferRequires: SpecAnnotation = {
    kind: SpecAnnotationKind::Requires,
    expression: "sender.balance >= amount && amount > 0",
    sourceLocation: { file: "token.pulsar", line: 45, column: 5 },
};
```

### @ensures Annotation

Postconditions that must hold after a function returns.

```typescript
let transferEnsures: SpecAnnotation = {
    kind: SpecAnnotationKind::Ensures,
    expression: "sender.balance == old(sender.balance) - amount && receiver.balance == old(receiver.balance) + amount",
    sourceLocation: { file: "token.pulsar", line: 46, column: 5 },
};
```

### @modifies Annotation

Declares which state variables a function may modify.

```typescript
let transferModifies: SpecAnnotation = {
    kind: SpecAnnotationKind::Modifies,
    expression: "balances, nonce",
    sourceLocation: { file: "token.pulsar", line: 47, column: 5 },
};
```

### @spec Annotation

Top-level specification block attaching a named spec to a contract or function.

```typescript
let tokenSpec: SpecAnnotation = {
    kind: SpecAnnotationKind::Spec,
    expression: "TokenBehavior { transfer_preserves_supply, approve_no_balance_change }",
    sourceLocation: { file: "token.pulsar", line: 8, column: 1 },
};
```

### Full Contract Example

```typescript
let tokenContractSpec: ContractSpec = {
    contractAddress: "nova1token...",
    invariants: [
        {
            kind: SpecAnnotationKind::Invariant,
            expression: "totalSupply >= 0",
            sourceLocation: { file: "token.pulsar", line: 10, column: 1 },
        },
        {
            kind: SpecAnnotationKind::Invariant,
            expression: "forall u: User :: u.balance >= 0",
            sourceLocation: { file: "token.pulsar", line: 11, column: 1 },
        },
    ],
    functionSpecs: {
        "transfer": [
            {
                kind: SpecAnnotationKind::Requires,
                expression: "sender.balance >= amount",
                sourceLocation: { file: "token.pulsar", line: 25, column: 5 },
            },
            {
                kind: SpecAnnotationKind::Ensures,
                expression: "sender.balance == old(sender.balance) - amount",
                sourceLocation: { file: "token.pulsar", line: 26, column: 5 },
            },
            {
                kind: SpecAnnotationKind::Modifies,
                expression: "balances",
                sourceLocation: { file: "token.pulsar", line: 27, column: 5 },
            },
        ],
        "mint": [
            {
                kind: SpecAnnotationKind::Requires,
                expression: "caller == minter && amount > 0",
                sourceLocation: { file: "token.pulsar", line: 40, column: 5 },
            },
            {
                kind: SpecAnnotationKind::Ensures,
                expression: "totalSupply == old(totalSupply) + amount",
                sourceLocation: { file: "token.pulsar", line: 41, column: 5 },
            },
        ],
    },
    stateVariables: [
        { name: "balances", type: "map<Address, u256>", modifies: ["transfer", "mint"] },
        { name: "totalSupply", type: "u256", modifies: ["mint", "burn"] },
    ],
    declaredAt: 1700000000,
};
```

### Prover Integration

```typescript
struct VerificationCondition {
    id: string;
    spec: SpecAnnotation;
    bytecodeRange: BytecodeRange;
    conditionType: string;
}

struct BytecodeRange {
    startOffset: u32;
    endOffset: u32;
}

fn generateVerificationConditions(contractSpec: ContractSpec) -> vec<VerificationCondition> {
    let conditions: vec<VerificationCondition> = [];
    for invariant in contractSpec.invariants {
        conditions.push({
            id: generateId(),
            spec: invariant,
            bytecodeRange: { startOffset: 0, endOffset: 0 },
            conditionType: "invariant",
        });
    }
    for entry in contractSpec.functionSpecs.entries() {
        for annotation in entry.value {
            conditions.push({
                id: generateId(),
                spec: annotation,
                bytecodeRange: { startOffset: 0, endOffset: 0 },
                conditionType: "function_spec",
            });
        }
    }
    return conditions;
}
```

### High-Assurance Requirement

```text
Specification requirements by verification level:

Standard:         Specs optional. Basic type checking and lint.
FormallyVerified:  @requires and @ensures on all public functions.
                  @invariant on all contract state.
HighAssurance:    Full spec coverage including @modifies.
                  All specs proven by Pulsar Prover (NRC-40).
                  Independent audit of spec completeness.
```

## Rationale

Spec annotations live in source alongside implementation code. This keeps specs in sync with code and makes them part of the developer workflow. The five annotation kinds cover the core specification surface: invariants (global properties), requires (preconditions), ensures (postconditions), modifies (frame conditions), and spec (named behavioral blocks).

Making specs optional for standard contracts avoids overhead for simple deployments. Making specs mandatory for high-assurance contracts ensures that the most critical code has machine-checkable behavioral definitions. The integration with NRC-40 provides the verification backend.

## Security Considerations

- Specs that are incomplete or too weak provide false confidence. A verified spec only guarantees what it specifies, not what it omits.
- @modifies annotations that understate the frame allow the prover to miss side effects. Frame conditions must be reviewed carefully.
- Specs must be versioned with code. A spec that diverges from its implementation is worse than no spec at all.
- The specification language expression syntax must be unambiguous. Parsing ambiguity can lead to specs meaning something different from what the developer intended.
- Specs do not replace testing or auditing. They formalize properties that should also be tested and reviewed independently.
