# NRC-15: Transaction Hash and Receipt Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines transaction identification, receipt format, and confirmation/finality status for NOVA transactions.

## Motivation

Users and applications need to:

- Uniquely identify transactions
- Track transaction status from submission to finality
- Understand gas consumption across multiple dimensions
- Verify transaction inclusion

## Specification

### Transaction Hash

Bech32m format:

```text
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9
```

Internal computation:

```text
tx_hash = BLAKE3-256("NOVA_TX_V1" || canonical_transaction_bytes)
```

Developer hex format:

```text
0x8f42c1e0a99b7d4d6c2e53f83a5b88dd934fb1e2b92a7126cc6f2dd5a7c9310e
```

### Transaction Status

```typescript
enum TransactionStatus {
    Submitted,         // in mempool
    Preconfirmed,      // included in a block, not yet finalized
    Finalized,         // Aurora checkpoint confirmed
    Failed,            // execution reverted
    Expired,           // validUntil exceeded
}
```

### Receipt Format

```text
NOVA TRANSACTION RECEIPT

Hash:
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9

Type:
Native Transfer

Status:
Finalized

From:
funny-horse-idaho-king
nova1z4m8...c3s5a

To:
silent-orbit-purple-wolf
nova1p7sr...v2q9x

Amount:
25 NOVA

Fee:
0.0031 NOVA

Preconfirmation:
102 ms

Finalized:
Aurora checkpoint 70153

Gas:
compute:      1,220
state_read:   2
state_write:  2
state_growth: 0
bandwidth:    3,912 bytes
```

### Receipt Structure

```typescript
struct TransactionReceipt {
    hash: bytes32;
    txType: TransactionType;
    status: TransactionStatus;
    from: address;
    to: address;
    amount: u256;
    fee: u256;
    gasUsed: MultiGas;
    preconfirmTime: u64;
    finalizedCheckpoint: option<u64>;
    logs: vec<EventLog>;
    blockHash: bytes32;
    blockNumber: u64;
    dagIndex: u64;
}
```

### Multi-Dimensional Gas

```typescript
struct MultiGas {
    compute: u64;
    stateRead: u64;
    stateWrite: u64;
    stateGrowth: u64;
    bandwidth: u64;
}
```

### Transaction Types

```typescript
enum TransactionType {
    NativeTransfer,
    ContractCall,
    ContractDeploy,
    AccountUpdate,
    Batch,
}
```

### Preconfirmation

Preconfirmation is the time from submission to inclusion in an ordered block:

```text
Submitted -> Preconfirmed: typically < 400ms (one block time)
Preconfirmed -> Finalized: depends on Aurora epoch length
```

### Finality

A transaction is finalized when:

1. It is included in an Aurora checkpoint
2. That checkpoint receives 2/3 committee votes

## Rationale

Multi-dimensional gas visibility helps developers understand what their contracts actually cost. Preconfirmation + finalization gives users progressive assurance.

## Security Considerations

- Preconfirmed transactions can theoretically be reorged; only finalized transactions are immutable
- Transaction expiry prevents mempool stagnation
- Receipt logs are covered by `logsRoot` for Merkle proof verification
