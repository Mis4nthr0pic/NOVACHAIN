# NRC-12: Oracle and Data Attestation Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines native oracle feeds and custom HTTP attestation standards for NOVA smart contracts.

## Motivation

Smart contracts need external data (prices, weather, sports results, etc.) but cannot directly access the internet during execution. This would break deterministic consensus.

## Specification

### Core Rule

> NOVA contracts cannot call the internet. NOVA contracts can consume signed, finalized attestations about internet data.

### Architecture

```text
contract requests data
oracle nodes fetch off-chain
quorum signs result
result is posted on-chain
contract reads deterministic attestation
```

### Oracle Price Object

```typescript
struct OraclePrice {
    value: u256;
    decimals: u8;
    updatedAt: u64;
    confidence: u256;
    sources: u16;
    twapWindow: u64;
    liquidityDepth: u256;
    deviationBps: u32;
}
```

### Oracle Interface

```typescript
interface Oracle {
    fn price(pair: string) -> OraclePrice;
    fn attestation(feedId: bytes32) -> DataAttestation;
    fn isFresh(feedId: bytes32, maxAge: u64) -> bool;
}
```

### Data Attestation

```typescript
struct DataAttestation {
    feedId: bytes32;
    data: bytes;
    signers: vec<address>;
    threshold: u16;
    timestamp: u64;
    confidence: u256;
}
```

### Usage Pattern

```typescript
let btc = oracle.price("BTC/USD");

if block.timestamp - btc.updatedAt > 60 {
    revert StalePrice(btc.updatedAt);
}

if btc.confidence < 95 {
    revert LowConfidence(btc.confidence);
}
```

### Freshness Checks

Contracts must always verify:

- Data is recent (`updatedAt` within acceptable window)
- Confidence meets threshold
- Sufficient oracle sources agree

### HTTP Attestation Model

For custom data not covered by native feeds:

```text
1. User or contract specifies HTTP endpoint and expected schema
2. Oracle nodes fetch the endpoint off-chain
3. Oracle nodes sign the response
4. Signed attestation is submitted on-chain
5. Contract validates quorum and freshness
```

## Rationale

Native oracle support at the protocol level reduces dependency on external oracle networks and makes price feeds more reliable and cheaper to consume.

## Security Considerations

- Oracle manipulation: always check confidence and freshness
- Stale data: always verify `updatedAt`
- Oracle collusion: quorum thresholds and multiple independent sources
- Contract should never assume oracle data is infallible
