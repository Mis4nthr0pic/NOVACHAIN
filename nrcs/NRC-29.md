# NRC-29: Safe Governance Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Defines a standard for transparent governance actions on NOVA. Every proposal must publish its full effect surface, and voting mechanisms must prevent manipulation through borrowed tokens or last-minute proposal changes.

## Motivation

Governance is the ultimate authority on a blockchain. If governance actions are opaque, token holders cannot assess what they are voting for. If voting is manipulable, attackers can pass harmful proposals through flash-loan-enabled vote buying. NOVA v0.5 section 40 requires that governance effects are fully visible before voting begins.

## Specification

### ProposalSurface

```typescript
struct ProposalSurface {
    proposalId: bytes32;
    assetsMoved: vec<AssetMovement>;
    contractsUpgraded: vec<ContractUpgrade>;
    permissionsGranted: vec<PermissionChange>;
    permissionsRevoked: vec<PermissionChange>;
    tokensMinted: vec<TokenMint>;
    adminRolesChanged: vec<RoleChange>;
    bridgesReconfigured: vec<BridgeReconfiguration>;
    parametersChanged: vec<ParameterChange>;
    proposer: address;
    proposedAt: u64;
}

struct AssetMovement {
    asset: address;
    amount: u256;
    from: address;
    to: address;
}

struct ContractUpgrade {
    contractAddress: address;
    currentCodeHash: bytes32;
    newCodeHash: bytes32;
    diffSummary: string;
}

struct PermissionChange {
    target: address;
    permission: string;
    grantee: address;
    scope: string;
}

struct TokenMint {
    token: address;
    amount: u256;
    recipient: address;
    reason: string;
}

struct RoleChange {
    role: string;
    target: address;
    action: RoleAction,
    grantee: option<address>,
}

enum RoleAction {
    Grant,
    Revoke,
    Transfer,
}

struct BridgeReconfiguration {
    bridgeId: bytes32;
    changes: vec<string>;
    riskImpact: BridgeRisk,
}

struct ParameterChange {
    parameter: string;
    currentValue: string;
    newValue: string;
    impact: string,
}
```

### Voting Checkpoint

```typescript
struct VotingCheckpoint {
    proposalId: bytes32;
    snapshotBlock: u64;
    snapshotTimestamp: u64;
    totalVotingPower: u256;
    quorumRequired: u256;
    borrowedTokenDetection: BorrowedTokenDetection;
}

struct BorrowedTokenDetection {
    enabled: bool,
    flashLoanWindow: u64,
    delegationChangesTracked: bool,
}

fn takeSnapshot(proposalId: bytes32) -> VotingCheckpoint {
    let block = currentBlock();
    VotingCheckpoint {
        proposalId,
        snapshotBlock: block.number,
        snapshotTimestamp: block.timestamp,
        totalVotingPower: computeTotalVotingPower(block.number),
        quorumRequired: computeQuorum(proposalId),
        borrowedTokenDetection: BorrowedTokenDetection {
            enabled: true,
            flashLoanWindow: 7200,
            delegationChangesTracked: true,
        },
    }
}

fn validateVotePower(checkpoint: VotingCheckpoint, voter: address, weight: u256) -> Result {
    let powerAtSnapshot = getVotingPower(voter, checkpoint.snapshotBlock);
    if weight > powerAtSnapshot {
        return Err::InsufficientVotingPower;
    }

    if checkpoint.borrowedTokenDetection.enabled {
        let recentDelegations = getDelegationsInWindow(
            voter,
            checkpoint.snapshotTimestamp - checkpoint.borrowedTokenDetection.flashLoanWindow,
            checkpoint.snapshotTimestamp,
        );
        if recentDelegations.len() > 0 {
            let borrowedPower = computeBorrowedPower(voter, recentDelegations);
            if borrowedPower > 0 {
                emitEvent(PossibleBorrowedVote { voter, borrowedPower, snapshot: checkpoint.snapshotBlock });
            }
        }
    }

    Ok
}
```

### Execution Timelock

```typescript
struct ExecutionTimelock {
    proposalId: bytes32;
    passedAt: u64;
    earliestExecution: u64;
    latestExecution: u64;
    vetoWindow: u64;
    emergencyVetoActive: bool,
}

fn executeProposal(proposal: ProposalSurface, timelock: ExecutionTimelock) -> Result {
    if now() < timelock.earliestExecution {
        return Err::TimelockNotExpired;
    }

    if now() > timelock.latestExecution {
        return Err::ProposalExpired;
    }

    if isVetoed(timelock.proposalId) {
        return Err::ProposalVetoed;
    }

    publishEffectSurface(proposal)?;
    executeEffects(proposal);
    emitEvent(ProposalExecuted { proposalId: proposal.proposalId, timestamp: now() });
    Ok
}
```

### Quorum and Emergency Veto

```typescript
struct QuorumConfig {
    standardQuorumBps: u32,
    criticalActionQuorumBps: u32,
    emergencyVetoQuorumBps: u32,
    criticalActions: vec<CriticalActionType>,
}

enum CriticalActionType {
    ContractUpgrade,
    BridgeReconfiguration,
    RoleGrant,
    ParameterChange,
    TokenMintAbove,
}

fn getRequiredQuorum(config: QuorumConfig, proposal: ProposalSurface) -> u256 {
    let hasCritical = proposal.contractsUpgraded.len() > 0
        || proposal.bridgesReconfigured.len() > 0
        || proposal.adminRolesChanged.len() > 0;

    let quorumBps = if hasCritical {
        config.criticalActionQuorumBps
    } else {
        config.standardQuorumBps
    };

    getTotalVotingPower() * quorumBps / 10000
}

fn emergencyVeto(proposalId: bytes32, vetoVoters: vec<(address, u256)>) -> Result {
    let config = getQuorumConfig();
    let totalVetoPower = vetoVoters.iter().fold(0u256, |acc, (_, w)| acc + w);
    let required = getTotalVotingPower() * config.emergencyVetoQuorumBps / 10000;

    if totalVetoPower < required {
        return Err::InsufficientVetoPower;
    }

    for voter in vetoVoters {
        validateVotePower(getCheckpoint(proposalId), voter.0, voter.1)?;
    }

    markVetoed(proposalId);
    emitEvent(EmergencyVetoExecuted { proposalId, vetoPower: totalVetoPower, timestamp: now() });
    Ok
}
```

## Rationale

Opaque governance enables social engineering at the protocol level. If voters cannot see that a proposal upgrades a core contract, they may approve it thinking it is a minor parameter change. Snapshot-before-proposal prevents flash-loan vote buying. Emergency veto provides a safety valve for proposals that pass but are later discovered to be harmful.

## Security Considerations

- Effect surface declarations must be verified against actual proposal execution code
- Snapshot timing must precede proposal publication to prevent advance trading on governance outcomes
- Emergency veto should require a lower threshold than passing a proposal, enabling the minority to block harmful actions
- Borrowed token detection is heuristic and may produce false positives; it should flag, not block
- Timelock durations should scale with the severity of the proposed changes
