# NRC-36: Token Traits Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines behavioral traits that every token must declare at deployment. Traits are labeled by enforcement layer and enforced at the VM level where appropriate. Protocol trait policies allow applications to reject tokens whose traits are incompatible with their requirements.

## Motivation

NOVA v0.5 section 39 requires every token to declare behavioral traits. Today, users and protocols discover token behavior by reading contract code or after unexpected behavior occurs. A token that rebases, charges fees on transfer, or calls external contracts during transfer creates risks that should be visible and enforceable at the protocol level.

Behavior locks allow permanent renouncement of specific authorities. Protocol trait policies let applications set minimum requirements and reject incompatible tokens at the protocol boundary rather than failing at runtime.

## Specification

### Trait Enforcement Label

```typescript
enum TraitEnforcement {
    ConsensusEnforced,
    RuntimeDetectable,
    VerificationDependent,
    MetadataOnly,
}
```

### TokenTraits

```typescript
struct TokenTraits {
    tokenAddress: address;
    rebasing: bool;
    feeOnTransfer: bool;
    externalCallback: bool;
    mintable: bool;
    pausable: bool;
    blacklistable: bool;
    oracleDependent: bool;
    fixedSupply: bool;
    paymaster: bool;
    traitEnforcement: map<string, TraitEnforcement>;
    behaviorLocks: vec<BehaviorLock>;
    declaredAt: u64;
}
```

### BehaviorLock

```typescript
struct BehaviorLock {
    trait: string;
    lockedValue: bool;
    lockedAt: u64;
    lockHash: bytes32;
}
```

### VM Enforcement Examples

When `externalCallback` is declared `ConsensusEnforced` and set to `false`:

```typescript
fn executeTransfer(ctx: TransferContext) -> Result {
    let traits = getTokenTraits(ctx.token);
    if !traits.externalCallback {
        if hasExternalCallDuringTransfer(ctx) {
            return Err("VM rejected: external callback during transfer on token with externalCallback=false");
        }
    }
    return applyTransfer(ctx);
}
```

When `mintable` is locked to `false` via a behavior lock:

```typescript
fn executeMint(token: address, to: address, amount: u256) -> Result {
    let traits = getTokenTraits(token);
    let lock = traits.behaviorLocks.find(|l| l.trait == "mintable");
    if lock.isSome() && !lock.unwrap().lockedValue {
        return Err("VM rejected: minting disabled by behavior lock");
    }
    return applyMint(token, to, amount);
}
```

### ProtocolTraitPolicy

```typescript
struct ProtocolTraitPolicy {
    protocolAddress: address;
    acceptsRebasing: bool;
    acceptsFeeOnTransfer: bool;
    acceptsCallback: bool;
    acceptsPausable: bool;
    acceptsBlacklistable: bool;
    requiresFixedSupply: bool;
    minimumAuthorityDelay: u64;
}
```

### Trait Compatibility Check

```typescript
fn checkTraitCompatibility(token: address, policy: ProtocolTraitPolicy) -> Option<string> {
    let traits = getTokenTraits(token);
    if traits.rebasing && !policy.acceptsRebasing {
        return some("Token is rebasing; protocol does not accept rebasing tokens");
    }
    if traits.feeOnTransfer && !policy.acceptsFeeOnTransfer {
        return some("Token has fee-on-transfer; protocol does not accept fee-on-transfer tokens");
    }
    if traits.externalCallback && !policy.acceptsCallback {
        return some("Token has external callbacks; protocol does not accept callback tokens");
    }
    if traits.pausable && !policy.acceptsPausable {
        return some("Token is pausable; protocol does not accept pausable tokens");
    }
    if traits.blacklistable && !policy.acceptsBlacklistable {
        return some("Token is blacklistable; protocol does not accept blacklistable tokens");
    }
    if policy.requiresFixedSupply && !traits.fixedSupply {
        return some("Protocol requires fixed supply; token does not have fixed supply");
    }
    return none;
}
```

### Wallet Display

```text
Token: NOVA
Traits:
  Rebasing:          No
  Fee on transfer:   No
  External callback: No
  Mintable:          Yes (timelock: 48h)
  Pausable:          Yes (timelock: 24h)
  Blacklistable:     Yes (timelock: 24h)
  Oracle dependent:  No
  Fixed supply:      No
  Paymaster:         No

  Behavior locks: Mintable locked to false (permanent)
```

### Trait Declaration at Deployment

```typescript
struct TokenDeployment {
    name: string;
    symbol: string;
    decimals: u8;
    initialSupply: u256;
    traits: TokenTraits;
    adminRoles: vec<AdminRole>;
}
```

## Rationale

Enforcement labels acknowledge that not all traits can or should be enforced at the consensus level. Consensus enforcement is strongest but most expensive. Metadata-only is weakest but still valuable for risk tooling. The label tells wallets and tools how much to trust each trait declaration.

Behavior locks are permanent: once a trait is locked, it cannot be unlocked. This supports permanent authority renouncement without requiring trust in the deployer's future behavior.

## Security Considerations

- Consensus-enforced traits that are incorrectly labeled as metadata-only create a trust gap. Tooling should verify enforcement labels against actual contract behavior.
- Behavior locks are irreversible. Once mintable is locked to false, the supply is permanently fixed. This is by design but must be clearly communicated.
- Protocol trait policies that are too permissive accept risky tokens. Policies that are too restrictive may reject legitimate tokens. Defaults should err on the side of caution.
- Trait declarations at deployment must match actual contract behavior. Mismatches should be flagged by verification tooling.
