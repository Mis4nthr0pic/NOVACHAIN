# NRC-26: Wallet Risk Label Standard

**Status:** Draft  
**Category:** Wallet UX  

## Abstract

Defines a standard for wallet and explorer risk labeling. Every transaction must display its risk surface in plain language, and high-risk actions cannot hide behind vague labels.

## Motivation

Wallets today display transactions as opaque hex data or generic "contract interaction" labels. Users approve transactions they do not understand. NOVA v0.5 section 37 requires that wallets label risk in plain language so users can make informed decisions before signing.

## Specification

### RiskLabel

```typescript
enum RiskLabel {
    FundsTransfer,
    SpendingPermission,
    CodeChange,
    ContractUpgrade,
    OracleChange,
    BridgePermissionChange,
    SignerAddition,
    GuardianRemoval,
    DelayDisabled,
    FutureWithdrawals,
    TokenApproval,
    DelegationGranted,
    RecoveryInitiated,
    KeyRotation,
    BridgeExecution,
    GovernanceVote,
    AssetMint,
}
```

### TransactionRiskDisplay

```typescript
struct TransactionRiskDisplay {
    action: string;
    expectedResult: string;
    risks: vec<string>;
    permissionCreated: option<PermissionDetail>;
    riskLabels: vec<RiskLabel>;
    severity: RiskSeverity;
    requiresExplicitConfirmation: bool;
}

struct PermissionDetail {
    amount: option<u256>;
    token: option<address>;
    expiry: option<u64>;
    oneTime: bool,
    recipient: option<address>;
    methods: vec<string>;
}

enum RiskSeverity {
    Informational,
    Low,
    Medium,
    High,
    Critical,
}
```

### Display Formatting Rules

```typescript
fn formatRiskDisplay(display: TransactionRiskDisplay) -> string {
    let header = format("Action: {}", display.action);
    let expected = format("Expected result: {}", display.expectedResult);

    let labels = display.riskLabels.iter().map(|label| {
        match label {
            RiskLabel::FundsTransfer => "You are sending funds",
            RiskLabel::SpendingPermission => "You are granting spending permission",
            RiskLabel::CodeChange => "You are changing contract code",
            RiskLabel::ContractUpgrade => "You are upgrading a contract",
            RiskLabel::OracleChange => "You are changing an oracle source",
            RiskLabel::BridgePermissionChange => "You are changing bridge permissions",
            RiskLabel::SignerAddition => "You are adding a new signer to your account",
            RiskLabel::GuardianRemoval => "You are removing a recovery guardian",
            RiskLabel::DelayDisabled => "You are disabling a security delay",
            RiskLabel::FutureWithdrawals => "You are authorizing future withdrawals",
            RiskLabel::TokenApproval => "You are approving token spending",
            RiskLabel::DelegationGranted => "You are delegating authority",
            RiskLabel::RecoveryInitiated => "A recovery process is being initiated",
            RiskLabel::KeyRotation => "You are rotating a signing key",
            RiskLabel::BridgeExecution => "You are executing a bridge transfer",
            RiskLabel::GovernanceVote => "You are casting a governance vote",
            RiskLabel::AssetMint => "You are minting an asset",
        }
    }).collect::<vec<string>>();

    let riskSummary = format("Risks: {}", display.risks.join(", "));
    let permDetail = match display.permissionCreated {
        Some(p) => formatPermission(p),
        None => "No new permissions created".to_string(),
    };

    format("{}\n{}\n{}\n{}\n{}", header, expected, labels.join("\n"), riskSummary, permDetail)
}
```

### High-Risk Action Rules

```typescript
fn isHighRiskAction(labels: vec<RiskLabel>) -> bool {
    let highRiskLabels = vec![
        RiskLabel::SpendingPermission,
        RiskLabel::CodeChange,
        RiskLabel::ContractUpgrade,
        RiskLabel::OracleChange,
        RiskLabel::GuardianRemoval,
        RiskLabel::DelayDisabled,
        RiskLabel::SignerAddition,
    ];

    labels.iter().any(|label| highRiskLabels.contains(label))
}

fn enforceDisplayRules(display: TransactionRiskDisplay) -> DisplayEnforcement {
    if isHighRiskAction(display.riskLabels) {
        return DisplayEnforcement {
            mustShowDetailedBreakdown: true,
            mustShowExactAmounts: true,
            mustShowAllRecipients: true,
            cannotUseGenericLabel: true,
            requiresDoubleConfirm: true,
            confirmDelaySeconds: 5,
        };
    }

    DisplayEnforcement {
        mustShowDetailedBreakdown: false,
        mustShowExactAmounts: true,
        mustShowAllRecipients: true,
        cannotUseGenericLabel: false,
        requiresDoubleConfirm: false,
        confirmDelaySeconds: 0,
    }
}
```

### Prohibited Labels

Wallets MUST NOT display any of the following for high-risk actions:

```typescript
enum ProhibitedLabel {
    GenericContractInteraction,
    UnspecifiedMethodCall,
    RawHexData,
    TechnicalOnlyDescription,
}
```

A transaction that transfers 10,000 NOVA must not be displayed as "Contract Interaction". It must say "You are sending 10,000 NOVA to 0x..." with the FundsTransfer label.

## Rationale

Users cannot protect themselves from what they cannot see. Plain-language risk labels transform opaque transaction data into actionable information. Prohibiting generic labels for high-risk actions eliminates the most common vector for social engineering attacks in wallet approvals.

## Security Considerations

- Wallets that do not implement risk labeling should be flagged as non-compliant
- The risk label set must be extensible through NRC updates as new attack patterns emerge
- Display formatting rules are minimum requirements; wallets may provide additional detail
- Double-confirmation for high-risk actions creates a friction point that prevents impulsive approvals
