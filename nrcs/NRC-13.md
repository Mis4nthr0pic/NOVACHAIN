# NRC-13: Native Randomness Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines NOVA's native randomness APIs, derived from Aurora finality commitments.

## Motivation

On-chain randomness is critical for lotteries, gaming, NFT minting, and fair distribution. Using `block.timestamp`, `blockhash`, or `msg.sender` for randomness is exploitable.

## Specification

### Core Rule

> The randomness exists only after the user action is committed.

### Architecture

```text
1. User submits transaction
2. Transaction is ordered into the BlockDAG
3. Aurora finalizes checkpoint
4. Committee produces randomness beacon
5. Beacon is post-processed
6. Contract reads randomness through safe API
```

### Pulsar API

```typescript
let roll: u256 = random.uniform(1, 6, salt);
```

This produces a uniform random value between 1 and 6 inclusive, seeded by the user-provided salt and the Aurora randomness beacon.

### Available Functions

```typescript
namespace random {
    fn uniform(min: u256, max: u256, salt: bytes32) -> u256;
    fn bytes(length: u32, salt: bytes32) -> bytes;
    fn bool(salt: bytes32) -> bool;
    fn shuffle<T>(items: vec<T>, salt: bytes32) -> vec<T>;
}
```

### Anti-Pattern: Modulo Bias

Do not use:

```typescript
random.beacon() % 6
```

Modulo can introduce bias when the beacon range is not a multiple of 6.

Always use:

```typescript
random.uniform(1, 6, salt)
```

### Threat Model

| Attacker | Capability | Mitigation |
|---|---|---|
| Normal node | Cannot influence randomness | Beacon is consensus-derived |
| Miner | Can influence inclusion ordering | Randomness is post-commitment |
| Aurora coalition | Could withhold finality | Requires 2/3 collusion; slashable |
| User | Can choose salt | Cannot predict beacon output |

### Commit-Reveal Pattern

For applications needing user-committed randomness:

```text
1. User submits commitment hash = hash(choice + secret)
2. Randomness beacon is produced after commitment is ordered
3. User reveals choice and secret
4. Contract verifies commitment matches
5. Outcome is determined by commitment + beacon
```

## Rationale

Deriving randomness from Aurora finality ensures that the random value is not available until after the transaction is committed to the canonical ordering. This prevents miners and users from predicting outcomes.

## Security Considerations

- Beacon quality depends on Aurora committee size and diversity
- In bootstrap mode (few validators), randomness is weaker
- Applications should combine user salt with beacon output
- High-value applications should use commit-reveal patterns
