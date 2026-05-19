# NRC-7: Typed Intent Signing

**Status:** Draft  
**Category:** Account Security  

## Abstract

Defines anti-blind-signing transaction standards for NOVA. Wallets sign typed intents, not opaque byte arrays.

## Motivation

Blind signing is one of the most exploited attack vectors in crypto. Users sign opaque hex data without understanding what the transaction does. NOVA requires that every transaction be represented as a typed, displayable intent.

## Specification

### Core Rule

> If a wallet cannot explain the transaction, the account should not sign it.

### Intent Structure

```typescript
struct TransactionIntent {
    intentType: IntentType;
    from: address;
    chainId: u64;
    nonce: u64;
    validUntil: u64;
    feeLimit: FeeLimit;
    calls: vec<TypedCall>;
}
```

### Typed Call

```typescript
struct TypedCall {
    target: address;
    selector: bytes4;
    args: TypedArgs;
    value: u256;
}
```

### Intent Types

```typescript
enum IntentType {
    NativeTransfer,
    ContractCall,
    ContractDeploy,
    AccountUpdate,
    Batch,
}
```

### Transfer Intent Example

```typescript
struct TransferIntent {
    from: address;
    to: address;
    asset: AssetId;
    amount: u256;
    feeLimit: FeeLimit;
    validUntil: u64;
    nonce: u64;
}
```

### Wallet Display

```text
Send:
100 NOVA

To:
alex.nova
funny-horse-idaho-king
nova1z4m8...c3s5a

Expires:
2 minutes

Max fee:
0.01 NOVA
```

### Native Simulation

Before signing, wallets receive deterministic simulation results:

```typescript
struct SimulationResult {
    stateDiff: StateDiff[];
    assetDiff: AssetDiff[];
    newApprovals: ApprovalDiff[];
    externalCalls: ExternalCall[];
    upgradeEffects: UpgradeEffect[];
    riskFlags: RiskFlag[];
}
```

### Risk Flags

```typescript
enum RiskFlag {
    UnknownContract,
    NewApproval,
    UnlimitedApproval,
    UnsafeUpgrade,
    HighValueTransfer,
    UnverifiedContract,
    NoTimelock,
}
```

## Rationale

Typed intents give wallets the information they need to display human-readable transaction details. Simulation gives users a preview of state changes before signing.

## Security Considerations

- Smart accounts may reject transactions with risk flags
- Accounts can enforce policies: no unverified contracts, no unlimited approvals, mandatory timelocks
- `validUntil` prevents transaction replay after expiry
