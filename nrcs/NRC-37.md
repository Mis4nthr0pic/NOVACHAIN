# NRC-37: Governance Proposal Intent Hash

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines a canonical intent hash for governance proposals. Voters vote on a structured description of proposal effects, not arbitrary bytes. If proposal execution bytes do not match the declared intent, execution fails.

## Motivation

NOVA v0.5 section 40.2 requires governance proposals to carry canonical intent hashes. Today, governance proposals are opaque byte sequences. Voters approve or reject proposals they cannot fully decode. This creates a gap between what voters think they are approving and what the proposal actually does.

By requiring a structured intent declaration and hashing it canonically, voters can review human-readable effects. The execution layer verifies that execution matches the declared intent. This connects governance safety to NOVA's core typed intent model.

## Specification

### ProposalIntent

```typescript
struct AssetMovement {
    asset: address;
    amount: u256;
    from: address;
    to: address;
}

struct ContractUpgrade {
    contractAddress: address;
    newCodeHash: bytes32;
}

struct RoleChange {
    contractAddress: address;
    role: string;
    holder: address;
    action: string;
}

struct TokenMint {
    token: address;
    amount: u256;
    recipient: address;
}

struct OracleChange {
    oracle: address;
    feed: string;
    action: string;
    newOracle: option<address>;
}

struct BridgeReconfiguration {
    bridge: address;
    chainId: u64;
    parameter: string;
    newValue: string;
}

struct DelayChange {
    contractAddress: address;
    delayType: string;
    oldDelay: u64;
    newDelay: u64;
}

struct VetoPowerChange {
    contractAddress: address;
    vetoHolder: address;
    action: string;
}

struct ProposalIntent {
    assetsMoved: vec<AssetMovement>;
    contractsUpgraded: vec<ContractUpgrade>;
    rolesChanged: vec<RoleChange>;
    tokensMinted: vec<TokenMint>;
    oraclesChanged: vec<OracleChange>;
    bridgesReconfigured: vec<BridgeReconfiguration>;
    delaysChanged: vec<DelayChange>;
    vetoPowersChanged: vec<VetoPowerChange>;
}
```

### Intent Hash Computation

```text
proposalIntentHash = BLAKE3-256("NOVA_GOVERNANCE_INTENT_V1" || canonical_proposal_intent_bytes)
```

```typescript
fn computeIntentHash(intent: ProposalIntent) -> bytes32 {
    let encoded = canonicalEncode(intent);
    return blake3_256("NOVA_GOVERNANCE_INTENT_V1" || encoded);
}
```

### Governance Proposal

```typescript
struct GovernanceProposal {
    proposalId: u64;
    proposer: address;
    intent: ProposalIntent;
    intentHash: bytes32;
    executionCode: bytes;
    votingStartsAt: u64;
    votingEndsAt: u64;
    executionDelay: u64;
    status: ProposalStatus;
}

enum ProposalStatus {
    Pending,
    Active,
    Passed,
    Executed,
    Failed,
    Expired,
}
```

### Execution Validation

```typescript
fn executeProposal(proposal: GovernanceProposal) -> Result {
    let computedHash = computeIntentHash(proposal.intent);
    if computedHash != proposal.intentHash {
        return Err("Intent hash mismatch: proposal may have been tampered with");
    }
    let executionIntent = decodeExecutionIntent(proposal.executionCode);
    if !intentsMatch(proposal.intent, executionIntent) {
        return Err("Execution does not match declared intent");
    }
    return applyExecution(proposal.executionCode);
}
```

### Voter Display

Wallets must render the structured intent for voters:

```text
Governance Proposal #42

Intent Hash:
0x3a7f...b2c1

Effects:
  Assets moved:
    10,000 NOVA from treasury to nova1abc...

  Contracts upgraded:
    LendingPool -> 0x8d2e...

  Roles changed:
    Pauser role granted to nova1def... on LendingPool

  Tokens minted:
    None

  Oracles changed:
    None

  Bridges reconfigured:
    None

  Delays changed:
    Upgrade delay: 24h -> 48h on LendingPool

  Veto powers changed:
    None

Vote: [For] [Against] [Abstain]
```

### Intent Matching

```typescript
fn intentsMatch(declared: ProposalIntent, execution: ProposalIntent) -> bool {
    if declared.assetsMoved.len() != execution.assetsMoved.len() { return false; }
    if declared.contractsUpgraded.len() != execution.contractsUpgraded.len() { return false; }
    if declared.rolesChanged.len() != execution.rolesChanged.len() { return false; }
    if declared.tokensMinted.len() != execution.tokensMinted.len() { return false; }
    if declared.oraclesChanged.len() != execution.oraclesChanged.len() { return false; }
    if declared.bridgesReconfigured.len() != execution.bridgesReconfigured.len() { return false; }
    if declared.delaysChanged.len() != execution.delaysChanged.len() { return false; }
    if declared.vetoPowersChanged.len() != execution.vetoPowersChanged.len() { return false; }
    return true;
}
```

## Rationale

Structured intent transforms governance from "trust the bytes" to "review the effects." Each effect category is explicitly enumerated so that voters can quickly assess scope. The intent hash commits to the canonical encoding, making tampering detectable. The execution-time mismatch check ensures that the bytes actually do what the intent says.

## Security Considerations

- The intent hash only covers declared effects. A malicious proposal could declare an incomplete intent. Voters must review the full intent, not just the hash.
- Intent matching compares structure and counts but not necessarily all field values. Comprehensive matching should be implemented in the execution layer.
- The canonical encoding must be deterministic. Any ambiguity in encoding creates hash collision risks.
- Execution code that has effects outside the declared intent categories (e.g., arbitrary storage writes) cannot be fully captured by this model. Additional execution sandboxing may be necessary.
- The proposer can set misleading intent descriptions. Voter review tooling should highlight any unusual patterns in the intent fields.
