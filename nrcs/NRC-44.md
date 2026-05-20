# NRC-44: AI-Assisted Verification Guidelines

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines guidelines for using AI in formal verification of Nova contracts. AI can assist with proof generation, spec inference, vulnerability detection, and test generation. AI-generated outputs are advisory only and require human review and sign-off. AI model versions must be declared. False positives and negatives must be tracked. AI does not replace auditors or formal methods.

## Motivation

NOVA v0.5 section 46 addresses AI-assisted verification as a tooling concern. AI models are increasingly capable of generating proofs, inferring specifications, detecting vulnerabilities, and creating tests. However, AI outputs are probabilistic, not deterministic. An AI-generated proof may be syntactically valid but semantically wrong. An AI-inferred spec may miss critical properties. An AI-detected vulnerability may be a false positive.

Guidelines are necessary because the temptation to treat AI output as verified output is strong. Without explicit rules, AI-generated proofs could be submitted to the Pulsar Prover (NRC-40) without human review, AI-inferred specs could replace hand-written specs without validation, and AI vulnerability reports could trigger false alerts in monitoring systems (NRC-41).

## Specification

### AIVerificationAssist

```typescript
enum AITask {
    ProofGeneration,
    SpecInference,
    VulnerabilityDetection,
    TestGeneration,
}

struct AIVerificationAssist {
    toolId: string;
    task: AITask;
    inputHash: bytes32;
    outputHash: bytes32;
    humanReviewer: address;
    reviewStatus: ReviewStatus;
    aiModelVersion: string;
    timestamp: u64,
}
```

### ReviewStatus

```typescript
enum ReviewStatus {
    Pending,
    Accepted,
    Rejected,
    Modified,
}
```

### AI Output Record

```typescript
struct AIOutputRecord {
    assistId: string;
    toolId: string;
    task: AITask;
    modelVersion: AIModelVersion;
    inputHash: bytes32;
    outputHash: bytes32;
    createdAt: u64;
    reviewRecord: option<ReviewRecord>,
}

struct AIModelVersion {
    modelName: string;
    version: string;
    trainingCutoff: u64;
    provider: string,
}

struct ReviewRecord {
    reviewer: address;
    status: ReviewStatus;
    reviewedAt: u64;
    modifications: option<bytes32>,
    notes: string,
}
```

### AI Verification Rules

```typescript
struct AIVerificationRule {
    ruleId: string;
    description: string;
    category: RuleCategory;
    enforcement: EnforcementLevel,
}

enum RuleCategory {
    OutputClassification,
    HumanReview,
    ModelDeclaration,
    AccuracyTracking,
    ScopeLimitation,
}

enum EnforcementLevel {
    Mandatory,
    Recommended,
}
```

### Core Rules

```typescript
let aiVerificationRules: vec<AIVerificationRule> = [
    {
        ruleId: "AI-001",
        description: "AI outputs are advisory only and do not constitute verification",
        category: RuleCategory::OutputClassification,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-002",
        description: "All AI-generated proofs require human review before submission to Pulsar Prover",
        category: RuleCategory::HumanReview,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-003",
        description: "AI model version must be declared in all verification submissions",
        category: RuleCategory::ModelDeclaration,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-004",
        description: "False positives and false negatives must be tracked and reported",
        category: RuleCategory::AccuracyTracking,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-005",
        description: "AI does not replace auditors or formal methods",
        category: RuleCategory::ScopeLimitation,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-006",
        description: "AI-inferred specs must be validated against contract behavior before acceptance",
        category: RuleCategory::HumanReview,
        enforcement: EnforcementLevel::Mandatory,
    },
    {
        ruleId: "AI-007",
        description: "AI-generated tests must achieve the same coverage standards as hand-written tests",
        category: RuleCategory::ScopeLimitation,
        enforcement: EnforcementLevel::Recommended,
    },
];
```

### Accuracy Tracking

```typescript
struct AIAccuracyRecord {
    toolId: string;
    task: AITask;
    totalOutputs: u64;
    truePositives: u64;
    falsePositives: u64;
    falseNegatives: u64;
    trueNegatives: u64;
    period: TimePeriod,
}

struct TimePeriod {
    start: u64;
    end: u64,
}

fn computeAccuracyMetrics(record: AIAccuracyRecord) -> AccuracyMetrics {
    let precision = record.truePositives / (record.truePositives + record.falsePositives);
    let recall = record.truePositives / (record.truePositives + record.falseNegatives);
    let f1Score = 2 * precision * recall / (precision + recall);
    return {
        toolId: record.toolId,
        task: record.task,
        precision: precision,
        recall: recall,
        f1Score: f1Score,
        totalOutputs: record.totalOutputs,
        period: record.period,
    };
}

struct AccuracyMetrics {
    toolId: string;
    task: AITask;
    precision: u256;
    recall: u256;
    f1Score: u256;
    totalOutputs: u64;
    period: TimePeriod,
}
```

### Submission Workflow

```typescript
fn submitAIAssistedVerification(assist: AIVerificationAssist) -> SubmissionResult {
    if assist.reviewStatus != ReviewStatus::Accepted {
        return SubmissionResult::Err("AI output must be reviewed and accepted by a human before submission");
    }
    let existingRecord = getAIOutputRecord(assist.toolId, assist.outputHash);
    if existingRecord.isNone() {
        return SubmissionResult::Err("No AI output record found for the given tool and output hash");
    }
    let review = existingRecord.unwrap().reviewRecord;
    if review.isNone() {
        return SubmissionResult::Err("No review record attached to AI output");
    }
    if review.unwrap().reviewer != assist.humanReviewer {
        return SubmissionResult::Err("Reviewer mismatch");
    }
    if review.unwrap().status != ReviewStatus::Accepted {
        return SubmissionResult::Err("Review was not accepted");
    }
    return SubmissionResult::Ok(recordVerificationSubmission(assist));
}

enum SubmissionResult {
    Ok(string),
    Err(string),
}
```

### Prover Integration

When AI assists with proof generation for the Pulsar Prover (NRC-40):

```typescript
struct AIAssistedProofSubmission {
    proverOutput: ProverOutput;
    aiAssistRecords: vec<AIVerificationAssist>;
    humanReviewComplete: bool,
}

fn validateAIAssistedProof(submission: AIAssistedProofSubmission) -> ValidationResult {
    if !submission.humanReviewComplete {
        return ValidationResult::Err("Human review not complete for AI-assisted proof");
    }
    for record in submission.aiAssistRecords {
        if record.reviewStatus != ReviewStatus::Accepted {
            return ValidationResult::Err("AI assist record not reviewed: " + record.toolId);
        }
    }
    return validateProverOutput(submission.proverOutput);
}
```

### Display Format

```text
AI-Assisted Verification Report

  Tool:            pulsar-ai-assist v1.4.0
  Model:           GLM-5.1 (cutoff: 2025-10)
  Task:            Proof Generation
  Output hash:     0x9c2f...a4b7

  Review:
    Reviewer:      nova1auditor...
    Status:        Accepted
    Reviewed at:   2025-12-01 16:45 UTC
    Modifications: Minor (3 line changes in proof script)

  Accuracy (last 90 days):
    Precision:     87.3%
    Recall:        82.1%
    F1 Score:      84.6%
    Total outputs: 1,247

  Rules applied:
    AI-001: Advisory only           [enforced]
    AI-002: Human review required   [enforced]
    AI-003: Model version declared  [enforced]
    AI-004: Accuracy tracked        [enforced]
    AI-005: Does not replace auditor [enforced]
```

## Rationale

AI assistance accelerates verification but cannot replace human judgment. The guidelines establish clear boundaries: AI outputs are advisory, humans must review and sign off, model versions must be declared for reproducibility, and accuracy must be tracked to identify degradation. By requiring human review before submission, the system ensures that AI is a force multiplier for auditors, not a replacement.

Accuracy tracking creates accountability. If an AI tool consistently produces false positives, its outputs should be weighted accordingly. If it produces false negatives, the gaps must be identified and addressed through other verification methods.

## Security Considerations

- AI-generated proofs that bypass human review could introduce subtle errors. The mandatory review requirement must be enforced at the tooling level, not just as a guideline.
- AI model version declaration is necessary but not sufficient. The same model version may produce different outputs on different inputs. Reproducibility of AI outputs is inherently limited.
- False negatives in vulnerability detection are the most dangerous failure mode. A vulnerability that the AI misses and the human reviewer also misses creates a gap. AI should augment human review, not replace it.
- False positives in vulnerability detection create alert fatigue and can mask real issues. Accuracy metrics should be monitored for false positive rates.
- AI tools trained on vulnerable code may reproduce vulnerabilities in generated proofs or specs. Training data provenance should be considered when evaluating AI output quality.
- The AI tool provider has significant influence over verification outcomes. Decentralization of AI tooling reduces single-provider risk.
