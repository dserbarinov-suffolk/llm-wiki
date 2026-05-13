# Deployment Requirements

> Requirements for infrastructure deployment. Actual CloudFormation/CDK templates live in `infra/`.

---

## Stack Boundaries

| Stack                 | Purpose                                      | Lifecycle          |
| --------------------- | -------------------------------------------- | ------------------ |
| processing-foundation | Durable resources (buckets, tables, secrets) | Rarely updated     |
| processing-app        | Runtime resources (Lambda, Step Functions)   | Frequently updated |

### Foundation Stack Resources

| Resource Type   | Resource                    | Retention        |
| --------------- | --------------------------- | ---------------- |
| S3 Bucket       | Processing bucket           | Retain on delete |
| S3 Bucket       | Wiki bucket                 | Retain on delete |
| S3 Bucket       | Deployment artifacts bucket | Retain on delete |
| DynamoDB Table  | ProcessingJobs              | Retain on delete |
| DynamoDB Table  | ProcessingTasks             | Retain on delete |
| DynamoDB Table  | WikiLocks                   | Delete OK        |
| DynamoDB Table  | RateLimits                  | Delete OK        |
| Secrets Manager | AppEnvSecret                | Retain on delete |
| SNS Topic       | Alert topic                 | Delete OK        |
| Lambda Layer    | pymupdf4llm                 | Versioned        |
| Lambda Layer    | wiki-core                   | Versioned        |
| Lambda Layer    | wiki-llm                    | Versioned        |

### App Stack Resources

| Resource Type      | Resource              | Retention |
| ------------------ | --------------------- | --------- |
| SQS Queue          | Intake queue          | Delete OK |
| SQS Queue          | Intake DLQ            | Delete OK |
| SQS Queue          | Wiki maintenance FIFO | Delete OK |
| SQS Queue          | Wiki maintenance DLQ  | Delete OK |
| Lambda Function    | All functions         | Delete OK |
| Step Functions     | State machine         | Delete OK |
| API Gateway        | HTTP API              | Delete OK |
| EventSourceMapping | Stream → Dispatcher   | Delete OK |
| CloudWatch Alarm   | All alarms            | Delete OK |

---

## Retention Requirements

### Retained Resources

The following resources must have `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`:

- ProcessingJobs table (audit trail)
- ProcessingTasks table (debugging data)
- Processing bucket (artifacts for in-flight jobs)
- Wiki bucket (wiki content)
- AppEnvSecret (configuration)

### Data Retention

| Data                 | Retention  | Mechanism           |
| -------------------- | ---------- | ------------------- |
| Processing artifacts | 30 days    | S3 lifecycle rule   |
| Wiki content         | Indefinite | No expiration       |
| Job records          | Indefinite | No TTL              |
| Task records         | 30 days    | DynamoDB TTL        |
| Wiki locks           | 10 minutes | DynamoDB TTL        |
| CloudWatch Logs      | 30 days    | Log group retention |

---

## Protected Resources

Stack policies must prevent accidental deletion/replacement of:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": [
        "LogicalResourceId/ProcessingJobsTable",
        "LogicalResourceId/ProcessingTasksTable",
        "LogicalResourceId/ProcessingBucket",
        "LogicalResourceId/WikisBucket",
        "LogicalResourceId/AppEnvSecret"
      ]
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

---

## Deployment Ordering

| Step | Stack/Change                | Depends On             |
| ---- | --------------------------- | ---------------------- |
| 1    | Zarya FileVersions stream   | Zarya foundation stack |
| 2    | Zarya Files stream          | Zarya foundation stack |
| 3    | Zarya stream ARN exports    | Steps 1-2              |
| 4    | Processing foundation stack | AWS account exists     |
| 5    | Processing app stack        | Steps 3-4              |

### Notes

- Foundation stack can deploy before Zarya streams exist.
- App stack requires Zarya streams (creates event source mappings).
- Feature flags allow gradual enablement after deployment.

---

## Required Parameters

### Foundation Stack

| Parameter        | Required | Description                            |
| ---------------- | -------- | -------------------------------------- |
| Environment      | Yes      | dev, stage, prod                       |
| ZaryaFilesBucket | Yes      | Zarya's committed objects bucket name  |
| BedrockModelId   | No       | Default: qwen.qwen3-coder-30b-a3b-v1:0 |

### App Stack

| Parameter                  | Required | Description                   |
| -------------------------- | -------- | ----------------------------- |
| Environment                | Yes      | dev, stage, prod              |
| ArtifactVersion            | Yes      | Git SHA or version tag        |
| ZaryaFileVersionsStreamArn | Yes      | Zarya FileVersions stream ARN |
| ZaryaFilesStreamArn        | Yes      | Zarya Files stream ARN        |
| ZaryaFilesTableArn         | Yes      | Zarya Files table ARN         |
| ZaryaFilesTableName        | Yes      | Zarya Files table name        |
| BedrockModelId             | No       | Override foundation default   |
| Phase1Concurrency          | No       | Default: 5                    |
| Phase2Concurrency          | No       | Default: 3                    |

---

## Concurrency Limits

> **Source**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)

Initial concurrency should be configurable, not hard-coded. Conservative starting points:

| Phase   | Suggested Starting Limit   | Rationale                                                          |
| ------- | -------------------------- | ------------------------------------------------------------------ |
| Phase 0 | 5-10 PDFs                  | Raise only after measuring CPU, memory, timeout, ephemeral storage |
| Phase 1 | 20-100 slug tasks globally | Derive from LLM requests-per-minute and tokens-per-minute          |
| Phase 2 | 20-100 slug tasks globally | Separate limit from Phase 1 if model or prompt differs             |
| Phase 3 | 1 per folder/wiki          | Use FIFO MessageGroupId = folderId or DynamoDB lock                |

### Backlog-Driven Processing

For the 60-PDF batch scenario, expected behavior:

1. 60 file-version events arrive quickly
2. Dispatcher creates or confirms 60 processing jobs
3. Phase 0 drains at CPU-safe concurrency
4. Phase 1 and Phase 2 drain at LLM-rate-safe concurrency
5. Phase 3 runs only after prerequisites complete and folder wiki lock is available

---

## Lambda Configuration

Initial resource allocations. Adjust based on observed metrics.

| Lambda             | Memory (MB) | Timeout (s) | Ephemeral (MB) | Notes                          |
| ------------------ | ----------- | ----------- | -------------- | ------------------------------ |
| Stream Dispatcher  | 256         | 30          | 512 (default)  | Light DynamoDB reads           |
| Intake Consumer    | 256         | 60          | 512 (default)  | SQS + Step Functions start     |
| Phase 0 Normalize  | 2048        | 300         | 2048           | PDF extraction is memory-heavy |
| Phase 1 Extract    | 1024        | 120         | 512 (default)  | Bedrock call per chunk         |
| Phase 1 Aggregate  | 1024        | 60          | 512 (default)  | In-memory claim dedup          |
| Phase 2 Synthesize | 1024        | 180         | 512 (default)  | Bedrock call per page          |
| Phase 2 Filter     | 256         | 30          | 512 (default)  | Light filtering                |
| Phase 3 Wiki       | 1024        | 300         | 512 (default)  | S3 reads/writes                |
| Wiki API           | 512         | 30          | 512 (default)  | Read-only API                  |

### Timeout Rationale

- **Phase 0 (300s)**: PDF conversion can take 2-3 minutes for large documents
- **Phase 1/2 (120-180s)**: Bedrock calls typically complete in 30-60s; extra buffer for retries
- **Phase 3 (300s)**: Multiple S3 operations for wiki maintenance

### Memory Rationale

- **Phase 0 (2048 MB)**: pymupdf4llm loads full PDF into memory
- **Phase 1/2 (1024 MB)**: Comfortable buffer for prompt construction and response parsing
- **Others (256-512 MB)**: Minimal compute, mostly I/O

---

## Large Source Size Handling

> **Source**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)

### Size Thresholds

| Source Size | Upload Path          | Phase 0 Path               | Notes                                       |
| ----------- | -------------------- | -------------------------- | ------------------------------------------- |
| < 100 MB    | Single presigned PUT | Lambda worker              | Still avoid loading whole file into memory  |
| 100-250 MB  | Multipart upload     | Container worker preferred | Explicit /tmp, memory, timeout, concurrency |
| > 250 MB    | Reject at upload     | Do not process             | Report unsupported size error               |

### Rules

- Large PDFs are object references, never queue/Lambda/Step Functions payloads
- Pass `fileVersionId`, `objectKey`, `sizeBytes` only—not bytes
- 250 MB is proposed product maximum for automatic processing
- Phase 0 should record preflight metadata before conversion

---

## Artifact Naming Contract

Build artifacts must match these names exactly (referenced by CloudFormation S3Key):

```
dist/
├── lambda/
│   ├── node/
│   │   ├── stream-dispatcher.zip
│   │   └── intake-consumer.zip
│   └── python/
│       ├── phase0-normalize.zip
│       ├── phase1-extract.zip
│       ├── phase1-aggregate.zip
│       ├── phase2-synthesize.zip
│       ├── phase3-wiki.zip
│       └── filter-phase2.zip
├── layers/
│   ├── pymupdf4llm-layer.zip
│   ├── wiki-core-layer.zip
│   └── wiki-llm-layer.zip
└── step-functions/
    └── processing-workflow.asl.json
```

Upload paths in artifacts bucket:

```
lambda/${ArtifactVersion}/<name>.zip
layers/${ArtifactVersion}/<name>.zip
sfn/${ArtifactVersion}/workflow.asl.json
```

---

## Rollback Requirements

### Safe Rollback

1. App stack can be rolled back without data loss.
2. Foundation stack should not need rollback (durable resources).
3. In-flight workflows may fail on rollback (expected).

### Rollback Procedure

1. Disable event source mappings (stop new workflows).
2. Wait for in-flight workflows to complete or timeout.
3. Rollback CloudFormation stack.
4. Re-enable event source mappings with previous version.

### Rollback Triggers

- Smoke test failures
- Error rate > 20% in first 30 minutes
- Critical alarm firing

---

## Environment Progression

```
dev  →  stage  →  prod
```

| Environment | Branch | Trigger                               |
| ----------- | ------ | ------------------------------------- |
| dev         | dev    | Automatic on push                     |
| stage       | stage  | Automatic on push                     |
| prod        | main   | Automatic on push (requires approval) |

---

## Changelog

| Date       | Change                         |
| ---------- | ------------------------------ |
| 2026-05-13 | Initial requirements extracted |
