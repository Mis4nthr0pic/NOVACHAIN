# NRC-41: On-Chain Anomaly Detection & Monitoring Hooks

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines on-chain monitoring hooks and anomaly detection for the Nova protocol. Monitor hooks observe specific behavioral patterns and emit structured anomaly events when thresholds are exceeded. Response actions range from alert emission to automatic pause. Monitoring data is published as on-chain metadata. Independent monitoring nodes subscribe to anomaly events. Detection thresholds are configurable per account and per protocol.

## Motivation

NOVA v0.5 section 43 establishes on-chain monitoring as a protocol-level concern. Today, anomaly detection is off-chain, opaque, and protocol-specific. Each application builds its own monitoring. There is no standardized way to detect unusual transfer velocity, oracle deviation, capability abuse, or governance anomalies at the protocol level.

On-chain monitoring hooks make detection visible, configurable, and composable. Instead of every application building bespoke monitoring, the protocol provides standard monitor types. Anomaly events are structured, machine-readable, and available to any subscriber. Response actions are graduated: not every anomaly requires a pause, but every anomaly should be visible.

## Specification

### MonitorHook

```typescript
enum MonitorType {
    TransferVelocity,
    OracleDeviation,
    CapabilityAbuse,
    GovernanceAnomaly,
}

struct MonitorHook {
    monitorType: MonitorType;
    monitorId: string;
    targetAddress: address;
    config: MonitorConfig;
    enabled: bool;
    registeredAt: u64,
}
```

### MonitorConfig

```typescript
struct MonitorConfig {
    thresholds: map<string, u256>;
    windowSeconds: u64;
    cooldownSeconds: u64;
    responseActions: vec<ResponseAction>;
    subscribers: vec<address>;
}

enum ResponseAction {
    EmitAlert,
    RequireConfirmation,
    AutoPause,
}
```

### TransferVelocityMonitor

```typescript
struct TransferVelocityConfig {
    maxTransfersPerWindow: u32;
    maxValuePerWindow: u256;
    windowSeconds: u64;
    uniqueRecipientThreshold: u32,
}

fn checkTransferVelocity(account: address, transfer: Transfer, config: TransferVelocityConfig) -> option<AnomalyEvent> {
    let window = getRecentTransfers(account, config.windowSeconds);
    if window.len() > config.maxTransfersPerWindow {
        return some(buildAnomaly(account, "transfer_count_exceeded", Severity::High));
    }
    let totalValue = sumTransferValues(window) + transfer.amount;
    if totalValue > config.maxValuePerWindow {
        return some(buildAnomaly(account, "transfer_value_exceeded", Severity::Critical));
    }
    let uniqueRecipients = countUniqueRecipients(window);
    if uniqueRecipients > config.uniqueRecipientThreshold {
        return some(buildAnomaly(account, "unusual_recipient_pattern", Severity::Medium));
    }
    return none;
}
```

### OracleDeviationMonitor

```typescript
struct OracleDeviationConfig {
    maxDeviationPercent: u8;
    comparisonWindowSeconds: u64;
    minSamplesForAlert: u32,
}

fn checkOracleDeviation(oracle: address, newValue: u256, config: OracleDeviationConfig) -> option<AnomalyEvent> {
    let historicalValues = getOracleValues(oracle, config.comparisonWindowSeconds);
    if historicalValues.len() < config.minSamplesForAlert {
        return none;
    }
    let median = computeMedian(historicalValues);
    let deviation = absDiff(newValue, median) * 100 / median;
    if deviation > config.maxDeviationPercent {
        return some(buildAnomaly(oracle, "oracle_deviation_exceeded", Severity::Critical));
    }
    return none;
}
```

### CapabilityAbuseMonitor

```typescript
struct CapabilityAbuseConfig {
    maxInvocationsPerWindow: u32;
    windowSeconds: u64;
    restrictedCapabilities: vec<string>,
}

fn checkCapabilityAbuse(account: address, capability: string, config: CapabilityAbuseConfig) -> option<AnomalyEvent> {
    let recentInvocations = getRecentCapabilityUses(account, capability, config.windowSeconds);
    if recentInvocations > config.maxInvocationsPerWindow {
        return some(buildAnomaly(account, "capability_invocation_exceeded", Severity::High));
    }
    if config.restrictedCapabilities.contains(capability) {
        return some(buildAnomaly(account, "restricted_capability_used", Severity::Medium));
    }
    return none;
}
```

### GovernanceAnomalyMonitor

```typescript
struct GovernanceAnomalyConfig {
    maxProposalsPerWindow: u32;
    windowSeconds: u64;
    minVotingPeriodSeconds: u64;
    maxParameterChangePercent: u8,
}

fn checkGovernanceAnomaly(proposal: GovernanceProposal, config: GovernanceAnomalyConfig) -> option<AnomalyEvent> {
    let recentProposals = getRecentProposals(proposal.proposer, config.windowSeconds);
    if recentProposals.len() > config.maxProposalsPerWindow {
        return some(buildAnomaly(proposal.proposer, "proposal_velocity_exceeded", Severity::Medium));
    }
    let votingPeriod = proposal.votingEndsAt - proposal.votingStartsAt;
    if votingPeriod < config.minVotingPeriodSeconds {
        return some(buildAnomaly(proposal.proposer, "insufficient_voting_period", Severity::High));
    }
    let valueChange = computeParameterChange(proposal);
    if valueChange > config.maxParameterChangePercent {
        return some(buildAnomaly(proposal.proposer, "excessive_parameter_change", Severity::High));
    }
    return none;
}
```

### AnomalyEvent

```typescript
enum Severity {
    Low,
    Medium,
    High,
    Critical,
}

struct AnomalyEvent {
    detector: address;
    anomalyType: string;
    severity: Severity;
    details: string;
    timestamp: u64;
    triggeredActions: vec<ResponseAction>;
    relatedEntities: vec<address>,
    monitorId: string,
}
```

### Response Action Execution

```typescript
fn executeResponseAction(event: AnomalyEvent) -> vec<ActionResult> {
    let results: vec<ActionResult> = [];
    for action in event.triggeredActions {
        match action {
            ResponseAction::EmitAlert => {
                emitAnomalyEvent(event);
                results.push({ action: action, success: true, message: "Alert emitted" });
            },
            ResponseAction::RequireConfirmation => {
                let confirmed = requestOnChainConfirmation(event);
                results.push({ action: action, success: confirmed, message: "Confirmation requested" });
            },
            ResponseAction::AutoPause => {
                if event.severity == Severity::Critical {
                    pauseTarget(event.detector);
                    results.push({ action: action, success: true, message: "Target paused" });
                } else {
                    emitAnomalyEvent(event);
                    results.push({ action: action, success: false, message: "Auto-pause requires Critical severity" });
                }
            },
        }
    }
    return results;
}

struct ActionResult {
    action: ResponseAction;
    success: bool;
    message: string,
}
```

### Monitoring Node Subscription

```typescript
struct MonitoringSubscription {
    nodeAddress: address;
    subscribedMonitors: vec<string>;
    subscriptionType: SubscriptionType;
    registeredAt: u64,
}

enum SubscriptionType {
    AllEvents,
    SeverityThreshold(Severity),
    MonitorType(MonitorType),
}

fn subscribeMonitor(subscription: MonitoringSubscription) -> bool {
    for monitorId in subscription.subscribedMonitors {
        addSubscriber(monitorId, subscription.nodeAddress);
    }
    return true;
}
```

### Per-Account Threshold Configuration

```typescript
struct AccountThresholdConfig {
    account: address;
    monitorOverrides: map<MonitorType, MonitorConfig>;
    updatedAt: u64,
    updatedBy: address,
}

fn setAccountThresholds(config: AccountThresholdConfig) -> bool {
    let account = getAccount(config.account);
    if config.updatedBy != account.admin {
        return false;
    }
    for entry in config.monitorOverrides.entries() {
        updateMonitorConfig(config.account, entry.key, entry.value);
    }
    return true;
}
```

### Anomaly Display

```text
Anomaly Alert

  Detector:   nova1monitor...
  Type:       transfer_velocity_exceeded
  Severity:   CRITICAL
  Time:       2025-12-01 14:32:05 UTC
  Monitor:    transfer-vel-001

  Details:
  Account nova1abc... made 47 transfers in 5 minutes
  (threshold: 30 per 5 minutes). Total value: 2,340,000 NOVA.

  Actions triggered:
    [x] Alert emitted
    [x] Auto-pause executed
    [ ] Confirmation requested

  Related entities:
    nova1abc... (source account)
    nova1dex...  (primary recipient)
```

## Rationale

Four monitor types cover the most critical anomaly patterns: transfer velocity catches draining attacks, oracle deviation catches price manipulation, capability abuse catches unauthorized or excessive privilege use, and governance anomaly catches governance attacks. The graduated response model avoids overreaction: low-severity anomalies emit alerts, critical anomalies can trigger automatic pauses.

Per-account configuration allows protocols to set their own thresholds based on their risk profile. Independent monitoring nodes create redundancy and reduce trust in any single monitor.

## Security Considerations

- Monitoring hooks add gas overhead to every monitored operation. Threshold configuration should minimize unnecessary checks.
- Auto-pause on Critical severity is powerful but dangerous if triggered by false positives. The severity classification must be conservative.
- Monitoring nodes see anomaly events that may contain sensitive operational data. Subscriptions should be authenticated and anomaly details scoped appropriately.
- Thresholds that are set too high miss anomalies. Thresholds that are set too low create alert fatigue. Default thresholds should be reviewed regularly.
- A compromised monitoring node could suppress anomaly events or generate false events. Multiple independent nodes mitigate this risk.
- The governance anomaly monitor checks proposal patterns but cannot assess proposal content. A sophisticated governance attack that spreads proposals across accounts may evade velocity detection.
