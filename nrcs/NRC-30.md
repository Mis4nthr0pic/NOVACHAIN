# NRC-30: Oracle Safety Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines safety requirements for oracle data consumption on NOVA. Price feeds are among the most dangerous DeFi inputs, and protocols must validate oracle freshness, confidence, and deviation before acting on reported values.

## Motivation

Oracle manipulation has caused billions in DeFi losses. Protocols that use a single price update without checking its age, confidence, or deviation from recent values are vulnerable to stale-price exploitation, flash-loan-driven oracle manipulation, and compromised oracle nodes. NOVA v0.5 section 34 specifies required oracle fields and safety constraints.

## Specification

### OracleData

```typescript
struct OracleData {
    feedId: bytes32;
    price: u256;
    decimals: u8;
    updatedAt: u64;
    confidence: u32;
    sourceCount: u32;
    twapWindow: u64;
    liquidityDepth: u256;
    deviation: u32;
    circuitBreakerStatus: CircuitBreakerStatus;
}

enum CircuitBreakerStatus {
    Active,
    Paused,
    Tripped,
    StaleData,
}
```

### OracleSafetyConfig

```typescript
struct OracleSafetyConfig {
    feedId: bytes32;
    minimumLiquidity: u256;
    maxDeviationPerBlock: u32;
    minOracleAge: u64;
    maxOracleAge: u64;
    multiSourceQuorum: u32;
    fallbackOracle: option<bytes32>,
    emergencyStaleMode: StaleMode;
}

enum StaleMode {
    UseLastGoodPrice,
    HaltProtocol,
    UseFallback,
}
```

### Oracle Validation

```typescript
fn validateOracleData(data: OracleData, config: OracleSafetyConfig) -> Result {
    if data.circuitBreakerStatus != CircuitBreakerStatus::Active {
        return handleCircuitBreaker(data, config);
    }

    let age = now() - data.updatedAt;
    if age < config.minOracleAge {
        return Err::OracleTooFresh;
    }

    if age > config.maxOracleAge {
        return handleStaleData(data, config);
    }

    if data.liquidityDepth < config.minimumLiquidity {
        return Err::InsufficientLiquidity;
    }

    if data.deviation > config.maxDeviationPerBlock {
        return Err::ExcessiveDeviation;
    }

    if data.sourceCount < config.multiSourceQuorum {
        return Err::InsufficientSources;
    }

    Ok
}

fn handleStaleData(data: OracleData, config: OracleSafetyConfig) -> Result {
    match config.emergencyStaleMode {
        StaleMode::UseLastGoodPrice => {
            emitEvent(OracleStaleUsingLastGood { feedId: data.feedId, staleAt: data.updatedAt });
            Ok
        }
        StaleMode::HaltProtocol => {
            Err::ProtocolHaltedStaleOracle
        }
        StaleMode::UseFallback => {
            let fallback = config.fallbackOracle.unwrap();
            emitEvent(OracleFallbackActivated { primary: data.feedId, fallback, reason: "stale data" });
            Ok
        }
    }
}

fn handleCircuitBreaker(data: OracleData, config: OracleSafetyConfig) -> Result {
    match data.circuitBreakerStatus {
        CircuitBreakerStatus::Paused => Err::OraclePaused,
        CircuitBreakerStatus::Tripped => {
            emitEvent(OracleCircuitBreakerTripped { feedId: data.feedId, deviation: data.deviation });
            Err::OracleCircuitBreakerTripped
        }
        CircuitBreakerStatus::StaleData => handleStaleData(data, config),
        _ => Ok,
    }
}
```

### Same-Transaction Oracle Use Warning

```typescript
struct OracleUseRecord {
    feedId: bytes32;
    txHash: bytes32;
    blockNumber: u64;
    wasUpdated: bool,
    wasConsumed: bool,
}

fn detectSameTxUpdateAndConsume(records: vec<OracleUseRecord>) -> vec<SameTxWarning> {
    let mut warnings = vec![];

    let grouped = groupByTxHash(records);
    for (txHash, entries) in grouped {
        let updated = entries.iter().any(|r| r.wasUpdated);
        let consumed = entries.iter().any(|r| r.wasConsumed);
        if updated && consumed {
            warnings.push(SameTxWarning {
                feedId: entries[0].feedId,
                txHash,
                blockNumber: entries[0].blockNumber,
                severity: WarningSeverity::Critical,
                message: "Oracle was updated and consumed in the same transaction",
            });
        }
    }

    warnings
}

struct SameTxWarning {
    feedId: bytes32;
    txHash: bytes32;
    blockNumber: u64;
    severity: WarningSeverity;
    message: string,
}
```

### Unsafe Oracle Usage Visibility

```typescript
struct UnsafeOracleReport {
    protocol: address;
    feedId: bytes32;
    violations: vec<OracleViolation>;
    overallSafety: OracleSafetyGrade;
}

enum OracleViolation {
    NoMaxAgeCheck,
    NoDeviationCheck,
    NoLiquidityCheck,
    SingleSourceOnly,
    NoCircuitBreaker,
    SameTxUpdateAndConsume,
    NoFallbackOracle,
    AllowStaleDataWithoutHalt,
}

enum OracleSafetyGrade {
    Safe,
    MostlySafe,
    Risky,
    Unsafe,
}

fn computeSafetyGrade(violations: vec<OracleViolation>) -> OracleSafetyGrade {
    match violations.len() {
        0 => OracleSafetyGrade::Safe,
        1 => OracleSafetyGrade::MostlySafe,
        2..=3 => OracleSafetyGrade::Risky,
        _ => OracleSafetyGrade::Unsafe,
    }
}
```

### Wallet Display Format

```text
Oracle Safety Report for [Protocol Name]

Feed: NOVA/USD
Price: $12.45
Updated: 15 seconds ago
Sources: 5 of 5
Confidence: 98%
Liquidity: $2,400,000
Deviation: 0.02%

Safety Grade: SAFE

⚠ This protocol does not check oracle deviation limits
⚠ This protocol has no fallback oracle configured
```

## Rationale

Oracle safety is not optional. Protocols that skip age checks, deviation limits, or liquidity validation are inherently unsafe regardless of their other security measures. Same-transaction update-and-consume patterns are a primary vector for flash-loan oracle manipulation and must be flagged.

## Security Considerations

- Min oracle age prevents using a price that was pushed in the same block by an attacker
- Deviation limits create a mechanical circuit breaker for flash-crash scenarios
- Multi-source quorum prevents single-oracle compromise from affecting protocol behavior
- Same-transaction detection is post-hoc but enables rapid response to active manipulation
- Protocols graded Unsafe should trigger wallet warnings before user deposits
- Fallback oracles must not share failure modes with primary oracles
