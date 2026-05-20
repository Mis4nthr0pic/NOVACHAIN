# NRC-28: Wrapped Asset Mint Limits

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines mint limits for wrapped assets on NOVA. Wrapped assets enforce daily, per-transaction, and total supply caps so that bridge failures cannot cause infinite inflation in a single transaction.

## Motivation

Bridge exploits have resulted in hundreds of millions in losses, often because there is no limit on how many wrapped tokens can be minted. A compromised bridge can mint unlimited wrapped tokens, draining liquidity pools and collapsing the wrapped asset's value. NOVA v0.5 section 35.2 introduces hard mint limits as a protocol-level circuit breaker.

## Specification

### WrappedAssetConfig

```typescript
struct WrappedAssetConfig {
    assetId: bytes32;
    sourceChain: ChainId;
    bridgeId: bytes32;
    maxDailyMint: u256;
    maxTxMint: u256;
    maxTotalSupply: u256;
    emergencyPause: bool,
    slowExitMode: bool,
    proofDelay: u64;
    circuitBreakerThresholds: CircuitBreakerThresholds;
    governanceTimelock: u64;
}

struct CircuitBreakerThresholds {
    dailyMintPercentageTrigger: u32,
    txMintPercentageTrigger: u32,
    rapidMintTrigger: RapidMintTrigger,
    autoPauseEnabled: bool,
}

struct RapidMintTrigger {
    mintCountInWindow: u32,
    windowDuration: u64,
    totalAmountInWindow: u256,
}
```

### MintState

```typescript
struct MintState {
    assetId: bytes32;
    dailyMinted: u256;
    totalMinted: u256;
    lastResetTime: u64;
    paused: bool,
    pauseReason: option<PauseReason>,
    pausedAt: option<u64>,
    pausedBy: option<address>;
    recentMints: vec<MintRecord>;
}

enum PauseReason {
    CircuitBreakerTriggered,
    GovernanceAction,
    EmergencyAction,
    BridgeCompromise,
    AnomalousMintPattern,
}

struct MintRecord {
    amount: u256;
    txHash: bytes32;
    timestamp: u64;
    minter: address;
}
```

### Mint Validation

```typescript
fn validateMint(config: WrappedAssetConfig, state: MintState, amount: u256) -> Result {
    if state.paused {
        return Err::MintingPaused(state.pauseReason.unwrap());
    }

    if amount > config.maxTxMint {
        return Err::TxMintExceeded;
    }

    let dailyMinted = resetDailyIfNeeded(state);
    if dailyMinted + amount > config.maxDailyMint {
        return Err::DailyMintExceeded;
    }

    if state.totalMinted + amount > config.maxTotalSupply {
        return Err::TotalSupplyExceeded;
    }

    checkCircuitBreaker(config, state, amount)?;
    Ok
}
```

### Circuit Breaker

```typescript
fn checkCircuitBreaker(config: WrappedAssetConfig, state: MintState, amount: u256) -> Result {
    let thresholds = config.circuitBreakerThresholds;

    let dailyUsagePercent = (state.dailyMinted + amount) * 100 / config.maxDailyMint;
    if dailyUsagePercent >= thresholds.dailyMintPercentageTrigger {
        if thresholds.autoPauseEnabled {
            pauseMinting(state.assetId, PauseReason::CircuitBreakerTriggered);
        }
        emitEvent(DailyMintThresholdWarning {
            assetId: state.assetId,
            usagePercent: dailyUsagePercent,
            threshold: thresholds.dailyMintPercentageTrigger,
        });
    }

    let windowMints = getMintsInWindow(state, thresholds.rapidMintTrigger.windowDuration);
    let windowTotal = windowMints.iter().fold(0u256, |acc, m| acc + m.amount);
    if windowMints.len() >= thresholds.rapidMintTrigger.mintCountInWindow
        || windowTotal + amount >= thresholds.rapidMintTrigger.totalAmountInWindow
    {
        if thresholds.autoPauseEnabled {
            pauseMinting(state.assetId, PauseReason::AnomalousMintPattern);
        }
        return Err::RapidMintDetected;
    }

    Ok
}

fn pauseMinting(assetId: bytes32, reason: PauseReason) {
    let state = getMintState(assetId);
    MintState {
        paused: true,
        pauseReason: Some(reason),
        pausedAt: Some(now()),
        pausedBy: Some(msg.sender),
        ..state
    };
    emitEvent(MintPaused { assetId, reason, timestamp: now() });
}
```

### Daily Reset

```typescript
fn resetDailyIfNeeded(state: MintState) -> u256 {
    let daySeconds: u64 = 86400;
    if now() - state.lastResetTime >= daySeconds {
        0
    } else {
        state.dailyMinted
    }
}
```

### Emergency Controls

```typescript
fn emergencyPause(config: WrappedAssetConfig, caller: address, signatures: vec<Signature>) -> Result {
    verifyPauseAuthority(caller, config.bridgeId, signatures)?;

    pauseMinting(config.assetId, PauseReason::EmergencyAction);
    Ok
}

fn resumeMinting(config: WrappedAssetConfig, caller: address) -> Result {
    verifyGovernanceAuthority(caller, config.bridgeId)?;

    if !config.emergencyPause {
        return Err::EmergencyPauseNotEnabled;
    }

    let state = getMintState(config.assetId);
    let elapsed = now() - state.pausedAt.unwrap();

    if elapsed < config.governanceTimelock {
        return Err::GovernanceTimelockNotExpired;
    }

    MintState {
        paused: false,
        pauseReason: None,
        pausedAt: None,
        pausedBy: None,
        ..state
    };

    emitEvent(MintResumed { assetId: config.assetId, timestamp: now() });
    Ok
}
```

## Rationale

Hard mint caps create an absolute ceiling on bridge damage. Even if a bridge is fully compromised, the attacker cannot mint more than the daily cap, and the circuit breaker will halt minting well before that cap is reached in anomalous conditions.

## Security Considerations

- Circuit breaker thresholds should be calibrated to normal bridge usage patterns
- Auto-pause should default to enabled for new wrapped assets
- Governance timelock on resume prevents hasty re-enabling after a compromise
- Total supply cap provides an absolute upper bound regardless of other limits
- Mint records should be retained for forensic analysis after any circuit breaker event
