# NRC-11: Event Logs and Bloom Filters

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines event emission, logs, receipts, and filtering for NOVA smart contracts.

## Motivation

Smart contracts need a way to communicate state changes to off-chain observers. Events must be efficiently searchable and cryptographically provable.

## Specification

### Architecture

```text
Event logs in receipts
→ Merkle logsRoot for proofs
→ logsBloom for fast discovery
→ typed Pulsar ABI for decoding
```

### Block Header

```typescript
struct BlockHeader {
    logsRoot: bytes32;
    logsBloom: bytes;
}
```

### Event Log

```typescript
struct EventLog {
    emitter: address;
    eventName: bytes32;
    topics: bytes32[];
    data: bytes;
}
```

### Pulsar Event Syntax

```typescript
event Transfer(
    indexed from: address,
    indexed to: address,
    amount: u256
);
```

### Bloom Filter Inserts

- Emitter address
- Event name hash
- Indexed topics

### Bloom Filter Properties

- Can have false positives
- Cannot prove an event occurred
- Used for fast discovery only

### Merkle Proof

- `logsRoot` provides cryptographic proof
- Light clients can verify event inclusion
- No trust in indexers required

### Event Emission

```typescript
emit Transfer(msg.sender, to, amount);
```

### Filtering

Clients can filter events by:

- Contract address
- Event name
- Indexed topic values
- Block range

### Decoding

Events are decoded using the typed Pulsar ABI:

```typescript
struct EventAbi {
    name: string;
    params: vec<ParamAbi>;
}

struct ParamAbi {
    name: string;
    type: AbiType;
    indexed: bool;
}
```

## Rationale

Three-layer design (Bloom for speed, Merkle for proof, ABI for decoding) gives developers both efficient indexing and cryptographic verification.

## Security Considerations

- Bloom filters are probabilistic; always verify with Merkle proofs
- Event data is not consensus-critical but logsRoot commitments are
- Light clients should verify event inclusion before trusting indexed data
