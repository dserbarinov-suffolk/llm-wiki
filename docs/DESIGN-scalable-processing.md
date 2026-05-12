# Scalable File Processing Service Design

> This document is the implementation source of truth for the document processing service that
> consumes Zarya file uploads and maintains LLM Wikis. It builds on
> [SCALABLE-FILE-PROCESSING-HANDOFF.md](SCALABLE-FILE-PROCESSING-HANDOFF.md).

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture](#2-system-architecture)
3. [Data Models](#3-data-models)
4. [Phase Implementations](#4-phase-implementations)
5. [Concurrency and Rate Limiting](#5-concurrency-and-rate-limiting)
6. [AWS Infrastructure](#6-aws-infrastructure)
7. [API Design](#7-api-design)
8. [Observability](#8-observability)
9. [Failure Handling](#9-failure-handling)
10. [Security](#10-security)
11. [Developer Workflow & Testing](#11-developer-workflow--testing)
12. [IaC Layout](#12-iac-layout)
13. [CI/CD Pipeline](#13-cicd-pipeline)
14. [Drift Detection](#14-drift-detection)
15. [Deployment](#15-deployment)
16. [Cost Estimation](#16-cost-estimation)

---

## 1. Executive Summary

### 1.1 Problem Statement

Users upload up to 60 PDFs at once to Zarya folders. Each PDF must be:

1. Converted to normalized Markdown
2. Processed by LLM to extract claims
3. Synthesized into wiki pages
4. Merged into a persistent folder-level wiki

The system must handle this workload without overwhelming LLM rate limits, while providing
visibility into processing status and producing high-quality, contradiction-aware wikis.

### 1.2 Key Design Decisions

| Decision      | Choice                          | Rationale                                                                 |
| ------------- | ------------------------------- | ------------------------------------------------------------------------- |
| LLM Backend   | AWS Bedrock                     | Managed inference, IAM auth, no API key management                        |
| LLM Model     | Qwen3-Coder 30b (config-driven) | Optimized for extraction, ~10x cheaper than Claude, switchable via config |
| Orchestrator  | AWS Step Functions              | Visual workflow, built-in retries, Map state for parallelism              |
| Queue System  | SQS with FIFO for wiki writes   | Rate limit control, FIFO ordering for wiki serialization                  |
| State Storage | DynamoDB                        | Serverless, streams for triggers, conditional writes for locks            |
| Wiki Storage  | S3                              | Durable Markdown, versioned, cheap                                        |
| Compute       | Lambda + ECS Fargate            | Lambda for most work, Fargate for large PDFs                              |

### 1.3 Target Performance

| Metric                        | Target        | Notes                             |
| ----------------------------- | ------------- | --------------------------------- |
| 60 PDFs total processing time | < 4 hours     | Assumes 3 concurrent LLM workers  |
| Single PDF processing time    | 5-30 minutes  | Depends on page count             |
| Phase 0 (PDF→MD)              | 1-5 minutes   | CPU-bound                         |
| Phase 1 (extract claims)      | 2-10 minutes  | LLM-bound, ~10 chunks             |
| Phase 2 (synthesize)          | 2-15 minutes  | LLM-bound, ~5-10 pages            |
| Phase 3 (wiki maintenance)    | 30-60 seconds | Serialized per folder             |
| Cost per PDF (Qwen default)   | ~$0.08        | ~10x cheaper than Claude (~$0.45) |

### 1.4 Prerequisites

Before deploying this service, the following Zarya infrastructure changes must be deployed:

| Prerequisite             | Change Required                                                              | Deploy Order                      |
| ------------------------ | ---------------------------------------------------------------------------- | --------------------------------- |
| **DynamoDB Streams**     | Enable `StreamSpecification: NEW_IMAGE` on Zarya's `FileVersionsTable`       | Before processor foundation stack |
| **Stream ARN Export**    | Add CloudFormation Export `ZaryaFileVersionsStreamArn` from foundation stack | Before processor foundation stack |
| **Cross-account access** | If processor runs in separate account, add IAM policy for stream access      | With processor foundation stack   |

Example Zarya IaC change (in Zarya's foundation CloudFormation):

```yaml
Resources:
  FileVersionsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      # ... existing properties ...
      StreamSpecification:
        StreamViewType: NEW_IMAGE

Outputs:
  FileVersionsTableStreamArn:
    Description: DynamoDB Stream ARN for FileVersions table
    Value: !GetAtt FileVersionsTable.StreamArn
    Export:
      Name: !Sub ${Environment}-ZaryaFileVersionsStreamArn
```

The processor foundation stack imports this via:

```yaml
Parameters:
  ZaryaFileVersionsStreamArn:
    Type: String
    Default: ""
    Description: Zarya FileVersions stream ARN (imported or provided)

Conditions:
  UseImportedStreamArn: !Equals [!Ref ZaryaFileVersionsStreamArn, ""]

Resources:
  StreamDispatcherEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      FunctionName: !Ref StreamDispatcherFunction
      EventSourceArn:
        Fn::If:
          - UseImportedStreamArn
          - Fn::ImportValue:
              Fn::Sub: "${Environment}-ZaryaFileVersionsStreamArn"
          - !Ref ZaryaFileVersionsStreamArn
      StartingPosition: LATEST
      BatchSize: 10
      FilterCriteria:
        Filters:
          - Pattern: '{"eventName": ["INSERT"]}'
```

> **Note**: `UseImportedStreamArn` is true when the parameter is empty, meaning "import from
> Zarya's CloudFormation export." When a specific ARN is provided, the condition is false and
> the provided ARN is used directly.

---

## 2. System Architecture

### 2.1 High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ZARYA (Source System)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  User Upload → S3 pending-uploads/ → S3 objects/ → FileVersion committed   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ DynamoDB Stream
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STREAM DISPATCHER (Lambda)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Filter: INSERT only, PDF/MD content types, size ≤ 250MB                   │
│  Enrich: Load File record for folderId, validate currentVersionId          │
│  Emit: FileVersionCommitted event → Intake Queue                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ SQS Standard Queue
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WORKFLOW ORCHESTRATOR (Step Functions)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│   │ Phase 0  │───▶│ Phase 1  │───▶│ Phase 2  │───▶│ Phase 3  │            │
│   │ Normalize│    │ Extract  │    │Synthesize│    │ Maintain │            │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘            │
│        │               │               │               │                   │
│        ▼               ▼               ▼               ▼                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│   │  Lambda  │    │  Lambda  │    │  Lambda  │    │  Lambda  │            │
│   │ or ECS   │    │ w/ Rate  │    │ w/ Rate  │    │ w/ Lock  │            │
│   └──────────┘    │  Limiter │    │  Limiter │    │ (FIFO)   │            │
│                   └──────────┘    └──────────┘    └──────────┘            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROCESSOR-OWNED STORAGE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  S3: processing/{fileVersionId}/*, wikis/{folderId}/*                      │
│  DynamoDB: ProcessingJobs, ProcessingTasks, WikiLocks                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Inventory

| Component           | AWS Service                                | Purpose                                         |
| ------------------- | ------------------------------------------ | ----------------------------------------------- |
| Stream Dispatcher   | Lambda (DynamoDB Stream trigger)           | Filter/enrich FileVersion events                |
| Intake Queue        | SQS Standard                               | Buffer incoming work                            |
| **Intake Consumer** | **Lambda (SQS trigger)**                   | **Start Step Functions with idempotency check** |
| Workflow Engine     | Step Functions                             | Orchestrate phases per file                     |
| Phase 0 Worker      | Lambda (default), ECS Fargate (large PDFs) | PDF→MD conversion                               |
| Phase 1 Worker      | Lambda                                     | Claim extraction with Bedrock                   |
| Phase 2 Worker      | Lambda                                     | Page synthesis with Bedrock                     |
| Phase 3 Queue       | SQS FIFO                                   | Serialize wiki writes per folder                |
| Phase 3 Worker      | Lambda (SQS trigger)                       | Wiki maintenance with callback                  |
| Rate Limiter        | DynamoDB + Lambda layer                    | Token bucket per Bedrock model                  |
| Jobs Table          | DynamoDB                                   | Workflow state                                  |
| Tasks Table         | DynamoDB                                   | Per-phase task state                            |
| Wiki Locks Table    | DynamoDB                                   | Defense-in-depth lock for Phase 3               |
| Processing Bucket   | S3                                         | Intermediate artifacts                          |
| Wiki Bucket         | S3                                         | Persistent wiki Markdown                        |

### 2.2.1 Intake Consumer (Lambda)

The Intake Consumer is a Lambda function that polls the intake queue and starts Step Functions
executions with explicit idempotency control via `fileVersionId` as the execution name.

> **Design Decision**: We use a Lambda consumer instead of EventBridge Pipes because Pipes cannot
> reliably set the Step Functions execution name for idempotency. The Lambda approach handles
> `ExecutionAlreadyExists` explicitly, ensuring exactly-once workflow initiation.

```yaml
IntakeConsumerFunction:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: !Sub processing-intake-consumer-${Environment}
    Runtime: nodejs20.x
    Handler: index.handler
    MemorySize: 256
    Timeout: 30
    ReservedConcurrentExecutions: 10
    Role: !GetAtt IntakeConsumerRole.Arn
    Environment:
      Variables:
        STATE_MACHINE_ARN: !GetAtt ProcessingWorkflow.Arn

IntakeConsumerEventSourceMapping:
  Type: AWS::Lambda::EventSourceMapping
  Properties:
    FunctionName: !Ref IntakeConsumerFunction
    EventSourceArn: !GetAtt IntakeQueue.Arn
    BatchSize: 1
    Enabled: true

IntakeConsumerRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
    Policies:
      - PolicyName: IntakeConsumerPolicy
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
                - sqs:ReceiveMessage
                - sqs:DeleteMessage
                - sqs:GetQueueAttributes
              Resource: !GetAtt IntakeQueue.Arn
            - Effect: Allow
              Action:
                - states:StartExecution
              Resource: !GetAtt ProcessingWorkflow.Arn
```

**Idempotency Implementation**:

```typescript
// intake-consumer/index.ts
import {
  SFNClient,
  StartExecutionCommand,
  ExecutionAlreadyExists,
} from "@aws-sdk/client-sfn";

const sfn = new SFNClient({});

export async function handler(event: SQSEvent): Promise<void> {
  for (const record of event.Records) {
    const message = JSON.parse(record.body) as FileVersionCommittedEvent;
    const executionName = message.fileVersionId; // Idempotency key

    // Map event fields to state machine input contract
    // FileVersionCommittedEvent uses objectKey/sizeBytes/contentType
    // State machine expects sourceObjectKey/sourceSizeBytes/sourceContentType
    const input = {
      fileVersionId: message.fileVersionId,
      fileId: message.fileId,
      folderId: message.folderId,
      sourceObjectKey: message.objectKey,
      sourceSizeBytes: message.sizeBytes,
      sourceContentType: message.contentType,
      sourceEtagOrChecksum: message.etagOrChecksum,
    };

    try {
      await sfn.send(
        new StartExecutionCommand({
          stateMachineArn: process.env.STATE_MACHINE_ARN,
          name: executionName,
          input: JSON.stringify(input),
        }),
      );
      console.log(`Started execution ${executionName}`);
    } catch (error) {
      if (error instanceof ExecutionAlreadyExists) {
        // Idempotent: execution already running or completed
        console.log(`Execution ${executionName} already exists, skipping`);
      } else {
        throw error; // Let SQS retry
      }
    }
  }
}
```

### 2.3 Step Functions State Machine

The state machine preserves job context through all phases using `ResultPath` to merge task outputs
with the original input. Each phase adds its outputs without overwriting `fileVersionId`, `folderId`,
`fileId`, or artifact keys.

**Input shape** (from intake consumer):

```json
{
  "fileVersionId": "v-123",
  "fileId": "f-456",
  "folderId": "folder-789",
  "sourceObjectKey": "objects/...",
  "sourceSizeBytes": 1234567,
  "sourceContentType": "application/pdf"
}
```

**State Machine Definition**:

```json
{
  "Comment": "Process one FileVersion through all phases",
  "StartAt": "ValidateAndEnrich",
  "States": {
    "ValidateAndEnrich": {
      "Type": "Task",
      "Resource": "${ValidateAndEnrichFunctionArn}",
      "ResultPath": "$.validation",
      "Next": "Phase0Normalize",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ]
    },
    "Phase0Normalize": {
      "Type": "Task",
      "Resource": "${Phase0NormalizeFunctionArn}",
      "ResultPath": "$.phase0",
      "TimeoutSeconds": 600,
      "Retry": [{ "ErrorEquals": ["States.Timeout"], "MaxAttempts": 2 }],
      "Next": "Phase1ExtractClaims",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ]
    },
    "Phase1ExtractClaims": {
      "Type": "Map",
      "ItemsPath": "$.phase0.chunks",
      "ItemSelector": {
        "fileVersionId.$": "$.fileVersionId",
        "folderId.$": "$.folderId",
        "chunk.$": "$$.Map.Item.Value"
      },
      "MaxConcurrency": 5,
      "ResultPath": "$.phase1.chunkResults",
      "Iterator": {
        "StartAt": "ExtractChunk",
        "States": {
          "ExtractChunk": {
            "Type": "Task",
            "Resource": "${ExtractChunkFunctionArn}",
            "Retry": [
              {
                "ErrorEquals": ["RateLimitError"],
                "IntervalSeconds": 30,
                "MaxAttempts": 5,
                "BackoffRate": 1.5
              }
            ],
            "End": true
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ],
      "Next": "AggregateClaims"
    },
    "AggregateClaims": {
      "Type": "Task",
      "Resource": "${AggregateClaimsFunctionArn}",
      "ResultPath": "$.phase1.aggregation",
      "Next": "Phase2Synthesize",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ]
    },
    "Phase2Synthesize": {
      "Type": "Map",
      "ItemsPath": "$.phase1.aggregation.candidates",
      "ItemSelector": {
        "fileVersionId.$": "$.fileVersionId",
        "folderId.$": "$.folderId",
        "fileId.$": "$.fileId",
        "normalizedMarkdownKey.$": "$.phase0.normalizedMarkdownKey",
        "claimsNormalizedKey.$": "$.phase1.aggregation.claimsNormalizedKey",
        "candidate.$": "$$.Map.Item.Value"
      },
      "MaxConcurrency": 3,
      "ResultPath": "$.phase2.synthesizedPages",
      "Iterator": {
        "StartAt": "SynthesizePage",
        "States": {
          "SynthesizePage": {
            "Type": "Task",
            "Resource": "${SynthesizePageFunctionArn}",
            "Retry": [
              {
                "ErrorEquals": ["RateLimitError"],
                "IntervalSeconds": 30,
                "MaxAttempts": 5,
                "BackoffRate": 1.5
              }
            ],
            "End": true
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ],
      "Next": "QueueWikiMaintenance"
    },
    "QueueWikiMaintenance": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "${WikiMaintenanceQueueUrl}",
        "MessageBody": {
          "requestType": "WikiMaintenance",
          "schemaVersion": 1,
          "fileVersionId.$": "$.fileVersionId",
          "fileId.$": "$.fileId",
          "folderId.$": "$.folderId",
          "normalizedMarkdownKey.$": "$.phase0.normalizedMarkdownKey",
          "claimsNormalizedKey.$": "$.phase1.aggregation.claimsNormalizedKey",
          "synthesizedPages.$": "$.phase2.synthesizedPages",
          "taskToken.$": "$$.Task.Token",
          "timestamp.$": "$$.State.EnteredTime"
        },
        "MessageGroupId.$": "$.folderId",
        "MessageDeduplicationId.$": "States.Format('{}:{}', $.fileVersionId, $$.State.EnteredTime)"
      },
      "TimeoutSeconds": 600,
      "ResultPath": "$.phase3",
      "Retry": [
        {
          "ErrorEquals": ["RetryableError"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Next": "MarkSucceeded",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "MarkFailed"
        }
      ]
    },
    "MarkSucceeded": {
      "Type": "Task",
      "Resource": "${MarkJobStatusFunctionArn}",
      "Parameters": {
        "fileVersionId.$": "$.fileVersionId",
        "status": "succeeded"
      },
      "End": true
    },
    "MarkFailed": {
      "Type": "Task",
      "Resource": "${MarkJobStatusFunctionArn}",
      "Parameters": {
        "fileVersionId.$": "$.fileVersionId",
        "status": "failed",
        "error.$": "$.error"
      },
      "End": true
    }
  }
}
```

**Template Substitution**: The state machine definition uses `${...}` placeholders for Lambda ARNs
and queue URLs. These are substituted at deploy time via CloudFormation `DefinitionSubstitutions`:

```yaml
ProcessingStateMachine:
  Type: AWS::StepFunctions::StateMachine
  Properties:
    StateMachineName: !Sub processing-workflow-${Environment}
    DefinitionS3Location:
      Bucket:
        Fn::ImportValue:
          Fn::Sub: "processing-foundation-${Environment}-ArtifactsBucket"
      Key: !Sub sfn/${ArtifactVersion}/workflow.asl.json
    DefinitionSubstitutions:
      ValidateAndEnrichFunctionArn: !GetAtt ValidateAndEnrichFunction.Arn
      Phase0NormalizeFunctionArn: !GetAtt Phase0NormalizeFunction.Arn
      ExtractChunkFunctionArn: !GetAtt ExtractChunkFunction.Arn
      AggregateClaimsFunctionArn: !GetAtt AggregateClaimsFunction.Arn
      SynthesizePageFunctionArn: !GetAtt SynthesizePageFunction.Arn
      MarkJobStatusFunctionArn: !GetAtt MarkJobStatusFunction.Arn
      WikiMaintenanceQueueUrl: !Ref WikiMaintenanceQueue
    RoleArn: !GetAtt StepFunctionsRole.Arn
```

**ASL Validation**: The state machine uses `States.Format` intrinsic function for `MessageDeduplicationId`,
which has strict syntax requirements. Validate the ASL definition before deployment:

```bash
aws stepfunctions validate-state-machine-definition \
  --definition "$(cat workflow.asl.json)" \
  --type STANDARD
```

CI should run this validation on every change to the ASL definition.

**Data Flow Summary**:

| State                | Input Reads                                | Output (ResultPath)                                       |
| -------------------- | ------------------------------------------ | --------------------------------------------------------- |
| ValidateAndEnrich    | `$.fileVersionId`, `$.sourceObjectKey`     | `$.validation`                                            |
| Phase0Normalize      | `$.fileVersionId`, `$.sourceObjectKey`     | `$.phase0.normalizedMarkdownKey`, `.chunks`               |
| Phase1ExtractClaims  | `$.phase0.chunks`, `$.fileVersionId`       | `$.phase1.chunkResults`                                   |
| AggregateClaims      | `$.phase1.chunkResults`, `$.fileVersionId` | `$.phase1.aggregation.candidates`, `.claimsNormalizedKey` |
| Phase2Synthesize     | `$.phase1.aggregation.candidates`          | `$.phase2.synthesizedPages`                               |
| QueueWikiMaintenance | Full context + `$$.Task.Token`             | `$.phase3`                                                |

```

---

## 3. Data Models

### 3.1 DynamoDB Tables

#### ProcessingJobs Table

Primary workflow tracking. One item per FileVersion.

```

Table: processing-jobs-{env}
Partition Key: fileVersionId (String)

Attributes:
fileVersionId: String # PK, workflow idempotency key
fileId: String # Zarya file identity
folderId: String # Wiki target, Phase 3 serialization key
sourceObjectKey: String # Zarya committed object key
sourceSizeBytes: Number # For size-based routing
sourceContentType: String # application/pdf or text/markdown
sourceEtagOrChecksum: String
status: String # queued | running | blocked | partial | succeeded | failed | canceled
currentPhase: String # phase0 | phase1 | phase2 | phase3 | complete
stepFunctionExecutionArn: String
attempts: Number
createdAt: String # ISO timestamp
updatedAt: String # ISO timestamp
completedAt: String # ISO timestamp (when succeeded/failed)
errorCode: String # Sanitized error code
errorMessage: String # Sanitized error message
artifactPaths: Map # Pointers to S3 artifacts
normalizedMarkdown: String
claimsRaw: String
claimsNormalized: String
synthesizedPages: List<String>

GSI: byFolder
Partition Key: folderId
Sort Key: createdAt
Projection: ALL

```

#### ProcessingTasks Table

Per-phase task tracking for debugging and resume.

```

Table: processing-tasks-{env}
Partition Key: taskId (String) # {fileVersionId}#{phase}#{slug}

Attributes:
taskId: String # PK
fileVersionId: String # Parent workflow
phase: String # normalizeDocument | extractClaims | synthesize | maintainWiki
slug: String # For phase1/phase2, null otherwise
chunkIndex: Number # For phase1 chunks
status: String # queued | running | succeeded | failed
attempts: Number
inputObjectKey: String
outputObjectKey: String
modelId: String # Bedrock model used
inputTokens: Number
outputTokens: Number
durationMs: Number
createdAt: String
updatedAt: String
errorCode: String
errorMessage: String

GSI: byFileVersion
Partition Key: fileVersionId
Sort Key: taskId
Projection: ALL

```

#### WikiLocks Table

Conditional lock for Phase 3 serialization.

```

Table: wiki-locks-{env}
Partition Key: folderId (String)

Attributes:
folderId: String # PK
lockedBy: String # fileVersionId holding the lock
lockedAt: String # ISO timestamp
ttl: Number # DynamoDB TTL epoch seconds (auto-release after 10 min)

```

### 3.2 S3 Object Layout

```

processing-{env}/
├── {fileVersionId}/
│ ├── phase-0/
│ │ ├── source.md # Normalized markdown
│ │ └── metadata.json # Page count, warnings, etc.
│ ├── phase-1/
│ │ ├── chunks.json # Chunk boundaries
│ │ ├── claims-raw.jsonl # Append-only raw claims
│ │ ├── claims-normalized.json # Deduped with stable IDs
│ │ └── candidates.json # Proposed pages
│ └── phase-2/
│ ├── {slug}.md # Synthesized page
│ └── {slug}.judge.json # Judge verdict

wikis-{env}/
├── {folderId}/
│ ├── schema.md # Wiki maintenance instructions
│ ├── index.md # Content catalog
│ ├── log.md # Append-only history
│ ├── \_graph.json # Machine-readable graph
│ ├── sources/
│ │ └── {fileId}-{fileVersionId}.md # Source summary page
│ ├── concepts/
│ │ └── {slug}.md
│ ├── entities/
│ │ └── {slug}.md
│ ├── references/
│ │ └── {slug}.md
│ ├── contradictions/
│ │ └── {slug}.md
│ └── answers/
│ └── {slug}.md

````

### 3.3 Event Schemas

#### FileVersionCommitted (Dispatcher → Intake Queue)

```typescript
interface FileVersionCommittedEvent {
  eventType: "FileVersionCommitted";
  schemaVersion: 1;
  environment: "dev" | "stage" | "prod";
  fileVersionId: string;
  fileId: string;
  folderId: string; // Resolved from File.parentFolderId
  versionNumber: number;
  objectKey: string;
  sizeBytes: number;
  contentType: string;
  etagOrChecksum: string;
  createdByUserId: string;
  createdAt: string; // ISO timestamp
}
````

#### WikiMaintenanceRequest (Phase 2 → FIFO Queue)

```typescript
interface WikiMaintenanceRequest {
  requestType: "WikiMaintenance";
  schemaVersion: 1;
  fileVersionId: string;
  fileId: string;
  folderId: string;
  normalizedMarkdownKey: string;
  claimsNormalizedKey: string;
  synthesizedPages: Array<{
    slug: string;
    objectKey: string;
    claimCount: number;
  }>;
  timestamp: string;
}

// Queue message envelope includes Step Functions callback token
interface WikiMaintenanceQueueMessage extends WikiMaintenanceRequest {
  taskToken: string; // Step Functions waitForTaskToken
}
```

---

## 4. Phase Implementations

### 4.1 Phase 0: Normalize Source

#### Compute Selection

| Source Size | Compute     | Memory  | Timeout | /tmp Storage |
| ----------- | ----------- | ------- | ------- | ------------ |
| < 100 MB    | Lambda      | 3008 MB | 5 min   | 512 MB       |
| 100-250 MB  | ECS Fargate | 8 GB    | 15 min  | 20 GB EFS    |

#### Implementation

```python
# Lambda handler pseudocode
def phase0_normalize(event, context):
    job = load_job(event["fileVersionId"])

    # Validate source still exists and matches
    s3_head = s3.head_object(Bucket=ZARYA_BUCKET, Key=job.sourceObjectKey)
    if s3_head["ETag"] != job.sourceEtagOrChecksum:
        raise SourceModifiedError()

    # Route large files to ECS (100MB threshold)
    if job.sourceSizeBytes > 100_000_000:
        return invoke_ecs_phase0(job)

    # Download source
    local_path = f"/tmp/{job.fileVersionId}/source.pdf"
    s3.download_file(ZARYA_BUCKET, job.sourceObjectKey, local_path)

    # Convert with pymupdf4llm
    import pymupdf4llm
    markdown = pymupdf4llm.to_markdown(local_path)
    markdown = clean_pdf_artifacts(markdown)

    # Upload normalized
    output_key = f"{job.fileVersionId}/phase-0/source.md"
    s3.put_object(Bucket=PROCESSING_BUCKET, Key=output_key, Body=markdown)

    # Update job
    update_job(job.fileVersionId, {
        "currentPhase": "phase1",
        "artifactPaths.normalizedMarkdown": output_key,
    })

    # Return chunks for Phase 1 Map state
    chunks = create_structured_chunks(markdown, target_tokens=6500)
    return {
        "normalizedMarkdownKey": output_key,
        "chunks": [c.to_dict() for c in chunks],
    }
```

### 4.2 Phase 1: Extract Claims

#### Per-Chunk Worker

```python
def phase1_extract_chunk(event, context):
    job_id = event["fileVersionId"]
    chunk = Chunk.from_dict(event["chunk"])

    # Load normalized source
    markdown = load_s3_text(PROCESSING_BUCKET, f"{job_id}/phase-0/source.md")
    chunk_text = extract_lines(markdown, chunk.start_line, chunk.end_line)

    # Rate limit check (token bucket in DynamoDB)
    acquire_rate_limit(model_id=BEDROCK_MODEL, tokens=estimate_tokens(chunk_text))

    # Call Bedrock
    claims = extract_claims_from_chunk(
        chunk_text=chunk_text,
        chunk_index=chunk.index,
        model_id=BEDROCK_MODEL,
    )

    # Write per-chunk output (avoids S3 append race condition)
    # Parallel workers write to distinct keys; aggregator collects them
    save_s3_json(
        PROCESSING_BUCKET,
        f"{job_id}/phase-1/chunks/{chunk.index:04d}.json",
        {
            "chunkIndex": chunk.index,
            "claims": [c.to_dict() for c in claims],
            "inputTokens": claims.input_tokens,
            "outputTokens": claims.output_tokens,
        },
    )

    # Record task
    save_task(
        taskId=f"{job_id}#extractClaims#{chunk.index}",
        phase="extractClaims",
        chunkIndex=chunk.index,
        inputTokens=claims.input_tokens,
        outputTokens=claims.output_tokens,
    )

    return {"chunkIndex": chunk.index, "claimCount": len(claims)}
```

#### Claim Aggregation

```python
def aggregate_claims(event, context):
    job_id = event["fileVersionId"]
    # State machine passes full state; chunk results are at $.phase1.chunkResults
    chunk_results = event["phase1"]["chunkResults"]
    chunk_count = len(chunk_results)

    # Collect per-chunk files in deterministic order
    raw_claims = []
    for i in range(chunk_count):
        chunk_data = load_s3_json(
            PROCESSING_BUCKET,
            f"{job_id}/phase-1/chunks/{i:04d}.json"
        )
        raw_claims.extend(chunk_data["claims"])

    # Deduplicate and normalize
    normalized = deduplicate_claims(raw_claims, slug=job_id)

    # Group by topic
    topics = aggregate_by_topic(normalized)

    # Generate candidates (pages to synthesize)
    candidates = generate_candidates(topics, min_claims_per_topic=3)

    # Save
    claims_key = f"{job_id}/phase-1/claims-normalized.json"
    save_s3_json(PROCESSING_BUCKET, claims_key, normalized)
    save_s3_json(PROCESSING_BUCKET, f"{job_id}/phase-1/candidates.json", candidates)

    # Update job
    update_job(job_id, {
        "artifactPaths.claimsNormalized": claims_key,
    })

    return {
        "candidates": candidates,
        "claimsNormalizedKey": claims_key,
    }
```

### 4.3 Phase 2: Synthesize Pages

```python
def phase2_synthesize_page(event, context):
    job_id = event["fileVersionId"]
    candidate = event["candidate"]  # {topic, slug, claims, page_type}

    # Load context
    normalized_markdown = load_s3_text(PROCESSING_BUCKET, f"{job_id}/phase-0/source.md")
    claims = load_s3_json(PROCESSING_BUCKET, f"{job_id}/phase-1/claims-normalized.json")

    # Select claims for this page
    page_claims = select_claims_for_topic(claims, candidate["topic"])

    # Build evidence bank
    evidence_bank = build_evidence_bank(page_claims, normalized_markdown)

    # Rate limit
    acquire_rate_limit(model_id=BEDROCK_MODEL, tokens=estimate_synthesis_tokens(evidence_bank))

    # Synthesize
    page_content = synthesize_page(
        topic=candidate["topic"],
        page_type=candidate["page_type"],
        evidence_bank=evidence_bank,
        model_id=BEDROCK_MODEL,
    )

    # Validate
    validation_result = validate_synthesized_page(page_content)
    if not validation_result.passed:
        page_content = repair_page(page_content, validation_result.issues)

    # Judge claims (optional quality gate)
    judge_result = judge_claims(page_content, evidence_bank)

    # Save
    output_key = f"{job_id}/phase-2/{candidate['slug']}.md"
    save_s3_text(PROCESSING_BUCKET, output_key, page_content)
    save_s3_json(PROCESSING_BUCKET, f"{job_id}/phase-2/{candidate['slug']}.judge.json", judge_result)

    return {
        "slug": candidate["slug"],
        "objectKey": output_key,
        "claimCount": len(page_claims),
        "judgeVerdict": judge_result.verdict,
    }
```

### 4.4 Phase 3: Wiki Maintenance

**SQS Deletion Model**: Lambda event source mapping handles message deletion when the handler
returns successfully (no exception raised). This applies to all paths:

- **Success**: Send callback, return normally → Lambda deletes message
- **Handled failure**: Send failure callback, return normally → Lambda deletes message
- **Unhandled exception**: Handler raises → Lambda does NOT delete, message returns to queue

This unified model avoids the risk of `delete_message` failing after callback is sent. Callbacks
are idempotent in Step Functions (duplicate callbacks to a closed task token return error but are
harmless). The handler never manually deletes messages.

**Idempotency Guarantee**: Phase 3 is idempotent by `fileVersionId`. Each wiki mutation uses
upsert semantics keyed by `fileVersionId`:

- Source page: `sources/{fileId}-{fileVersionId}.md` — deterministic path, upsert
- Page merges: `merge_wiki_pages` produces identical output for same inputs
- Log entries: contain `fileVersionId`, deduped on append
- Graph/index: deterministic rebuild from current wiki state

**Atomic Save Protocol**: `wiki.save()` writes to a staging prefix, then atomically updates a
manifest object (`wiki-manifest.json`) that points to the current version. Readers always follow
the manifest. If save is interrupted mid-write, the manifest still points to the previous complete
version. On retry, the worker writes a fresh staging version and updates the manifest.

Retryable errors (`WikiLockContention`, `S3TransientError`) only occur before `wiki.save()` commits.
If an unexpected error occurs mid-mutation, the manifest is not updated; retry starts from the last
committed wiki state.

```python
def phase3_maintain_wiki(event, context):
    """Process wiki maintenance from FIFO queue with Step Functions callback.

    Message deletion is Lambda-managed: return normally to delete, raise to keep.
    All handled paths (success or failure) return normally after sending callback.
    Callback failures after wiki commit raise to trigger Lambda retry (idempotent replay).
    """
    record = json.loads(event["Records"][0]["body"])
    request = WikiMaintenanceRequest.from_dict(record)
    task_token = record.get("taskToken")  # Step Functions callback token
    lock_acquired = False  # Track lock ownership for safe release
    wiki_committed = False  # Track whether wiki was committed

    try:
        # Acquire folder lock (defense-in-depth; SQS FIFO is primary serialization)
        lock_acquired = acquire_wiki_lock(request.folderId, request.fileVersionId, ttl_seconds=600)
        if not lock_acquired:
            raise WikiLockContention(f"Folder {request.folderId} locked")

        wiki = load_wiki(request.folderId)

        # 1. Create/update source summary page
        source_page = create_source_page(
            file_id=request.fileId,
            file_version_id=request.fileVersionId,
            normalized_markdown_key=request.normalizedMarkdownKey,
            claims_key=request.claimsNormalizedKey,
        )
        wiki.upsert_page(f"sources/{request.fileId}-{request.fileVersionId}.md", source_page)

        # 2. Merge synthesized pages
        for page_info in request.synthesizedPages:
            page_content = load_s3_text(PROCESSING_BUCKET, page_info["objectKey"])
            existing = wiki.get_page(page_info["slug"])

            if existing:
                # Check for contradictions
                contradictions = detect_contradictions(existing, page_content)
                if contradictions:
                    wiki.create_or_update_contradiction(contradictions)
                # Merge claims (newer version wins on conflicts)
                merged = merge_wiki_pages(existing, page_content, prefer_newer=True)
                wiki.upsert_page(page_info["slug"], merged)
            else:
                wiki.create_page(page_info["slug"], page_content)

        # 3. Update index
        wiki.rebuild_index()

        # 4. Update graph
        wiki.rebuild_graph()

        # 5. Append to log
        wiki.append_log(
            operation="ingest",
            file_version_id=request.fileVersionId,
            pages_created=len([p for p in request.synthesizedPages if not wiki.page_existed(p["slug"])]),
            pages_updated=len([p for p in request.synthesizedPages if wiki.page_existed(p["slug"])]),
        )

        # 6. Save wiki to S3 (atomic via manifest)
        wiki.save()
        wiki_committed = True

    except TerminalError as e:
        # Terminal errors: send failure callback, return normally (Lambda deletes message)
        # Examples: SourceDeleted, InvalidClaims, SchemaViolation
        if task_token:
            sfn_client.send_task_failure(
                taskToken=task_token,
                error=type(e).__name__,
                cause=str(e)[:256],
            )
        # Return normally so Lambda deletes the message
        return

    except RetryableError as e:
        # Retryable errors: signal to Step Functions for workflow-level retry
        # Examples: WikiLockContention, S3TransientError, ThrottlingException
        logger.warning(f"Retryable error, signaling Step Functions: {e}")
        if task_token:
            sfn_client.send_task_failure(
                taskToken=task_token,
                error="RetryableError",
                cause=str(e)[:256],
            )
        update_job(request.fileVersionId, {
            "status": "blocked",
            "currentPhase": "phase3",
            "blockReason": str(e),
        })
        # Return normally so Lambda deletes the message; Step Functions will retry
        return

    except Exception as e:
        # Unexpected errors during wiki maintenance: fail the task
        # But NOT if wiki already committed—post-commit errors are handled in success path
        if wiki_committed:
            # Wiki is committed; skip failure callback, fall through to success path
            logger.warning(f"Post-commit error (wiki saved): {e}")
        else:
            if task_token:
                sfn_client.send_task_failure(
                    taskToken=task_token,
                    error="UnexpectedError",
                    cause=str(e)[:256],
                )
            update_job(request.fileVersionId, {
                "status": "failed",
                "currentPhase": "phase3",
                "errorCode": "UnexpectedError",
                "errorMessage": str(e)[:256],
            })
            # Return normally so Lambda deletes the message
            return

    finally:
        # Only release lock if we acquired it (conditional on lockedBy == fileVersionId)
        # Best-effort: don't let release failure affect the handler result
        if lock_acquired:
            try:
                release_wiki_lock(request.folderId, request.fileVersionId)
            except Exception as release_err:
                logger.warning(f"Lock release failed (best-effort): {release_err}")

    # SUCCESS PATH: Wiki committed
    # Update job and send callback OUTSIDE try block
    # If these fail, Lambda retries; Phase 3 idempotency ensures replay is safe
    try:
        update_job(request.fileVersionId, {
            "status": "succeeded",
            "currentPhase": "complete",
            "completedAt": datetime.utcnow().isoformat(),
        })
    except Exception as job_err:
        # Best-effort: wiki is committed, so job status is stale but not critical
        logger.warning(f"Job status update failed (wiki committed): {job_err}")

    if task_token:
        try:
            sfn_client.send_task_success(
                taskToken=task_token,
                output=json.dumps({
                    "fileVersionId": request.fileVersionId,
                    "folderId": request.folderId,
                    "pagesCreated": len(request.synthesizedPages),
                }),
            )
        except sfn_client.exceptions.TaskTimedOut:
            # Task token expired—Step Functions already timed out; nothing to do
            logger.warning("Task token timed out; Step Functions already failed")
        except sfn_client.exceptions.InvalidToken:
            # Task token already used (retry after success)—this is expected on replay
            logger.info("Task token already closed (expected on idempotent replay)")
```

#### FIFO Queue + Lock Semantics

**Why both?**

- **SQS FIFO with MessageGroupId=folderId**: Authoritative ordering. Ensures one message per folder processes at a time. This is the primary serialization mechanism.
- **WikiLocks DynamoDB table**: Defense-in-depth. Handles edge cases where SQS visibility timeout expires before Lambda completes (e.g., long-running merge). TTL-based with conditional writes.

**Phase 3 State Model**:

Phase 3 uses `waitForTaskToken` to synchronize with Step Functions. The callback contract is strict:
every handled path sends exactly one callback (success or failure) and returns normally, allowing
Lambda to delete the message. This avoids split-brain between Step Functions and SQS.

**Callback Contract**:

| Error Type       | Example                      | Worker Action                                           | Step Functions Result |
| ---------------- | ---------------------------- | ------------------------------------------------------- | --------------------- |
| Success          | —                            | commit wiki, `send_task_success` (catches closed token) | Succeeded             |
| Post-commit fail | update_job/callback error    | catch + log; idempotent replay on retry                 | Succeeded (on retry)  |
| Terminal         | SourceDeleted, InvalidClaims | `send_task_failure`, return normally (Lambda deletes)   | Failed                |
| Retryable        | WikiLockContention, Throttle | `send_task_failure` + `RetryableError`, return          | Failed (retry via SF) |
| Unexpected       | pre-commit NullPointerExc    | `send_task_failure`, return normally (Lambda deletes)   | Failed                |

**Key invariant**: Wiki errors are caught and handled (callback + return). The success path runs
OUTSIDE the try block: wiki is committed, then job status and callback are best-effort with explicit
error handling. `InvalidToken` and `TaskTimedOut` exceptions from Step Functions are caught and
logged, allowing Lambda to delete the message normally. On retry, Phase 3 idempotency ensures the
wiki is already committed; the callback may succeed or be a harmless duplicate.

**FIFO deduplication**: The `MessageDeduplicationId` includes `$$.State.EnteredTime` so that
Step Functions retries send distinct messages. Without this, FIFO's 5-minute dedupe window would
silently drop retried sends, leaving the workflow waiting on a task token no worker receives.

**Retryable error handling**:

```python
except RetryableError as e:
    # Signal retryable failure to Step Functions
    # Workflow can retry the entire QueueWikiMaintenance state
    if task_token:
        sfn_client.send_task_failure(
            taskToken=task_token,
            error="RetryableError",
            cause=str(e)[:256],
        )
    # Mark job as blocked; successful retry will overwrite with succeeded/complete
    update_job(request.fileVersionId, {
        "status": "blocked",
        "currentPhase": "phase3",
        "blockReason": str(e),
    })
    # Return normally so Lambda deletes the message; Step Functions will retry
    return
```

**Retry behavior**: The `Retry` clause in `QueueWikiMaintenance` (see authoritative ASL in §2.3)
handles `RetryableError` with 30s/60s/120s intervals and max 3 attempts. Each retry sends a new
SQS message with a distinct `MessageDeduplicationId` (includes `$$.State.EnteredTime`), avoiding
FIFO's 5-minute dedupe window. A successful retry overwrites the job's `blocked` status with
`succeeded`/`complete`.

This keeps the state model clean: Step Functions is authoritative for workflow status, SQS FIFO
provides ordering, and the job table reflects the current state. No split-brain.

---

## 5. Concurrency and Rate Limiting

### 5.1 Bedrock Rate Limits

Default model: **Qwen3-Coder 30b** (`qwen.qwen3-coder-30b-a3b-v1:0`)

| Model                         | Requests/min | Tokens/min | Context Window | Recommended Concurrency |
| ----------------------------- | ------------ | ---------- | -------------- | ----------------------- |
| **qwen3-coder-30b** (default) | 60           | 100,000    | 32,768         | 5-10                    |
| claude-3-5-sonnet (fallback)  | 100          | 200,000    | 200,000        | 10                      |
| claude-3-haiku (high-volume)  | 1000         | 400,000    | 200,000        | 50                      |

Model selection is config-driven via Secrets Manager. To switch models:

1. Update `BEDROCK_MODEL_ID` in Secrets Manager
2. Adjust rate limiter token bucket capacity for the new model
3. No code deployment required

**Qwen3-Coder 30b constraints:**

- 32K context window (8-12K practical input budget, 4-8K output reserve)
- `max_tokens` = 8192 (output only)
- Optimized for code and structured extraction tasks

### 5.2 Token Bucket Implementation

```python
# DynamoDB-backed token bucket
Table: rate-limits-{env}
Partition Key: modelId (String)

Attributes:
  modelId: String
  tokens: Number              # Current available tokens
  lastRefill: Number          # Epoch ms of last refill
  refillRate: Number          # Tokens per second
  maxTokens: Number           # Bucket capacity

def acquire_rate_limit(model_id: str, tokens: int, timeout_ms: int = 30000) -> bool:
    """Attempt to acquire tokens, blocking until available or timeout."""
    deadline = time.time() + timeout_ms / 1000

    while time.time() < deadline:
        # Atomic conditional update
        try:
            dynamodb.update_item(
                TableName=RATE_LIMITS_TABLE,
                Key={"modelId": model_id},
                UpdateExpression="SET tokens = tokens - :t, lastRefill = :now",
                ConditionExpression="tokens >= :t",
                ExpressionAttributeValues={
                    ":t": tokens,
                    ":now": int(time.time() * 1000),
                },
            )
            return True
        except ConditionalCheckFailedException:
            # Not enough tokens - refill and retry
            refill_bucket(model_id)
            time.sleep(1)

    raise RateLimitError(f"Could not acquire {tokens} tokens for {model_id}")
```

### 5.3 Concurrency Controls

| Phase      | Control Mechanism                 | Limit          | Notes               |
| ---------- | --------------------------------- | -------------- | ------------------- |
| Phase 0    | Lambda reserved concurrency       | 10             | CPU/memory bound    |
| Phase 1    | Step Functions Map MaxConcurrency | 5 per workflow | + global rate limit |
| Phase 2    | Step Functions Map MaxConcurrency | 3 per workflow | + global rate limit |
| Phase 3    | SQS FIFO MessageGroupId           | 1 per folder   | Serialized          |
| Global LLM | Token bucket                      | Model-specific | Cross-workflow      |

### 5.4 Backpressure Handling

When 60 PDFs arrive simultaneously:

1. **Intake Queue** buffers all 60 FileVersionCommitted events
2. **Step Functions** starts workflows with Lambda concurrency limits
3. **Phase 0** runs 10 concurrent normalizations
4. **Phase 1/2** workers check token bucket before each LLM call
5. **Rate limit exhaustion** causes exponential backoff retries
6. **Phase 3 FIFO** serializes wiki writes per folder

Expected timeline for 60 PDFs (same folder):

```
T+0:00   60 events received
T+0:05   10 Phase 0 normalizations complete
T+0:30   All 60 normalizations complete
T+1:00   ~20 Phase 1 extractions complete (rate limited)
T+2:00   All Phase 1 complete
T+2:30   ~30 Phase 2 syntheses complete
T+3:30   All Phase 2 complete
T+4:00   All Phase 3 wiki maintenance complete (serialized)
```

---

## 6. AWS Infrastructure

### 6.1 CloudFormation Stack Structure

```
processing-service-{env}/
├── foundation/
│   ├── VPC (if needed for ECS)
│   ├── S3 Buckets (processing, wikis)
│   └── DynamoDB Tables (jobs, tasks, locks, rate-limits)
├── compute/
│   ├── Lambda Functions
│   │   ├── stream-dispatcher
│   │   ├── phase0-normalize
│   │   ├── phase0-normalize-large (ECS task trigger)
│   │   ├── phase1-extract-chunk
│   │   ├── phase1-aggregate-claims
│   │   ├── phase2-synthesize-page
│   │   ├── phase3-maintain-wiki
│   │   ├── mark-job-status
│   │   └── rate-limit-refill
│   └── ECS Fargate (large PDF processing)
├── orchestration/
│   ├── Step Functions State Machine
│   ├── SQS Queues
│   │   ├── processing-intake
│   │   ├── wiki-maintenance.fifo
│   │   └── DLQs
│   └── Lambda Event Source Mapping (DynamoDB Stream → Dispatcher)
└── api/
    ├── API Gateway
    └── Lambda (API handlers)
```

### 6.2 IAM Roles

#### Stream Dispatcher Role

```yaml
Policies:
  # DynamoDB Stream permissions (required for Lambda event source mapping)
  - Effect: Allow
    Action:
      - dynamodb:DescribeStream
      - dynamodb:GetRecords
      - dynamodb:GetShardIterator
      - dynamodb:ListStreams
    Resource:
      - arn:aws:dynamodb:*:*:table/zarya-{env}-file-versions/stream/*
  # Table read permissions for enrichment
  - Effect: Allow
    Action:
      - dynamodb:GetItem
    Resource:
      - arn:aws:dynamodb:*:*:table/zarya-{env}-file-versions
      - arn:aws:dynamodb:*:*:table/zarya-{env}-files
  - Effect: Allow
    Action:
      - sqs:SendMessage
    Resource:
      - arn:aws:sqs:*:*:processing-intake-{env}
```

#### Phase 0-2 Worker Role

```yaml
Policies:
  - Effect: Allow
    Action:
      - s3:GetObject
    Resource:
      - arn:aws:s3:::zarya-files-{env}/objects/*
  - Effect: Allow
    Action:
      - s3:GetObject
      - s3:PutObject
    Resource:
      - arn:aws:s3:::processing-{env}/*
  - Effect: Allow
    Action:
      - bedrock:InvokeModel
    Resource:
      - arn:aws:bedrock:*:*:foundation-model/*
  - Effect: Allow
    Action:
      - dynamodb:GetItem
      - dynamodb:PutItem
      - dynamodb:UpdateItem
    Resource:
      - arn:aws:dynamodb:*:*:table/processing-jobs-{env}
      - arn:aws:dynamodb:*:*:table/processing-tasks-{env}
      - arn:aws:dynamodb:*:*:table/rate-limits-{env}
```

#### Phase 3 Worker Role

```yaml
# All of Phase 0-2 permissions, plus:
# SQS permissions (Lambda event source mapping uses execution role for polling)
- Effect: Allow
  Action:
    - sqs:ReceiveMessage
    - sqs:DeleteMessage
    - sqs:GetQueueAttributes
    - sqs:ChangeMessageVisibility
  Resource:
    - arn:aws:sqs:*:*:wiki-maintenance-{env}.fifo
- Effect: Allow
  Action:
    - s3:GetObject
    - s3:PutObject
    # Note: No DeleteObject on wikis. Wiki pages are retained indefinitely.
    # Page "removal" is soft-delete via status marker in frontmatter.
  Resource:
    - arn:aws:s3:::wikis-{env}/*
- Effect: Allow
  Action:
    - dynamodb:GetItem
    - dynamodb:PutItem
    - dynamodb:DeleteItem
  Resource:
    - arn:aws:dynamodb:*:*:table/wiki-locks-{env}
# Step Functions callback permission
- Effect: Allow
  Action:
    - states:SendTaskSuccess
    - states:SendTaskFailure
  Resource: "*"
```

**Retention Policy**:

| Bucket           | Retention                       | Lifecycle Rules                          |
| ---------------- | ------------------------------- | ---------------------------------------- |
| ProcessingBucket | Intermediate artifacts          | Noncurrent versions expire after 90 days |
| WikisBucket      | **Indefinite** (no auto-delete) | None                                     |

Wiki pages are never deleted. To "remove" a page, set `status: archived` in frontmatter. This preserves audit history and allows recovery.

### 6.3 Lambda Configuration

| Function                | Memory  | Timeout | Concurrency | Layers      |
| ----------------------- | ------- | ------- | ----------- | ----------- |
| stream-dispatcher       | 256 MB  | 30s     | 10          | -           |
| phase0-normalize        | 3008 MB | 5 min   | 10          | pymupdf4llm |
| phase1-extract-chunk    | 1024 MB | 2 min   | 50          | wiki-llm    |
| phase1-aggregate-claims | 1024 MB | 1 min   | 20          | wiki-core   |
| phase2-synthesize-page  | 1024 MB | 3 min   | 30          | wiki-llm    |
| phase3-maintain-wiki    | 1024 MB | 5 min   | 10          | wiki-core   |

### 6.4 Runtime Packaging

The repo produces multiple deployment artifacts for different compute targets:

#### Artifact Structure

```
dist/
├── lambda/
│   ├── node/
│   │   └── handlers.zip          # TypeScript handlers (API, dispatcher)
│   └── python/
│       ├── phase0.zip            # PDF extraction (pymupdf4llm)
│       ├── phase1-extract.zip    # Claim extraction (wiki_llm)
│       ├── phase1-aggregate.zip  # Aggregation (wiki_core)
│       ├── phase2.zip            # Synthesis (wiki_llm)
│       └── phase3.zip            # Wiki maintenance (wiki_core, wiki_io)
├── layers/
│   ├── pymupdf4llm-layer.zip     # PDF extraction deps
│   ├── wiki-core-layer.zip       # wiki_core package
│   └── wiki-llm-layer.zip        # wiki_llm + Bedrock deps
├── step-functions/
│   └── processing-workflow.asl.json
└── ecs/
    └── phase0-large/
        └── Dockerfile            # For PDFs > 50MB
```

#### Build Commands

```bash
# Build all artifacts
pnpm build:lambda

# Individual builds
pnpm build:lambda:node       # TypeScript handlers
pnpm build:lambda:python     # Python handlers
pnpm build:layers            # Lambda layers
pnpm build:ecs               # Docker image for large PDFs
```

#### Layer Dependencies

| Layer       | Python Packages           | Size  |
| ----------- | ------------------------- | ----- |
| pymupdf4llm | pymupdf, pymupdf4llm      | ~50MB |
| wiki-core   | wiki_core (editable)      | ~5MB  |
| wiki-llm    | wiki_llm, boto3, tiktoken | ~30MB |

#### Upload Strategy

Artifacts are uploaded to the deployment artifacts bucket before stack deployment:

```bash
# Upload lambda artifacts
aws s3 cp dist/lambda/node/handlers.zip s3://${ARTIFACTS_BUCKET}/lambda/${VERSION}/handlers.zip
aws s3 cp dist/lambda/python/phase0.zip s3://${ARTIFACTS_BUCKET}/lambda/${VERSION}/phase0.zip
# ... etc

# Upload layers
aws s3 cp dist/layers/wiki-llm-layer.zip s3://${ARTIFACTS_BUCKET}/layers/${VERSION}/wiki-llm.zip

# Upload Step Functions definition
aws s3 cp dist/step-functions/processing-workflow.asl.json s3://${ARTIFACTS_BUCKET}/sfn/${VERSION}/workflow.json
```

The app CloudFormation stack references these via parameters:

```yaml
Parameters:
  ArtifactVersion:
    Type: String
    Description: Version prefix for artifacts in S3 (usually git SHA)

Resources:
  Phase1ExtractFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        S3Bucket: !Ref ArtifactsBucket
        S3Key: !Sub lambda/${ArtifactVersion}/phase1-extract.zip
      Layers:
        - !Ref WikiLlmLayer
```

---

## 7. API Design

### 7.1 Endpoints

| Method | Path                             | Purpose            |
| ------ | -------------------------------- | ------------------ |
| GET    | /processing/jobs/{fileVersionId} | Job status         |
| GET    | /processing/jobs?folderId={id}   | Jobs for folder    |
| GET    | /wikis/{folderId}                | Wiki summary       |
| GET    | /wikis/{folderId}/index          | Index page         |
| GET    | /wikis/{folderId}/log            | Recent log entries |
| GET    | /wikis/{folderId}/pages          | Page catalog       |
| GET    | /wikis/{folderId}/pages/{pageId} | Single page        |
| GET    | /wikis/{folderId}/contradictions | Contradiction list |

### 7.2 Response DTOs

```typescript
interface ProcessingJobResponse {
  fileVersionId: string; // Primary trace ID
  fileId: string; // Stable file identity
  folderId: string; // Wiki aggregate key
  status:
    | "queued"
    | "running"
    | "blocked"
    | "partial"
    | "succeeded"
    | "failed"
    | "canceled";
  currentPhase: "phase0" | "phase1" | "phase2" | "phase3" | "complete";
  progress: {
    phase0: { status: string; durationMs?: number };
    phase1: {
      status: string;
      chunksTotal: number;
      chunksComplete: number;
      durationMs?: number;
    };
    phase2: {
      status: string;
      pagesTotal: number;
      pagesComplete: number;
      durationMs?: number;
    };
    phase3: { status: string; durationMs?: number };
  };
  createdAt: string;
  updatedAt: string;
  completedAt?: string;
  error?: { code: string; message: string };
}

interface WikiSummaryResponse {
  folderId: string; // Wiki aggregate key
  wikiStatus: "empty" | "ready" | "updating" | "degraded";
  latestUpdatedAt: string;
  stats: {
    sourceCount: number;
    conceptCount: number;
    entityCount: number;
    referenceCount: number;
    contradictionCount: number;
  };
  // Trace-aware status fields
  activeFileVersionIds: string[]; // Currently processing
  failedFileVersionIds: string[]; // Failed, may need retry
  lastSuccessfulMaintenanceTraceId: string | null; // Last successful ingest
  recentIngests: Array<{
    fileVersionId: string; // Trace ID for this ingest
    fileId: string;
    status: "succeeded" | "failed" | "partial";
    completedAt: string;
    pagesCreated: number;
    pagesUpdated: number;
  }>;
}

interface WikiPageResponse {
  pageId: string;
  path: string;
  title: string;
  category:
    | "source"
    | "concept"
    | "entity"
    | "reference"
    | "contradiction"
    | "answer";
  markdown: string;
  sourceFileVersionIds: string[]; // Trace IDs of sources cited
  outboundLinks: string[];
  updatedAt: string;
}
```

### 7.3 Authorization

All API endpoints require authorization. The processing service does not define its own user model;
it delegates authorization to Zarya.

#### Authorization Flow

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Client    │──JWT───▶│  API GW     │──JWT───▶│  Lambda     │
│  (Zarya FE) │         │  Authorizer │         │  Handler    │
└─────────────┘         └─────────────┘         └─────────────┘
                               │                       │
                               ▼                       ▼
                        ┌─────────────┐         ┌─────────────┐
                        │  Cognito    │         │   Zarya     │
                        │  (verify)   │         │ Permissions │
                        └─────────────┘         └─────────────┘
```

#### Implementation

**Authentication vs Authorization:**

- **Authentication (local)**: API Gateway Lambda authorizer validates the Cognito JWT signature and expiration using the same Cognito User Pool as Zarya. This confirms the request has a valid user identity.
- **Authorization (Zarya)**: Handlers call Zarya's wiki authorization API to determine folder access. Do not derive folder permissions from local authorizer claims—Zarya is authoritative for access decisions.

```typescript
// API Gateway authorizer: authentication only
// Validates JWT signature/expiration, extracts claims for logging
// Does NOT determine folder access
interface AuthorizerContext {
  sub: string; // Cognito subject (for logging only)
  email: string; // For logging only
  exp: number; // Expiration
}

// Handler permission check: authorization via Zarya
// Forward the original JWT—Zarya resolves and authorizes the principal
async function assertWikiAccess(
  userJwt: string,
  folderId: string,
  access: "viewWikiSummary" | "readWikiContent",
): Promise<void> {
  const decision = await wikiAuthClient.checkAccess(userJwt, folderId, access);
  if (!decision.allowed) {
    throw new ForbiddenError(`Wiki access denied: ${decision.reason}`);
  }
}
```

#### Zarya Wiki Authorization API Contract

The processing service delegates all user authorization to Zarya via a narrow internal API.
Zarya owns resource identity, folder/file visibility, user identity, share grants, and authorization
rules. The processing service must not interpret Zarya grants directly.

**Full specification:** See `docs/INTERNAL-WIKI-AUTHORIZATION-API.md`.

##### API Surface

```http
POST /internal/wiki-authorization/check
Authorization: Bearer <Zarya Cognito access token>
X-Internal-Service: processing-service
X-Request-Id: <optional request id>
Content-Type: application/json
```

##### Request DTO

```typescript
type WikiAuthorizationAccess = "viewWikiSummary" | "readWikiContent";

type WikiAuthorizationCheckRequest = {
  checks: Array<{
    checkId: string; // Caller-defined, echoed in response
    folderId: string; // Zarya folder id
    access: WikiAuthorizationAccess;
  }>;
};
```

Constraints:

- `checks` must contain 1–30 items (matches Verified Permissions batch size)
- `folderId` is a Zarya folder id, not a wiki-specific id

##### Response DTO

```typescript
type WikiAuthorizationDecisionReason =
  | "ALLOWED"
  | "DENIED"
  | "FOLDER_NOT_FOUND"
  | "FOLDER_DELETED"
  | "USER_DISABLED";

type WikiAuthorizationCheckResponse = {
  checks: Array<{
    checkId: string;
    folderId: string;
    access: WikiAuthorizationAccess;
    allowed: boolean;
    reason: WikiAuthorizationDecisionReason;
    decisionTtlSeconds: number;
  }>;
};
```

##### Access Semantics

| Access            | Allows                                                       | Decision Rule                                             |
| ----------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| `viewWikiSummary` | Wiki status, counts, ingest statuses, page catalog (no body) | User can see folder in Zarya explorer                     |
| `readWikiContent` | Wiki page Markdown, index, log, contradictions, source pages | User can read all current live source files in the folder |

**MVP security invariant:** `readWikiContent` requires the user to be able to read all current live
source files in the folder (not just those "represented in the wiki"). This conservative rule avoids
leaking derived claims from a source the user cannot read. Page-level source filtering may be added
later if product requirements demand partial-folder access.

##### Processing Service Endpoint Mapping

| Processing Service Endpoint            | Required Access                                                      |
| -------------------------------------- | -------------------------------------------------------------------- |
| `GET /wikis/{folderId}`                | `viewWikiSummary`                                                    |
| `GET /processing/jobs?folderId={id}`   | `viewWikiSummary`                                                    |
| `GET /processing/jobs/{fileVersionId}` | `viewWikiSummary` (resolve folderId from ProcessingJobs table first) |
| `GET /wikis/{folderId}/index`          | `readWikiContent`                                                    |
| `GET /wikis/{folderId}/log`            | `readWikiContent`                                                    |
| `GET /wikis/{folderId}/pages`          | `viewWikiSummary` (metadata only: titles, types, timestamps)         |
| `GET /wikis/{folderId}/pages/{pageId}` | `readWikiContent`                                                    |
| `GET /wikis/{folderId}/contradictions` | `readWikiContent`                                                    |

##### Error Handling

Transport-level errors return HTTP status codes:

| Situation                   | HTTP  | API code           |
| --------------------------- | ----- | ------------------ |
| Missing/invalid JWT         | `401` | `UNAUTHENTICATED`  |
| Disabled current user       | `403` | `FORBIDDEN`        |
| Invalid request body        | `400` | `VALIDATION_ERROR` |
| Batch too large             | `400` | `VALIDATION_ERROR` |
| Unexpected internal failure | `500` | `INTERNAL_ERROR`   |

Per-check authorization failures return `200` with `allowed: false` and a `reason`.

##### Caching

- Cache key: `sha256(jwt):folderId:access` — hash the JWT to avoid trusting unvalidated claims
- Cache TTL: `decisionTtlSeconds` from response (matches Zarya's authorization decision cache TTL)
- Keep TTL short because folder sharing/revocation should take effect quickly

##### Implementation

```typescript
import { createHash } from "crypto";

class ZaryaWikiAuthorizationClient {
  private cache = new LRUCache<string, WikiAuthorizationDecision>({
    maxAge: 60_000,
  });

  async checkAccess(
    userJwt: string,
    folderId: string,
    access: WikiAuthorizationAccess,
  ): Promise<WikiAuthorizationDecision> {
    // Hash the JWT for cache key—do not extract/trust claims without validation
    const jwtHash = createHash("sha256")
      .update(userJwt)
      .digest("hex")
      .slice(0, 16);
    const cacheKey = `${jwtHash}:${folderId}:${access}`;
    const cached = this.cache.get(cacheKey);
    if (cached) return cached;

    const response = await fetch(
      `${ZARYA_API_BASE_URL}/internal/wiki-authorization/check`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${userJwt}`,
          "X-Internal-Service": "processing-service",
          "X-Request-Id": getTraceId(),
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          checks: [{ checkId: "1", folderId, access }],
        }),
        signal: AbortSignal.timeout(3000),
      },
    );

    if (!response.ok) {
      throw new ZaryaAuthorizationError(response.status, await response.text());
    }

    const envelope = await response.json();
    if (!envelope.ok) {
      throw new ZaryaAuthorizationError(
        envelope.error.code,
        envelope.error.message,
      );
    }

    const decision = envelope.data.checks[0];
    this.cache.set(cacheKey, decision, {
      ttl: decision.decisionTtlSeconds * 1000,
    });
    return decision;
  }
}

// Handler usage
async function assertWikiAccess(
  userJwt: string,
  folderId: string,
  access: WikiAuthorizationAccess,
): Promise<void> {
  const decision = await wikiAuthClient.checkAccess(userJwt, folderId, access);
  if (!decision.allowed) {
    throw new ForbiddenError(`Wiki access denied: ${decision.reason}`);
  }
}
```

#### Service-to-Service Auth

For internal calls (e.g., Step Functions calling APIs), use IAM auth:

```yaml
# API Gateway resource policy
Statement:
  - Effect: Allow
    Principal:
      AWS: !GetAtt ProcessingWorkflowRole.Arn
    Action: execute-api:Invoke
    Resource: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGateway}/*
```

#### Endpoint Auth Summary

| Endpoint                           | Auth Required | Permission Check                         |
| ---------------------------------- | ------------- | ---------------------------------------- |
| `/processing/jobs/{fileVersionId}` | User JWT      | Resolve folderId, then `viewWikiSummary` |
| `/processing/jobs?folderId={id}`   | User JWT      | `viewWikiSummary` on folderId            |
| `/wikis/{folderId}`                | User JWT      | `viewWikiSummary` on folderId            |
| `/wikis/{folderId}/index`          | User JWT      | `readWikiContent` on folderId            |
| `/wikis/{folderId}/log`            | User JWT      | `readWikiContent` on folderId            |
| `/wikis/{folderId}/pages`          | User JWT      | `viewWikiSummary` (metadata only)        |
| `/wikis/{folderId}/pages/{pageId}` | User JWT      | `readWikiContent` on folderId            |
| `/wikis/{folderId}/contradictions` | User JWT      | `readWikiContent` on folderId            |
| Internal workflow calls            | IAM           | Role must have `execute-api:Invoke`      |

---

## 8. Observability

### 8.1 CloudWatch Metrics

```
Namespace: Processing/FileProcessing

Dimensions:
  - Environment: dev | stage | prod
  - Phase: phase0 | phase1 | phase2 | phase3

Metrics:
  - EventsReceived (Count)
  - EventsAccepted (Count)
  - EventsRejected (Count)
  - JobsQueued (Count)
  - JobsRunning (Count)
  - JobsSucceeded (Count)
  - JobsFailed (Count)
  - PhaseDuration (Milliseconds, p50/p90/p99)
  - LLMRequestCount (Count)
  - LLMInputTokens (Count)
  - LLMOutputTokens (Count)
  - LLMRateLimitRetries (Count)
  - WikiPagesCreated (Count)
  - WikiPagesUpdated (Count)
  - ContradictionsDetected (Count)
  - QueueDepth (Count)
  - QueueOldestMessageAge (Seconds)
```

### 8.2 Structured Logging

All log lines include the trace ID hierarchy:

```json
{
  "level": "INFO",
  "timestamp": "2026-05-12T10:30:00.000Z",
  "requestId": "abc-123",
  "traceId": "version-1",
  "fileVersionId": "version-1",
  "fileId": "file-1",
  "folderId": "folder-1",
  "phase": "phase1",
  "chunkIndex": 3,
  "taskId": "version-1#extractClaims#3",
  "attempt": 1,
  "message": "Chunk extraction complete",
  "claimCount": 12,
  "inputTokens": 2500,
  "outputTokens": 800,
  "durationMs": 3200
}
```

Key fields:

| Field       | Purpose                                  | Indexed |
| ----------- | ---------------------------------------- | ------- |
| `traceId`   | Primary correlation ID (= fileVersionId) | Yes     |
| `folderId`  | Wiki aggregate key                       | Yes     |
| `taskId`    | Slug-level work unit                     | Yes     |
| `phase`     | Processing phase                         | Yes     |
| `requestId` | AWS request ID                           | No      |

### 8.3 Alarms

| Alarm                  | Threshold                    | Action       |
| ---------------------- | ---------------------------- | ------------ |
| HighFailureRate        | > 10% jobs failed in 1 hour  | Page on-call |
| LLMRateLimitExhaustion | > 100 retries in 5 min       | Alert Slack  |
| QueueBacklog           | > 100 messages, age > 30 min | Alert Slack  |
| DLQMessages            | > 0                          | Alert Slack  |
| WikiLockContention     | > 10 lock failures in 5 min  | Log warning  |

### 8.4 Distributed Tracing

#### Correlation ID Strategy

Use **`fileVersionId`** as the primary distributed trace/correlation ID for ingest processing.
Use **`folderId`** as the aggregate/wiki status key.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TRACE ID HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   traceId = fileVersionId     ←── One upload, end-to-end lifecycle          │
│   wikiId = folderId           ←── Folder/wiki aggregate status              │
│   sourceId = fileId           ←── Stable file identity (version-independent)│
│   taskId = {fileVersionId}#{phase}#{slug}  ←── Slug-level work unit         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

The lifecycle of a single uploaded source:

```
FileVersion INSERT (traceId = fileVersionId)
    │
    ├── dispatcher                    traceId: version-1
    │
    ├── Phase 0 normalize             traceId: version-1
    │
    ├── Phase 1 extract claims        traceId: version-1
    │   ├── chunk 0                   taskId: version-1#extractClaims#0
    │   ├── chunk 1                   taskId: version-1#extractClaims#1
    │   └── chunk N                   taskId: version-1#extractClaims#N
    │
    ├── Phase 2 synthesize            traceId: version-1
    │   ├── concepts/topic-a          taskId: version-1#synthesize#topic-a
    │   └── references/topic-b        taskId: version-1#synthesize#topic-b
    │
    └── Phase 3 maintain wiki         traceId: version-1, lockKey: folder-1
```

#### Why Not X-Ray Trace IDs?

AWS X-Ray/OpenTelemetry trace IDs are generated per execution/request and are awkward as durable
product identifiers. Instead:

1. Use `fileVersionId` as the **domain correlation ID**
2. Attach it as an **attribute/span tag** in X-Ray/OpenTelemetry
3. X-Ray traces become queryable by `fileVersionId`

```python
# Lambda tracing setup
from aws_xray_sdk.core import xray_recorder

def handler(event, context):
    file_version_id = event["fileVersionId"]
    folder_id = event["folderId"]

    # Add domain IDs to X-Ray segment
    segment = xray_recorder.current_segment()
    segment.put_annotation("fileVersionId", file_version_id)
    segment.put_annotation("folderId", folder_id)
    segment.put_annotation("phase", "phase1")

    # X-Ray console: filter by annotation.fileVersionId = "version-1"
```

#### Required Fields by Component

Every artifact must include the trace ID:

| Component                | Required Fields                                          |
| ------------------------ | -------------------------------------------------------- |
| Log line                 | `traceId` (fileVersionId), `folderId`, `phase`, `taskId` |
| Queue message            | `fileVersionId`, `folderId` in body                      |
| DynamoDB task row        | `fileVersionId` as GSI, `taskId` as PK                   |
| Step Functions execution | `fileVersionId` in input and execution name              |
| LLM call metadata        | `fileVersionId`, `taskId`, tokens, model                 |
| DLQ message              | `fileVersionId`, `phase`, `errorCode`                    |
| S3 object metadata       | `x-amz-meta-file-version-id` header                      |

#### User-Facing Status Queries

**Debug one upload:**

```
GET /processing/jobs/{fileVersionId}
```

Returns:

```json
{
  "fileVersionId": "version-1",
  "fileId": "file-1",
  "folderId": "folder-1",
  "status": "running",
  "currentPhase": "phase2",
  "progress": {
    "phase0": { "status": "complete", "durationMs": 45000 },
    "phase1": { "status": "complete", "chunksTotal": 8, "chunksComplete": 8 },
    "phase2": { "status": "running", "pagesTotal": 5, "pagesComplete": 2 },
    "phase3": { "status": "pending" }
  },
  "createdAt": "2026-05-12T10:00:00Z",
  "updatedAt": "2026-05-12T10:15:00Z"
}
```

**Show wiki status:**

```
GET /wikis/{folderId}
```

Returns:

```json
{
  "folderId": "folder-1",
  "wikiStatus": "updating",
  "latestUpdatedAt": "2026-05-12T12:00:00Z",
  "stats": {
    "sourceCount": 12,
    "conceptCount": 45,
    "entityCount": 23,
    "contradictionCount": 2
  },
  "activeFileVersionIds": ["version-7", "version-8"],
  "failedFileVersionIds": ["version-5"],
  "lastSuccessfulMaintenanceTraceId": "version-6",
  "recentIngests": [
    {
      "fileVersionId": "version-6",
      "fileId": "file-3",
      "status": "succeeded",
      "completedAt": "2026-05-12T11:30:00Z",
      "pagesCreated": 3,
      "pagesUpdated": 2
    }
  ]
}
```

Wiki status values:

| Status     | Meaning                                      |
| ---------- | -------------------------------------------- |
| `empty`    | No sources processed yet                     |
| `ready`    | All known sources processed, wiki up to date |
| `updating` | One or more sources currently processing     |
| `degraded` | Some sources failed, wiki partially complete |

#### Phase 3 Tracing

Phase 3 wiki maintenance includes both trace and lock context:

```json
{
  "traceId": "version-7",
  "lockKey": "folder-1",
  "message": "Acquired wiki lock",
  "lockedBy": "version-7",
  "queuedBehind": ["version-8"]
}
```

When multiple file versions target the same folder, log the queue position:

```json
{
  "traceId": "version-8",
  "lockKey": "folder-1",
  "message": "Waiting for wiki lock",
  "position": 2,
  "waitingBehind": "version-7"
}
```

#### Trace ID Propagation

```
┌──────────────┐    FileVersionCommitted     ┌──────────────┐
│   Zarya      │ ─────────────────────────▶  │  Dispatcher  │
│ DynamoDB     │   fileVersionId: v-1        │   Lambda     │
│   Stream     │   folderId: f-1             │              │
└──────────────┘                             └──────────────┘
                                                    │
                                    SQS message with:
                                    - fileVersionId: v-1
                                    - folderId: f-1
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │  Step Functions  │
                                          │  executionName:  │
                                          │  ingest-v-1      │
                                          └──────────────────┘
                                                    │
                    ┌───────────────────────────────┼───────────────────────────────┐
                    │                               │                               │
                    ▼                               ▼                               ▼
           ┌──────────────┐                ┌──────────────┐                ┌──────────────┐
           │   Phase 0    │                │   Phase 1    │                │   Phase 2    │
           │ traceId: v-1 │                │ traceId: v-1 │                │ traceId: v-1 │
           └──────────────┘                │ taskId:      │                │ taskId:      │
                    │                      │ v-1#extract#0│                │ v-1#synth#a  │
                    │                      └──────────────┘                └──────────────┘
                    │                               │                               │
                    └───────────────────────────────┼───────────────────────────────┘
                                                    │
                                    FIFO queue with:
                                    - MessageGroupId: f-1
                                    - traceId: v-1
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │   Phase 3        │
                                          │ traceId: v-1     │
                                          │ lockKey: f-1     │
                                          └──────────────────┘
```

#### Quick Reference

| Need to...                    | Use...                                    |
| ----------------------------- | ----------------------------------------- |
| Debug one upload              | `fileVersionId`                           |
| Show wiki status              | `folderId`                                |
| Distributed trace ID          | `fileVersionId`                           |
| File identity across versions | `fileId`                                  |
| Identify slug-level task      | `taskId = {fileVersionId}#{phase}#{slug}` |
| Serialize wiki writes         | `folderId` (FIFO MessageGroupId)          |

---

## 9. Failure Handling

### 9.1 Retry Policies

| Failure Type          | Retry Strategy       | Max Attempts | Backoff                      |
| --------------------- | -------------------- | ------------ | ---------------------------- |
| Transient AWS errors  | Exponential          | 5            | 1s, 2s, 4s, 8s, 16s          |
| Bedrock rate limit    | Exponential + jitter | 10           | 30s base                     |
| Source object missing | Linear               | 3            | 5s                           |
| Validation failure    | No retry             | 1            | -                            |
| Wiki lock contention  | Step Functions retry | 3            | 30s, 60s, 120s (exponential) |

### 9.2 Dead Letter Handling

```python
def process_dlq_message(event, context):
    """Alert on DLQ messages and optionally auto-remediate."""
    for record in event["Records"]:
        message = json.loads(record["body"])

        # Log with full context
        log.error("DLQ message received", extra={
            "fileVersionId": message.get("fileVersionId"),
            "phase": message.get("phase"),
            "errorCode": message.get("errorCode"),
            "attempts": message.get("attempts"),
        })

        # Send to Slack/PagerDuty
        alert_dlq_message(message)

        # Auto-remediate known issues
        if message.get("errorCode") == "SOURCE_OBJECT_MISSING":
            # Mark job as failed, don't retry
            update_job(message["fileVersionId"], {
                "status": "failed",
                "errorCode": "SOURCE_OBJECT_MISSING",
                "errorMessage": "Source file was deleted before processing completed",
            })
```

### 9.3 Partial Success Handling

When some pages fail synthesis but others succeed:

1. Mark job status as `partial`
2. Proceed to Phase 3 with successful pages
3. Record failed pages in job metadata
4. Allow manual retry of failed pages via API

---

## 10. Security

### 10.1 Data Classification

| Data                | Classification | Storage       | Encryption |
| ------------------- | -------------- | ------------- | ---------- |
| Source PDFs         | User content   | Zarya S3      | SSE-S3     |
| Normalized Markdown | Derived        | Processing S3 | SSE-S3     |
| Claims/Evidence     | Derived        | Processing S3 | SSE-S3     |
| Wiki pages          | Derived        | Wiki S3       | SSE-S3     |
| Job metadata        | Operational    | DynamoDB      | Encrypted  |

### 10.2 Secrets Management

| Secret            | Storage         | Rotation  |
| ----------------- | --------------- | --------- |
| AWS credentials   | IAM roles       | Automatic |
| Bedrock access    | IAM roles       | Automatic |
| API keys (if any) | Secrets Manager | 90 days   |

### 10.3 Audit Trail

- All job state changes logged to CloudWatch Logs
- All wiki modifications logged to `log.md`
- S3 access logging enabled
- CloudTrail for API/Console actions

---

## 11. Developer Workflow & Testing

### 11.1 Repository Structure

pnpm monorepo with isolated packages:

```
processing-service/
├── packages/
│   ├── core/                    # Pure business logic (no AWS SDK)
│   │   ├── src/
│   │   │   ├── claims/          # Claim extraction, normalization, dedup
│   │   │   ├── wiki/            # Wiki page models, merge logic
│   │   │   ├── validation/      # Page validation, repair
│   │   │   └── types/           # Shared TypeScript types
│   │   └── tests/
│   ├── lambda/                  # Lambda handlers (thin wrappers)
│   │   ├── src/
│   │   │   ├── stream-dispatcher/
│   │   │   ├── phase0-normalize/
│   │   │   ├── phase1-extract-chunk/
│   │   │   ├── phase1-aggregate/
│   │   │   ├── phase2-synthesize/
│   │   │   ├── phase3-maintain/
│   │   │   └── api/
│   │   └── tests/
│   └── e2e/                     # End-to-end tests
│       ├── fixtures/            # Sample PDFs, expected outputs
│       └── tests/
├── infra/
│   └── cloudformation/
├── .github/
│   └── workflows/
├── package.json
├── pnpm-workspace.yaml
└── tsconfig.base.json
```

### 11.2 Local Quality Gate

The one local and CI quality gate:

```bash
pnpm check
```

This runs:

```json
{
  "scripts": {
    "check": "pnpm format:check && pnpm lint && pnpm typecheck && pnpm build && pnpm test && pnpm coverage",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint packages/*/src packages/*/tests",
    "typecheck": "tsc --build",
    "build": "pnpm -r build",
    "test": "vitest run",
    "coverage": "vitest run --coverage",
    "e2e:local": "vitest run --project e2e --config vitest.e2e.config.ts",
    "e2e:dev": "ENVIRONMENT=dev vitest run --project e2e --config vitest.e2e.config.ts",
    "aws:check-profiles": "aws sts get-caller-identity --profile sdai-dev && aws sts get-caller-identity --profile sdai-stage && aws sts get-caller-identity --profile sdai-prod"
  }
}
```

### 11.3 Testing Strategy

#### Unit Tests (packages/core)

Pure business logic with no AWS dependencies:

```typescript
// packages/core/tests/claims/normalize.test.ts
import { describe, it, expect } from "vitest";
import { normalizeClaims, generateClaimId } from "../../src/claims/normalize";

describe("normalizeClaims", () => {
  it("deduplicates claims with same text and locator", () => {
    const raw = [
      { claim: "X is Y", evidence: "quote", locator: "L10" },
      { claim: "X is Y", evidence: "quote", locator: "L10" }, // dupe
    ];
    const normalized = normalizeClaims(raw, "test-slug");
    expect(normalized).toHaveLength(1);
  });

  it("generates stable claim IDs", () => {
    const id1 = generateClaimId("slug", "claim text", "L10", 0);
    const id2 = generateClaimId("slug", "claim text", "L10", 0);
    expect(id1).toBe(id2);
  });
});
```

#### Integration Tests (packages/lambda)

Lambda handlers with mocked AWS SDK:

```typescript
// packages/lambda/tests/phase1-extract-chunk.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { mockClient } from "aws-sdk-client-mock";
import {
  S3Client,
  GetObjectCommand,
  PutObjectCommand,
} from "@aws-sdk/client-s3";
import {
  BedrockRuntimeClient,
  InvokeModelCommand,
} from "@aws-sdk/client-bedrock-runtime";
import { handler } from "../../src/phase1-extract-chunk";

const s3Mock = mockClient(S3Client);
const bedrockMock = mockClient(BedrockRuntimeClient);

beforeEach(() => {
  s3Mock.reset();
  bedrockMock.reset();
});

describe("phase1-extract-chunk", () => {
  it("extracts claims from chunk and saves to S3", async () => {
    s3Mock.on(GetObjectCommand).resolves({
      Body: sdkStreamMixin(Readable.from(["# Source\n\nSome content here."])),
    });
    bedrockMock.on(InvokeModelCommand).resolves({
      body: JSON.stringify({
        claims: [
          { claim: "Content exists", evidence: "Some content", locator: "L3" },
        ],
      }),
    });

    const result = await handler({
      fileVersionId: "test-version",
      chunk: { index: 0, start_line: 1, end_line: 3 },
    });

    expect(result.claimCount).toBe(1);
    expect(s3Mock.calls()).toHaveLength(2); // GetObject + PutObject
  });
});
```

#### End-to-End Tests (packages/e2e)

Local fixtures for CI, deployed environment for smoke tests:

```typescript
// packages/e2e/tests/ingest-flow.test.ts
import { describe, it, expect } from "vitest";
import { processTestPdf, waitForJobCompletion, getWikiPage } from "../helpers";

describe("ingest flow", () => {
  it("processes a small PDF and creates wiki pages", async () => {
    const { fileVersionId, folderId } = await processTestPdf(
      "fixtures/sample-3page.pdf",
    );

    const job = await waitForJobCompletion(fileVersionId, {
      timeoutMs: 120_000,
    });
    expect(job.status).toBe("succeeded");

    const index = await getWikiPage(folderId, "index.md");
    expect(index).toContain("sources/");
  });

  it("detects contradictions when sources conflict", async () => {
    const { folderId } = await processTestPdf("fixtures/source-a.pdf");
    await processTestPdf("fixtures/source-b-contradicts-a.pdf", { folderId });

    const contradictions = await getWikiPage(folderId, "contradictions/");
    expect(contradictions).toBeDefined();
  });
});
```

### 11.4 Coverage Requirements

| Package | Line Coverage | Branch Coverage | Notes                     |
| ------- | ------------- | --------------- | ------------------------- |
| core    | 90%           | 85%             | Pure logic, high coverage |
| lambda  | 80%           | 75%             | Handler code              |
| e2e     | N/A           | N/A             | Integration coverage      |

### 11.5 Local Development Scripts

```bash
# Run full quality gate
pnpm check

# Run tests in watch mode
pnpm test:watch

# Run e2e against local fixtures
pnpm e2e:local

# Run e2e against deployed dev environment
pnpm e2e:dev

# Verify AWS profiles are configured
pnpm aws:check-profiles

# Lint CloudFormation templates
pnpm cfn:lint

# Validate CloudFormation against AWS
pnpm cfn:validate:dev
```

---

## 12. IaC Layout

### 12.1 CloudFormation Templates

```
infra/cloudformation/
├── processing-foundation.yaml      # Durable resources
├── processing-app.yaml             # Runtime resources
├── processing-github-actions.yaml  # CI/CD permissions
└── parameters/
    ├── dev.json
    ├── stage.json
    └── prod.json
```

### 12.2 Foundation Stack (processing-foundation.yaml)

Owns durable, retained resources:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Processing service foundation - durable resources

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
  ZaryaFilesBucket:
    Type: String
    Description: Zarya's committed objects bucket (cross-stack import)
  ZaryaFileVersionsTableArn:
    Type: String
    Description: Zarya's FileVersions table ARN for stream access

Conditions:
  IsProtected:
    !Or [!Equals [!Ref Environment, stage], !Equals [!Ref Environment, prod]]

Resources:
  # =========================================================================
  # S3 Buckets
  # =========================================================================
  ProcessingBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub processing-${Environment}-${AWS::AccountId}
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: CleanupOldVersions
            Status: Enabled
            NoncurrentVersionExpiration:
              NoncurrentDays: 90

  WikisBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub wikis-${Environment}-${AWS::AccountId}
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled

  DeploymentArtifactsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub processing-artifacts-${Environment}-${AWS::AccountId}
      VersioningConfiguration:
        Status: Enabled

  # =========================================================================
  # DynamoDB Tables
  # =========================================================================
  ProcessingJobsTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      TableName: !Sub processing-jobs-${Environment}
      BillingMode: PAY_PER_REQUEST
      DeletionProtectionEnabled: !If [IsProtected, true, false]
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
      AttributeDefinitions:
        - AttributeName: fileVersionId
          AttributeType: S
        - AttributeName: folderId
          AttributeType: S
        - AttributeName: createdAt
          AttributeType: S
      KeySchema:
        - AttributeName: fileVersionId
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: byFolder
          KeySchema:
            - AttributeName: folderId
              KeyType: HASH
            - AttributeName: createdAt
              KeyType: RANGE
          Projection:
            ProjectionType: ALL

  ProcessingTasksTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      TableName: !Sub processing-tasks-${Environment}
      BillingMode: PAY_PER_REQUEST
      DeletionProtectionEnabled: !If [IsProtected, true, false]
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
      AttributeDefinitions:
        - AttributeName: taskId
          AttributeType: S
        - AttributeName: fileVersionId
          AttributeType: S
      KeySchema:
        - AttributeName: taskId
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: byFileVersion
          KeySchema:
            - AttributeName: fileVersionId
              KeyType: HASH
            - AttributeName: taskId
              KeyType: RANGE
          Projection:
            ProjectionType: ALL

  WikiLocksTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub wiki-locks-${Environment}
      BillingMode: PAY_PER_REQUEST
      TimeToLiveSpecification:
        AttributeName: ttl
        Enabled: true
      AttributeDefinitions:
        - AttributeName: folderId
          AttributeType: S
      KeySchema:
        - AttributeName: folderId
          KeyType: HASH

  RateLimitsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub rate-limits-${Environment}
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: modelId
          AttributeType: S
      KeySchema:
        - AttributeName: modelId
          KeyType: HASH

  # =========================================================================
  # Secrets Manager
  # =========================================================================
  AppEnvSecret:
    Type: AWS::SecretsManager::Secret
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      Name: !Sub processing/${Environment}/app-env
      Description: Processing service runtime configuration
      SecretString: !Sub |
        {
          "BEDROCK_MODEL_ID": "qwen.qwen3-coder-30b-a3b-v1:0",
          "BEDROCK_MAX_TOKENS": "8192",
          "BEDROCK_TEMPERATURE": "0.1",
          "MAX_CONCURRENT_WORKFLOWS": "10",
          "FEATURE_FLAG_CONTRADICTION_DETECTION": "true"
        }

  # =========================================================================
  # IAM Managed Policies
  # =========================================================================
  LambdaRuntimePolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: !Sub processing-lambda-runtime-${Environment}
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - s3:GetObject
            Resource:
              - !Sub arn:aws:s3:::${ZaryaFilesBucket}/objects/*
          - Effect: Allow
            Action:
              - s3:GetObject
              - s3:PutObject
            Resource:
              - !Sub ${ProcessingBucket.Arn}/*
          - Effect: Allow
            Action:
              - s3:GetObject
              - s3:PutObject
              # No DeleteObject - wikis are retained indefinitely
            Resource:
              - !Sub ${WikisBucket.Arn}/*
          - Effect: Allow
            Action:
              - dynamodb:GetItem
              - dynamodb:PutItem
              - dynamodb:UpdateItem
              - dynamodb:Query
            Resource:
              - !GetAtt ProcessingJobsTable.Arn
              - !Sub ${ProcessingJobsTable.Arn}/index/*
              - !GetAtt ProcessingTasksTable.Arn
              - !Sub ${ProcessingTasksTable.Arn}/index/*
              - !GetAtt WikiLocksTable.Arn
              - !GetAtt RateLimitsTable.Arn
          - Effect: Allow
            Action:
              - bedrock:InvokeModel
            Resource:
              - arn:aws:bedrock:*:*:foundation-model/*
          - Effect: Allow
            Action:
              - secretsmanager:GetSecretValue
            Resource:
              - !Ref AppEnvSecret

Outputs:
  ProcessingBucketName:
    Value: !Ref ProcessingBucket
    Export:
      Name: !Sub ${AWS::StackName}-ProcessingBucket
  WikisBucketName:
    Value: !Ref WikisBucket
    Export:
      Name: !Sub ${AWS::StackName}-WikisBucket
  DeploymentArtifactsBucketName:
    Value: !Ref DeploymentArtifactsBucket
    Export:
      Name: !Sub ${AWS::StackName}-ArtifactsBucket
  ProcessingJobsTableArn:
    Value: !GetAtt ProcessingJobsTable.Arn
    Export:
      Name: !Sub ${AWS::StackName}-JobsTableArn
  LambdaRuntimePolicyArn:
    Value: !Ref LambdaRuntimePolicy
    Export:
      Name: !Sub ${AWS::StackName}-LambdaRuntimePolicy
  AppEnvSecretArn:
    Value: !Ref AppEnvSecret
    Export:
      Name: !Sub ${AWS::StackName}-AppEnvSecretArn
```

### 12.3 App Stack (processing-app.yaml)

Owns runtime and orchestration resources:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Processing service app - runtime resources

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
  ArtifactVersion:
    Type: String
    Description: Version prefix for artifacts in S3 (usually git SHA)
  ZaryaFileVersionsStreamArn:
    Type: String
    Description: Zarya FileVersions DynamoDB stream ARN

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  # =========================================================================
  # SQS Queues
  # =========================================================================
  IntakeQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub processing-intake-${Environment}
      VisibilityTimeoutSeconds: 900
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt IntakeDLQ.Arn
        maxReceiveCount: 3

  IntakeDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub processing-intake-dlq-${Environment}
      MessageRetentionPeriod: 1209600 # 14 days

  WikiMaintenanceQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub wiki-maintenance-${Environment}.fifo
      FifoQueue: true
      ContentBasedDeduplication: false
      VisibilityTimeoutSeconds: 600
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt WikiMaintenanceDLQ.Arn
        maxReceiveCount: 5

  WikiMaintenanceDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub wiki-maintenance-dlq-${Environment}.fifo
      FifoQueue: true
      MessageRetentionPeriod: 1209600

  # =========================================================================
  # Lambda Functions
  # =========================================================================
  StreamDispatcherFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub processing-stream-dispatcher-${Environment}
      Runtime: nodejs20.x
      Handler: stream-dispatcher/index.handler
      MemorySize: 256
      Timeout: 30
      ReservedConcurrentExecutions: 10
      Code:
        S3Bucket:
          Fn::ImportValue:
            Fn::Sub: "processing-foundation-${Environment}-ArtifactsBucket"
        S3Key: !Sub lambda/${ArtifactVersion}/stream-dispatcher.zip
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
          INTAKE_QUEUE_URL: !Ref IntakeQueue
      Role: !GetAtt StreamDispatcherRole.Arn

  StreamDispatcherEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !Ref ZaryaFileVersionsStreamArn
      FunctionName: !Ref StreamDispatcherFunction
      StartingPosition: LATEST
      FilterCriteria:
        Filters:
          - Pattern: '{"eventName": ["INSERT"]}'

  Phase0NormalizeFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub processing-phase0-normalize-${Environment}
      Runtime: python3.12
      Handler: phase0-normalize/index.handler
      MemorySize: 3008
      Timeout: 300
      EphemeralStorage:
        Size: 1024
      ReservedConcurrentExecutions: 10
      Code:
        S3Bucket:
          Fn::ImportValue:
            Fn::Sub: "processing-foundation-${Environment}-ArtifactsBucket"
        S3Key: !Sub lambda/${ArtifactVersion}/phase0-normalize.zip
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
      Role: !GetAtt LambdaExecutionRole.Arn
      Layers:
        - !Ref Pymupdf4llmLayer

  # ... additional Lambda functions follow same pattern

  # =========================================================================
  # Step Functions
  # =========================================================================
  ProcessingStateMachine:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      StateMachineName: !Sub processing-workflow-${Environment}
      DefinitionS3Location:
        Bucket:
          Fn::ImportValue:
            Fn::Sub: "processing-foundation-${Environment}-ArtifactsBucket"
        Key: !Sub sfn/${ArtifactVersion}/workflow.asl.json
      DefinitionSubstitutions:
        ValidateAndEnrichFunctionArn: !GetAtt ValidateAndEnrichFunction.Arn
        Phase0NormalizeFunctionArn: !GetAtt Phase0NormalizeFunction.Arn
        ExtractChunkFunctionArn: !GetAtt ExtractChunkFunction.Arn
        AggregateClaimsFunctionArn: !GetAtt AggregateClaimsFunction.Arn
        SynthesizePageFunctionArn: !GetAtt SynthesizePageFunction.Arn
        MarkJobStatusFunctionArn: !GetAtt MarkJobStatusFunction.Arn
        WikiMaintenanceQueueUrl: !Ref WikiMaintenanceQueue
      RoleArn: !GetAtt StepFunctionsRole.Arn

  # =========================================================================
  # API Gateway
  # =========================================================================
  ProcessingApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: !Sub processing-api-${Environment}
      ProtocolType: HTTP
      CorsConfiguration:
        AllowOrigins:
          - !If [IsProd, "https://app.example.com", "*"]
        AllowMethods: [GET, OPTIONS]
        AllowHeaders: [Authorization, Content-Type]

  # =========================================================================
  # CloudWatch Alarms
  # =========================================================================
  HighFailureRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub processing-high-failure-rate-${Environment}
      MetricName: JobsFailed
      Namespace: Processing/FileProcessing
      Statistic: Sum
      Period: 3600
      EvaluationPeriods: 1
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref AlertTopic

  DLQMessagesAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub processing-dlq-messages-${Environment}
      MetricName: ApproximateNumberOfMessagesVisible
      Namespace: AWS/SQS
      Dimensions:
        - Name: QueueName
          Value: !GetAtt IntakeDLQ.QueueName
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 0
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref AlertTopic

  AlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub processing-alerts-${Environment}

Outputs:
  ApiEndpoint:
    Value: !Sub https://${ProcessingApi}.execute-api.${AWS::Region}.amazonaws.com
    Export:
      Name: !Sub ${AWS::StackName}-ApiEndpoint
  StateMachineArn:
    Value: !Ref ProcessingStateMachine
    Export:
      Name: !Sub ${AWS::StackName}-StateMachineArn
```

### 12.4 GitHub Actions Bootstrap (processing-github-actions.yaml)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: GitHub Actions OIDC roles for processing service

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
  GitHubOrg:
    Type: String
    Default: SQLCODE917
  GitHubRepo:
    Type: String
    Default: processing-service

Mappings:
  # Branch restrictions per environment
  BranchRestrictions:
    dev:
      Pattern: "ref:refs/heads/dev"
    stage:
      Pattern: "ref:refs/heads/stage"
    prod:
      Pattern: "ref:refs/heads/main"

Resources:
  DeployRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub processing-github-deploy-${Environment}
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub arn:aws:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com
            Action: sts:AssumeRoleWithWebIdentity
            Condition:
              StringEquals:
                token.actions.githubusercontent.com:aud: sts.amazonaws.com
              StringLike:
                # Restrict to specific branch per environment
                token.actions.githubusercontent.com:sub: !Sub
                  - "repo:${GitHubOrg}/${GitHubRepo}:${BranchPattern}"
                  - BranchPattern:
                      !FindInMap [BranchRestrictions, !Ref Environment, Pattern]
      ManagedPolicyArns:
        - !Ref DeployPolicy

  DeployPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: !Sub processing-github-deploy-policy-${Environment}
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          # CloudFormation change set workflow (narrowed from cloudformation:*)
          - Effect: Allow
            Action:
              - cloudformation:CreateChangeSet
              - cloudformation:DeleteChangeSet
              - cloudformation:DescribeChangeSet
              - cloudformation:DescribeStacks
              - cloudformation:ExecuteChangeSet
              - cloudformation:GetStackPolicy
              - cloudformation:SetStackPolicy
              - cloudformation:ValidateTemplate
            Resource:
              - !Sub arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/processing-foundation-${Environment}/*
              - !Sub arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/processing-app-${Environment}/*
          # S3 artifact upload
          - Effect: Allow
            Action:
              - s3:PutObject
              - s3:GetObject
            Resource:
              - !Sub arn:aws:s3:::processing-artifacts-${Environment}-${AWS::AccountId}/*
          # Lambda update (for hotfix without full stack deploy)
          - Effect: Allow
            Action:
              - lambda:UpdateFunctionCode
              - lambda:GetFunction
            Resource:
              - !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:processing-*-${Environment}
          # IAM passrole for CloudFormation to create roles
          - Effect: Allow
            Action:
              - iam:PassRole
            Resource:
              - !Sub arn:aws:iam::${AWS::AccountId}:role/processing-*-${Environment}
            Condition:
              StringEquals:
                iam:PassedToService: cloudformation.amazonaws.com

  DriftCheckRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub processing-github-drift-${Environment}
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Federated: !Sub arn:aws:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com
            Action: sts:AssumeRoleWithWebIdentity
            Condition:
              StringEquals:
                token.actions.githubusercontent.com:aud: sts.amazonaws.com
              StringLike:
                # Drift check allowed from any branch (read-only operation)
                token.actions.githubusercontent.com:sub: !Sub repo:${GitHubOrg}/${GitHubRepo}:*
      Policies:
        - PolicyName: DriftDetection
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - cloudformation:DetectStackDrift
                  - cloudformation:DescribeStackDriftDetectionStatus
                  - cloudformation:DescribeStackResourceDrifts
                  - cloudformation:DescribeStacks
                Resource:
                  - !Sub arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/processing-*-${Environment}/*

Outputs:
  DeployRoleArn:
    Value: !GetAtt DeployRole.Arn
  DriftCheckRoleArn:
    Value: !GetAtt DriftCheckRole.Arn
```

### 12.5 Environment Parameter Files

Parameter files use a raw array shape (not `{"Parameters": [...]}`). Each stack has its own parameter file.

```
infra/cloudformation/parameters/
├── dev-foundation.json
├── dev-app.json
├── stage-foundation.json
├── stage-app.json
├── prod-foundation.json
└── prod-app.json
```

```json
// infra/cloudformation/parameters/dev-foundation.json
// Raw array shape for direct use with --parameters
[
  { "ParameterKey": "Environment", "ParameterValue": "dev" },
  {
    "ParameterKey": "ZaryaFilesBucket",
    "ParameterValue": "zarya-files-dev-123456789012"
  },
  {
    "ParameterKey": "ZaryaFileVersionsTableArn",
    "ParameterValue": "arn:aws:dynamodb:us-east-1:123456789012:table/zarya-dev-file-versions"
  },
  {
    "ParameterKey": "ZaryaFileVersionsStreamArn",
    "ParameterValue": "arn:aws:dynamodb:us-east-1:123456789012:table/zarya-dev-file-versions/stream/2026-01-01T00:00:00.000"
  }
]
```

```json
// infra/cloudformation/parameters/dev-app.json
[
  {
    "ParameterKey": "BedrockModelId",
    "ParameterValue": "qwen.qwen3-coder-30b-a3b-v1:0"
  },
  { "ParameterKey": "Phase1Concurrency", "ParameterValue": "10" },
  { "ParameterKey": "Phase2Concurrency", "ParameterValue": "5" }
]
```

---

## 13. CI/CD Pipeline

### 13.1 CI Workflow (.github/workflows/ci.yml)

```yaml
name: CI

on:
  push:
    branches: [dev, stage, main]
  pull_request:
    branches: [dev, stage, main]

jobs:
  verify:
    name: Verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: "10.33.2"

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      # Single quality gate (matches Zarya pattern)
      - name: pnpm check
        run: pnpm check

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage/lcov.info

  cloudformation:
    name: CloudFormation Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install cfn-lint
        run: pip install cfn-lint

      - name: Lint CloudFormation templates
        run: cfn-lint infra/cloudformation/*.yaml

      - name: Validate parameter files
        run: |
          # Parameters file shape: raw array of {ParameterKey, ParameterValue}
          for env in dev stage prod; do
            for stack in foundation app; do
              FILE="infra/cloudformation/parameters/${env}-${stack}.json"
              echo "Validating ${FILE}"
              jq -e 'type == "array" and (length == 0 or .[0].ParameterKey)' "$FILE" > /dev/null
            done
          done

  e2e-local:
    name: E2E Tests (Local)
    needs: verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: "10.33.2"

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run local e2e tests
        run: pnpm e2e:local

      - name: Upload test artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-local-results
          path: packages/e2e/test-results/
```

### 13.2 Deployment Workflow (.github/workflows/deploy.yml)

```yaml
name: Deploy

on:
  push:
    branches: [dev, stage, main]
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options: [dev, stage, prod]

env:
  AWS_REGION: us-east-1

jobs:
  resolve-environment:
    name: Resolve Environment
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.resolve.outputs.environment }}
    steps:
      - id: resolve
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            ENV="${{ inputs.environment }}"
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            ENV="prod"
          elif [[ "${{ github.ref }}" == "refs/heads/stage" ]]; then
            ENV="stage"
          else
            ENV="dev"
          fi
          echo "environment=$ENV" >> "$GITHUB_OUTPUT"

      - name: Validate branch/environment match
        if: github.event_name == 'workflow_dispatch'
        run: |
          BRANCH="${{ github.ref_name }}"
          ENV="${{ inputs.environment }}"
          if [[ "$ENV" == "prod" && "$BRANCH" != "main" ]]; then
            echo "::error::Cannot deploy to prod from $BRANCH. Use main branch."
            exit 1
          fi
          if [[ "$ENV" == "stage" && "$BRANCH" != "stage" ]]; then
            echo "::error::Cannot deploy to stage from $BRANCH. Use stage branch."
            exit 1
          fi

  verify:
    name: Verify
    needs: resolve-environment
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: "10.33.2"
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm check

  cloudformation-lint:
    name: CloudFormation Lint
    needs: resolve-environment
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install cfn-lint
      - run: cfn-lint infra/cloudformation/*.yaml

  deploy-foundation:
    name: Deploy Foundation
    needs: [resolve-environment, verify, cloudformation-lint]
    runs-on: ubuntu-latest
    environment: ${{ needs.resolve-environment.outputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Validate template
        run: |
          aws cloudformation validate-template \
            --template-body file://infra/cloudformation/processing-foundation.yaml

      - name: Determine change set type
        id: stackcheck
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          STACK_NAME="processing-foundation-${ENV}"

          if aws cloudformation describe-stacks --stack-name "$STACK_NAME" 2>/dev/null; then
            echo "change_set_type=UPDATE" >> "$GITHUB_OUTPUT"
          else
            echo "change_set_type=CREATE" >> "$GITHUB_OUTPUT"
          fi
          echo "stack_name=$STACK_NAME" >> "$GITHUB_OUTPUT"

      - name: Create change set
        id: changeset
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          STACK_NAME="${{ steps.stackcheck.outputs.stack_name }}"
          CHANGE_SET_NAME="deploy-$(date +%Y%m%d%H%M%S)"
          CHANGE_SET_TYPE="${{ steps.stackcheck.outputs.change_set_type }}"

          aws cloudformation create-change-set \
            --stack-name "$STACK_NAME" \
            --change-set-name "$CHANGE_SET_NAME" \
            --template-body file://infra/cloudformation/processing-foundation.yaml \
            --parameters file://infra/cloudformation/parameters/${ENV}-foundation.json \
            --capabilities CAPABILITY_NAMED_IAM \
            --change-set-type "$CHANGE_SET_TYPE"

          echo "change_set_name=$CHANGE_SET_NAME" >> "$GITHUB_OUTPUT"

      - name: Wait for change set
        id: waitchangeset
        run: |
          # Wait and check if change set has changes
          aws cloudformation wait change-set-create-complete \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.changeset.outputs.change_set_name }}" 2>/dev/null || true

          STATUS=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.changeset.outputs.change_set_name }}" \
            --query 'Status' --output text)

          REASON=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.changeset.outputs.change_set_name }}" \
            --query 'StatusReason' --output text)

          if [[ "$STATUS" == "FAILED" && "$REASON" == *"didn't contain changes"* ]]; then
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
            echo "No changes to deploy"
          elif [[ "$STATUS" == "CREATE_COMPLETE" ]]; then
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          else
            echo "::error::Change set failed: $REASON"
            exit 1
          fi

      - name: Review change set for dangerous changes
        if: steps.waitchangeset.outputs.has_changes == 'true'
        run: |
          CHANGES=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.changeset.outputs.change_set_name }}" \
            --query 'Changes[?ResourceChange.Replacement==`True` || ResourceChange.Action==`Delete`]')

          if [[ "$CHANGES" != "[]" ]]; then
            echo "::error::Change set contains DELETE or REPLACE operations on retained resources:"
            echo "$CHANGES" | jq .
            exit 1
          fi

      - name: Execute change set
        if: steps.waitchangeset.outputs.has_changes == 'true'
        run: |
          aws cloudformation execute-change-set \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.changeset.outputs.change_set_name }}"

          aws cloudformation wait stack-${{ steps.stackcheck.outputs.change_set_type == 'CREATE' && 'create' || 'update' }}-complete \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}"

      - name: Apply foundation stack policy
        if: steps.waitchangeset.outputs.has_changes == 'true'
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          aws cloudformation set-stack-policy \
            --stack-name "${{ steps.stackcheck.outputs.stack_name }}" \
            --stack-policy-body file://infra/cloudformation/policies/${ENV}-foundation-policy.json

  deploy-app:
    name: Deploy App
    needs: [resolve-environment, deploy-foundation]
    runs-on: ubuntu-latest
    environment: ${{ needs.resolve-environment.outputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: "10.33.2"

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - name: Build Lambda package
        run: pnpm build:lambda

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Get artifact bucket
        id: artifacts
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          BUCKET=$(aws cloudformation describe-stacks \
            --stack-name "processing-foundation-${ENV}" \
            --query 'Stacks[0].Outputs[?OutputKey==`DeploymentArtifactsBucketName`].OutputValue' \
            --output text)
          echo "bucket=$BUCKET" >> "$GITHUB_OUTPUT"

      - name: Upload Lambda artifact
        id: upload
        run: |
          VERSION="${{ github.sha }}"
          # Upload per-function artifacts
          for f in dist/lambda/node/*.zip dist/lambda/python/*.zip; do
            BASENAME=$(basename "$f")
            aws s3 cp "$f" "s3://${{ steps.artifacts.outputs.bucket }}/lambda/${VERSION}/${BASENAME}"
          done
          # Upload Step Functions definition
          aws s3 cp dist/step-functions/processing-workflow.asl.json \
            "s3://${{ steps.artifacts.outputs.bucket }}/sfn/${VERSION}/workflow.asl.json"
          echo "artifact_version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Determine app stack type
        id: appstackcheck
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          STACK_NAME="processing-app-${ENV}"

          if aws cloudformation describe-stacks --stack-name "$STACK_NAME" 2>/dev/null; then
            echo "change_set_type=UPDATE" >> "$GITHUB_OUTPUT"
          else
            echo "change_set_type=CREATE" >> "$GITHUB_OUTPUT"
          fi
          echo "stack_name=$STACK_NAME" >> "$GITHUB_OUTPUT"

      - name: Create app change set
        id: appchangeset
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          STACK_NAME="${{ steps.appstackcheck.outputs.stack_name }}"
          CHANGE_SET_NAME="deploy-$(date +%Y%m%d%H%M%S)"
          CHANGE_SET_TYPE="${{ steps.appstackcheck.outputs.change_set_type }}"

          # Build parameters: merge env file with dynamic overrides
          PARAMS=$(jq -n \
            --argjson base "$(cat infra/cloudformation/parameters/${ENV}-app.json)" \
            --arg env "$ENV" \
            --arg version "${{ steps.upload.outputs.artifact_version }}" \
            '$base + [
              {"ParameterKey": "Environment", "ParameterValue": $env},
              {"ParameterKey": "ArtifactVersion", "ParameterValue": $version}
            ] | unique_by(.ParameterKey)')

          aws cloudformation create-change-set \
            --stack-name "$STACK_NAME" \
            --change-set-name "$CHANGE_SET_NAME" \
            --template-body file://infra/cloudformation/processing-app.yaml \
            --parameters "$PARAMS" \
            --capabilities CAPABILITY_NAMED_IAM \
            --change-set-type "$CHANGE_SET_TYPE"

          echo "change_set_name=$CHANGE_SET_NAME" >> "$GITHUB_OUTPUT"

      - name: Wait for app change set
        id: waitappchangeset
        run: |
          aws cloudformation wait change-set-create-complete \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.appchangeset.outputs.change_set_name }}" 2>/dev/null || true

          STATUS=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.appchangeset.outputs.change_set_name }}" \
            --query 'Status' --output text)

          REASON=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.appchangeset.outputs.change_set_name }}" \
            --query 'StatusReason' --output text)

          if [[ "$STATUS" == "FAILED" && "$REASON" == *"didn't contain changes"* ]]; then
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
            echo "No changes to deploy"
          elif [[ "$STATUS" == "CREATE_COMPLETE" ]]; then
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          else
            echo "::error::Change set failed: $REASON"
            exit 1
          fi

      - name: Review app change set for dangerous changes
        if: steps.waitappchangeset.outputs.has_changes == 'true'
        run: |
          # Block DELETE/REPLACE on stateful app resources
          CHANGES=$(aws cloudformation describe-change-set \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.appchangeset.outputs.change_set_name }}" \
            --query "Changes[?(ResourceChange.Replacement==\`True\` || ResourceChange.Action==\`Delete\`) && (ResourceChange.LogicalResourceId | contains(@, 'Table') || contains(@, 'Bucket') || contains(@, 'Queue'))]")

          if [[ "$CHANGES" != "[]" ]]; then
            echo "::error::App change set contains DELETE or REPLACE operations on stateful resources:"
            echo "$CHANGES" | jq .
            exit 1
          fi

      - name: Execute app change set
        if: steps.waitappchangeset.outputs.has_changes == 'true'
        run: |
          aws cloudformation execute-change-set \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --change-set-name "${{ steps.appchangeset.outputs.change_set_name }}"

          aws cloudformation wait stack-${{ steps.appstackcheck.outputs.change_set_type == 'CREATE' && 'create' || 'update' }}-complete \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}"

      - name: Apply app stack policy
        if: steps.waitappchangeset.outputs.has_changes == 'true'
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          aws cloudformation set-stack-policy \
            --stack-name "${{ steps.appstackcheck.outputs.stack_name }}" \
            --stack-policy-body file://infra/cloudformation/policies/${ENV}-app-policy.json

  smoke-test:
    name: Smoke Test
    needs: [resolve-environment, deploy-app]
    runs-on: ubuntu-latest
    environment: ${{ needs.resolve-environment.outputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: "10.33.2"

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Get API endpoint
        id: api
        run: |
          ENV="${{ needs.resolve-environment.outputs.environment }}"
          ENDPOINT=$(aws cloudformation describe-stacks \
            --stack-name "processing-app-${ENV}" \
            --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`].OutputValue' \
            --output text)
          echo "endpoint=$ENDPOINT" >> "$GITHUB_OUTPUT"

      - name: Run smoke tests
        env:
          ENVIRONMENT: ${{ needs.resolve-environment.outputs.environment }}
          API_ENDPOINT: ${{ steps.api.outputs.endpoint }}
        run: pnpm e2e:smoke

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: smoke-test-report-${{ needs.resolve-environment.outputs.environment }}
          path: packages/e2e/test-results/
```

---

## 14. Drift Detection

### 14.1 Drift Check Workflow (.github/workflows/drift-check.yml)

```yaml
name: Drift Detection

on:
  schedule:
    - cron: "0 6 * * *" # Daily at 6 AM UTC
  workflow_dispatch:
    inputs:
      environment:
        description: Environment to check (leave empty for all)
        required: false
        type: choice
        options: ["", dev, stage, prod]

env:
  AWS_REGION: us-east-1

jobs:
  drift-check:
    name: Check ${{ matrix.environment }} - ${{ matrix.stack }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        environment: ${{ inputs.environment != '' && fromJson(format('["{0}"]', inputs.environment)) || fromJson('["dev", "stage", "prod"]') }}
        stack: [foundation, app]
    environment: ${{ matrix.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Start drift detection
        id: detect
        run: |
          STACK_NAME="processing-${{ matrix.stack }}-${{ matrix.environment }}"
          DETECTION_ID=$(aws cloudformation detect-stack-drift \
            --stack-name "$STACK_NAME" \
            --query 'StackDriftDetectionId' \
            --output text)
          echo "detection_id=$DETECTION_ID" >> "$GITHUB_OUTPUT"
          echo "stack_name=$STACK_NAME" >> "$GITHUB_OUTPUT"

      - name: Wait for detection
        run: |
          while true; do
            STATUS=$(aws cloudformation describe-stack-drift-detection-status \
              --stack-drift-detection-id "${{ steps.detect.outputs.detection_id }}" \
              --query 'DetectionStatus' \
              --output text)

            if [[ "$STATUS" == "DETECTION_COMPLETE" ]]; then
              break
            elif [[ "$STATUS" == "DETECTION_FAILED" ]]; then
              echo "::error::Drift detection failed"
              exit 1
            fi

            echo "Detection status: $STATUS. Waiting..."
            sleep 10
          done

      - name: Check drift status
        id: drift
        run: |
          DRIFT_STATUS=$(aws cloudformation describe-stack-drift-detection-status \
            --stack-drift-detection-id "${{ steps.detect.outputs.detection_id }}" \
            --query 'StackDriftStatus' \
            --output text)

          echo "drift_status=$DRIFT_STATUS" >> "$GITHUB_OUTPUT"

          if [[ "$DRIFT_STATUS" != "IN_SYNC" ]]; then
            echo "::error::Stack ${{ steps.detect.outputs.stack_name }} has drifted: $DRIFT_STATUS"

            # Get detailed drift info
            aws cloudformation describe-stack-resource-drifts \
              --stack-name "${{ steps.detect.outputs.stack_name }}" \
              --stack-resource-drift-status-filters MODIFIED DELETED \
              > drift-details.json

            cat drift-details.json | jq .
            exit 1
          fi

          echo "✅ Stack ${{ steps.detect.outputs.stack_name }} is IN_SYNC"

      - name: Upload drift artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: drift-${{ matrix.environment }}-${{ matrix.stack }}
          path: drift-details.json
          if-no-files-found: ignore

      - name: Write job summary
        if: always()
        run: |
          cat >> "$GITHUB_STEP_SUMMARY" << EOF
          ## Drift Check: ${{ steps.detect.outputs.stack_name }}

          | Property | Value |
          | -------- | ----- |
          | Stack | \`${{ steps.detect.outputs.stack_name }}\` |
          | Environment | ${{ matrix.environment }} |
          | Status | ${{ steps.drift.outputs.drift_status }} |
          | Detection ID | \`${{ steps.detect.outputs.detection_id }}\` |
          EOF
```

---

## 15. Deployment

### 15.1 Environment Progression

```
dev  →  stage  →  prod
  │        │        │
  │        │        └── Push to main (manual approval in GitHub Environment)
  │        └── Push to stage
  └── Push to dev
```

Branch-bound deployments:

| Branch | Environment | Trigger                                           |
| ------ | ----------- | ------------------------------------------------- |
| dev    | dev         | Automatic on push                                 |
| stage  | stage       | Automatic on push                                 |
| main   | prod        | Automatic on push (requires environment approval) |

### 15.2 Deployment Guardrails

#### Change Set Review

Before executing any CloudFormation change set, the pipeline:

1. Creates the change set
2. Waits for change set to be ready
3. Inspects for dangerous changes (DELETE, REPLACE on retained resources)
4. Fails if dangerous changes detected
5. Executes only if safe

```bash
# Check for dangerous changes
CHANGES=$(aws cloudformation describe-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGE_SET_NAME" \
  --query 'Changes[?ResourceChange.Replacement==`True` || ResourceChange.Action==`Delete`]')

if [[ "$CHANGES" != "[]" ]]; then
  echo "::error::Change set contains DELETE or REPLACE operations"
  exit 1
fi
```

#### Stack Policies

After deployment, apply stack policies to protect retained resources:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/ProcessingJobsTable"
    },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/ProcessingTasksTable"
    },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/ProcessingBucket"
    },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/WikisBucket"
    },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/AppEnvSecret"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

Apply with:

```bash
aws cloudformation set-stack-policy \
  --stack-name "$STACK_NAME" \
  --stack-policy-body file://infra/cloudformation/policies/${ENV}-foundation-policy.json
```

### 15.3 Feature Flags

Runtime configuration in Secrets Manager:

```json
{
  "BEDROCK_MODEL_ID": "qwen.qwen3-coder-30b-a3b-v1:0",
  "BEDROCK_MAX_TOKENS": "8192",
  "BEDROCK_TEMPERATURE": "0.1",
  "MAX_CONCURRENT_WORKFLOWS": "10",
  "FEATURE_FLAG_PHASE3_WIKI_MAINTENANCE": "true",
  "FEATURE_FLAG_CONTRADICTION_DETECTION": "true",
  "FEATURE_FLAG_JUDGE_QUALITY_GATE": "false"
}
```

**Model configuration is config-driven:**

| Config Key            | Default                         | Purpose                  |
| --------------------- | ------------------------------- | ------------------------ |
| `BEDROCK_MODEL_ID`    | `qwen.qwen3-coder-30b-a3b-v1:0` | Bedrock model identifier |
| `BEDROCK_MAX_TOKENS`  | `8192`                          | Max output tokens        |
| `BEDROCK_TEMPERATURE` | `0.1`                           | Model temperature        |

To switch models (e.g., to Claude for higher context needs), update Secrets Manager:

```json
{
  "BEDROCK_MODEL_ID": "anthropic.claude-3-5-sonnet-20241022-v2:0",
  "BEDROCK_MAX_TOKENS": "8192",
  "BEDROCK_TEMPERATURE": "0.1"
}
```

Feature flags allow:

- **Model switching** without code deployment (config-driven)
- Gating Phase 3 wiki writes during testing
- Controlling workflow concurrency
- Enabling/disabling quality gates without redeployment

### 15.4 AWS Profile Mapping

| Environment | AWS CLI Profile | GitHub Environment |
| ----------- | --------------- | ------------------ |
| dev         | sdai-dev        | dev                |
| stage       | sdai-stage      | stage              |
| prod        | sdai-prod       | prod               |

---

## 16. Cost Estimation

> **Note**: Model IDs, rate limits, and pricing are initial assumptions based on current AWS documentation.
> Verify these values in the target account before production deployment, as they vary by account/region
> and change over time.

### 16.1 Bedrock Model Pricing

Default model: **Qwen3-Coder 30b** (significantly cheaper than Claude)

| Model                         | Input (per 1K tokens) | Output (per 1K tokens) | Notes                       |
| ----------------------------- | --------------------- | ---------------------- | --------------------------- |
| **qwen3-coder-30b** (default) | $0.00035              | $0.0014                | ~10x cheaper than Claude    |
| claude-3-5-sonnet             | $0.003                | $0.015                 | Higher context, higher cost |
| claude-3-haiku                | $0.00025              | $0.00125               | Fast, cheap, less capable   |

### 16.2 Per-PDF Processing Cost (Qwen3-Coder 30b)

| Component                   | Unit Cost            | Units per PDF | Cost per PDF |
| --------------------------- | -------------------- | ------------- | ------------ |
| Lambda (Phase 0)            | $0.0000166/GB-s      | 15 GB-s       | $0.00025     |
| Lambda (Phase 1-2)          | $0.0000166/GB-s      | 100 GB-s      | $0.00166     |
| Bedrock (extraction input)  | $0.00035/1K tokens   | 50K tokens    | $0.0175      |
| Bedrock (extraction output) | $0.0014/1K tokens    | 15K tokens    | $0.021       |
| Bedrock (synthesis input)   | $0.00035/1K tokens   | 20K tokens    | $0.007       |
| Bedrock (synthesis output)  | $0.0014/1K tokens    | 20K tokens    | $0.028       |
| S3 storage                  | $0.023/GB/month      | 5 MB          | $0.00012     |
| DynamoDB                    | $1.25/million writes | 50 writes     | $0.00006     |
| **Total**                   |                      |               | **~$0.08**   |

### 16.3 60-PDF Batch Cost

| Scenario                      | Cost (Qwen) | Cost (Claude) | Notes                  |
| ----------------------------- | ----------- | ------------- | ---------------------- |
| 60 small PDFs (< 50 pages)    | ~$5         | ~$27          | Mostly LLM costs       |
| 60 medium PDFs (50-200 pages) | ~$10        | ~$50          | More extraction chunks |
| 60 large PDFs (200+ pages)    | ~$20        | ~$100         | ECS for Phase 0        |

### 16.4 Monthly Steady-State

| Usage           | Cost (Qwen) | Cost (Claude) |
| --------------- | ----------- | ------------- |
| 100 PDFs/month  | ~$10        | ~$50          |
| 500 PDFs/month  | ~$50        | ~$250         |
| 1000 PDFs/month | ~$100       | ~$500         |

---

## Appendix A: Migration from Local Pipeline

The existing `wiki_ingest.py` pipeline maps to this design:

| Local Component                  | Cloud Equivalent                        |
| -------------------------------- | --------------------------------------- |
| `wiki_phase0_import.py`          | phase0-normalize Lambda                 |
| `wiki_deep_extract.py`           | phase1-extract-chunk + aggregate Lambda |
| `wiki_phase2_single.py`          | phase2-synthesize-page Lambda           |
| `wiki_phase3_finalize.py`        | phase3-maintain-wiki Lambda             |
| `.wiki-extraction-state/`        | S3 `processing/{fileVersionId}/`        |
| `wiki/`                          | S3 `wikis/{folderId}/`                  |
| `tools/wiki_model_defaults.json` | Parameter Store / env vars              |

---

## Appendix B: Open Questions for Product

1. **Top-level files**: Reject at upload, or assign to a default personal wiki?
2. **File deletion**: What happens to wiki pages citing a deleted source?
3. **User-facing status**: What processing states should Zarya display?
4. **Contradiction resolution**: Auto-resolve or require user action?
5. **Wiki sharing**: Can wikis span multiple folders or users?

---

## Appendix C: Future Enhancements

1. **Query endpoint**: Allow users to ask questions against wikis
2. **Scheduled maintenance**: Periodic lint and contradiction checks
3. **Cross-folder knowledge**: Link related concepts across user wikis
