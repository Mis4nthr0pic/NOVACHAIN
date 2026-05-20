# NRC-38: Enforcement Layer Classification

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines NOVA's enforcement layer taxonomy. Safety rules are classified into five layers: consensus-enforced, account-enforced, interface-enforced, socially-enforced, and not enforceable. Safety is stronger closer to consensus, but not every risk belongs in consensus.

## Motivation

NOVA v0.5 section 5 separates safety into enforcement layers. Not all safety properties can or should be enforced by the consensus protocol. Some belong at the account level (spending limits, multisig). Some belong at the interface level (risk labels, warnings). Some require social coordination (governance decisions). Some are not enforceable at all (user behavior off-chain).

Without explicit classification, safety proposals default to "put it in consensus," which is expensive, rigid, and inappropriate for many risks. This NRC provides the framework for deciding where each rule belongs.

## Specification

### EnforcementLayer

```typescript
enum EnforcementLayer {
    ConsensusEnforced,
    AccountEnforced,
    InterfaceEnforced,
    SociallyEnforced,
    NotEnforceable,
}
```

### EnforcementClassification

```typescript
struct EnforcementClassification {
    rule: string;
    layer: EnforcementLayer;
    description: string;
    examples: vec<string>;
    escalationPath: option<EnforcementLayer>;
}
```

### Consensus-Enforced Rules

```typescript
let consensusRules: vec<EnforcementClassification> = [
    {
        rule: "invalid_signatures",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Transactions with invalid signatures are rejected by the protocol",
        examples: ["Ed25519 signature mismatch", "expired session key"],
        escalationPath: none,
    },
    {
        rule: "invalid_state_transitions",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "State transitions that violate protocol rules are rejected",
        examples: ["insufficient balance", "nonce mismatch", "overflow"],
        escalationPath: none,
    },
    {
        rule: "invalid_token_trait_behavior",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Token behavior that contradicts declared traits is rejected by the VM",
        examples: ["external callback during transfer on non-callback token", "mint on supply-locked token"],
        escalationPath: none,
    },
    {
        rule: "invalid_capability_use",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Capability usage without proper authorization or warm-up is rejected",
        examples: ["undeclared capability invocation", "capability before warm-up period"],
        escalationPath: none,
    },
    {
        rule: "invalid_randomness_api",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Randomness API misuse is rejected at the VM level",
        examples: ["predictable randomness", "biasable randomness source"],
        escalationPath: none,
    },
    {
        rule: "invalid_bytecode",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Bytecode that violates VM constraints is rejected",
        examples: ["undecodable instructions", "stack overflow", "gas limit exceeded"],
        escalationPath: none,
    },
    {
        rule: "invalid_supply_schedule",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Token supply changes that violate the declared schedule are rejected",
        examples: ["minting beyond cap", "inflation rate exceeded"],
        escalationPath: none,
    },
    {
        rule: "invalid_finality_vote",
        layer: EnforcementLayer::ConsensusEnforced,
        description: "Finality votes that violate consensus rules are rejected",
        examples: ["double vote", "vote on unknown block", "equivocation"],
        escalationPath: none,
    },
];
```

### Account-Enforced Rules

```typescript
let accountRules: vec<EnforcementClassification> = [
    {
        rule: "spending_limits",
        layer: EnforcementLayer::AccountEnforced,
        description: "Per-transaction or per-period spending caps enforced by account logic",
        examples: ["daily limit of 1000 NOVA", "per-transaction limit of 100 NOVA"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "recipient_cooldowns",
        layer: EnforcementLayer::AccountEnforced,
        description: "Time delays before sending to a new recipient",
        examples: ["24h cooldown before first transfer to new address"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "multisig_thresholds",
        layer: EnforcementLayer::AccountEnforced,
        description: "Multiple signatures required for high-value or sensitive operations",
        examples: ["2-of-3 for transfers above 10,000 NOVA", "3-of-5 for contract upgrades"],
        escalationPath: none,
    },
    {
        rule: "hardware_signer_requirements",
        layer: EnforcementLayer::AccountEnforced,
        description: "Specific hardware signer required for certain operations",
        examples: ["hardware key required for governance votes", "HSM required for treasury operations"],
        escalationPath: none,
    },
    {
        rule: "recovery_delays",
        layer: EnforcementLayer::AccountEnforced,
        description: "Time delays during account recovery to allow intervention",
        examples: ["72h delay before new key becomes active during recovery"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "upgrade_delays",
        layer: EnforcementLayer::AccountEnforced,
        description: "Time delays before contract upgrades take effect",
        examples: ["48h delay between upgrade proposal and execution"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "capability_warm_up",
        layer: EnforcementLayer::AccountEnforced,
        description: "Time delay between capability declaration and first use",
        examples: ["24h warm-up before new admin capability is usable"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "simulation_attestation",
        layer: EnforcementLayer::AccountEnforced,
        description: "Require simulated execution to succeed before submitting transaction",
        examples: ["simulate swap before submitting", "verify gas estimate matches actual"],
        escalationPath: none,
    },
];
```

### Interface-Enforced Rules

```typescript
let interfaceRules: vec<EnforcementClassification> = [
    {
        rule: "risk_labels",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Wallets and explorers must display risk labels for token interactions",
        examples: ["high-risk token warning", "unverified contract warning"],
        escalationPath: some(EnforcementLayer::AccountEnforced),
    },
    {
        rule: "bridge_trust_warnings",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Cross-chain operations must display bridge trust information",
        examples: ["bridge not audited warning", "bridge delay information"],
        escalationPath: none,
    },
    {
        rule: "token_trait_warnings",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Token traits that create user risk must be surfaced",
        examples: ["rebasing token warning", "fee-on-transfer disclosure", "blacklistable token notice"],
        escalationPath: some(EnforcementLayer::ConsensusEnforced),
    },
    {
        rule: "governance_effect_previews",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Governance proposals must display structured effect previews",
        examples: ["asset movements listed", "contract upgrades identified", "role changes enumerated"],
        escalationPath: none,
    },
    {
        rule: "frontend_mismatch_warnings",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Warn when frontend manifest signature does not match expected key",
        examples: ["frontend key rotation alert", "unsigned manifest warning"],
        escalationPath: some(EnforcementLayer::AccountEnforced),
    },
    {
        rule: "contract_safety_display",
        layer: EnforcementLayer::InterfaceEnforced,
        description: "Contract safety metadata must be rendered before interaction",
        examples: ["audit status", "fuzz test results", "known risks"],
        escalationPath: none,
    },
];
```

### Layer Escalation Rule

```text
Safety is stronger closer to consensus, but not every risk belongs in consensus.
Rules escalate upward only when the lower layer fails and the harm justifies the cost.
```

```typescript
fn shouldEscalate(rule: EnforcementClassification) -> bool {
    match rule.escalationPath {
        some(EnforcementLayer::ConsensusEnforced) => true,
        some(EnforcementLayer::AccountEnforced) => true,
        some(EnforcementLayer::InterfaceEnforced) => true,
        some(_) => false,
        none => false,
    }
}
```

### Display Format

```text
Enforcement Layer Summary:

Consensus (8 rules):     Signatures, state transitions, token traits,
                         capabilities, randomness, bytecode, supply, finality
Account (8 rules):       Spending limits, cooldowns, multisig, hardware
                         signers, recovery delays, upgrade delays, warm-up,
                         simulation attestation
Interface (6 rules):     Risk labels, bridge warnings, trait warnings,
                         governance previews, frontend mismatch, safety display
Social:                  Governance decisions, community standards
Not enforceable:         User behavior off-chain, social engineering

Rule: safety is stronger closer to consensus, but not every risk
belongs in consensus.
```

## Rationale

The five-layer taxonomy creates a shared vocabulary for discussing where safety rules belong. Consensus is the strongest layer but also the most expensive and rigid. Account-level enforcement gives individual accounts flexibility while still being on-chain. Interface enforcement shapes user behavior through information. Social enforcement relies on governance and community. Some risks are simply not enforceable by any protocol mechanism.

The escalation path field acknowledges that some rules may need to move to a higher layer if the lower layer proves insufficient in practice.

## Security Considerations

- Rules that should be consensus-enforced but are classified at a lower layer create safety gaps. Classification should be conservative: when in doubt, classify higher.
- Account-enforced rules depend on correct account implementation. Compromised account code can bypass these rules.
- Interface-enforced rules can be bypassed by users who use custom interfaces or CLI tools. They are informational, not protective.
- Socially-enforced rules have no protocol guarantee. They rely on community vigilance and governance participation.
- The escalation path is a recommendation, not a mandate. Escalation should require evidence that the current layer is insufficient.
