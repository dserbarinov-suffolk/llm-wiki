# Feature 00: Foundation Infrastructure

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - All persisted artifacts must be idempotent by `fileVersionId` or deterministic event ID.
> - Derived data has the same sensitivity classification as source content.

---

## Goal

Create durable buckets, tables, secrets, queues, and layers as the baseline infrastructure.
No Zarya stream consumption. No LLM processing.

---

## Scope

### Included

- Processing S3 bucket with lifecycle rules
- Wiki S3 bucket with versioning
- ProcessingJobs DynamoDB table with byFolder GSI
- ProcessingTasks DynamoDB table with byFileVersion GSI
- WikiLocks DynamoDB table with TTL
- RateLimits DynamoDB table
- Application configuration secret in Secrets Manager
- Intake SQS queue with DLQ
- Wiki maintenance FIFO queue with DLQ
- DLQ alarms (CloudWatch)
- Lambda layers (pymupdf4llm, wiki-core, wiki-llm)
- Deployment artifacts bucket

### Excluded

- Stream event source mappings (Feature 01)
- Lambda functions (Feature 01+)
- Step Functions state machine (Feature 01)
- API Gateway (Feature 07)

---

## Preconditions

- AWS account and environment (dev/stage/prod) exist
- Zarya source bucket name is known (for IAM policies)
- Bedrock model ID is selected

---

## Postconditions

After deployment:

| Resource                    | State                                    |
| --------------------------- | ---------------------------------------- |
| Processing bucket           | Exists, lifecycle rules active           |
| Wiki bucket                 | Exists, versioning enabled               |
| ProcessingJobs table        | Exists, GSI active, on-demand capacity   |
| ProcessingTasks table       | Exists, GSI active, TTL enabled          |
| WikiLocks table             | Exists, TTL enabled                      |
| RateLimits table            | Exists, seeded with default model config |
| AppEnvSecret                | Exists, contains BEDROCK_MODEL_ID        |
| Intake queue + DLQ          | Exist                                    |
| Wiki maintenance FIFO + DLQ | Exist                                    |
| DLQ alarms                  | Configured, SNS topic exists             |
| Lambda layers               | Published, version ARNs exported         |
| Artifacts bucket            | Exists, versioning enabled               |

---

## Interfaces Consumed

None (foundation has no upstream dependencies within processing service).

## Interfaces Produced

| Interface                    | Location                                                       |
| ---------------------------- | -------------------------------------------------------------- |
| ProcessingJobs table schema  | [`../interfaces/data-models.md`](../interfaces/data-models.md) |
| ProcessingTasks table schema | [`../interfaces/data-models.md`](../interfaces/data-models.md) |
| WikiLocks table schema       | [`../interfaces/data-models.md`](../interfaces/data-models.md) |
| RateLimits table schema      | [`../interfaces/data-models.md`](../interfaces/data-models.md) |
| Processing bucket layout     | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md)     |
| Wiki bucket layout           | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md)     |

---

## Functional Requirements

### S3 Buckets

1. Processing bucket must have 30-day lifecycle expiration for all objects.
2. Wiki bucket must have versioning enabled for object-level history.
3. Both buckets must block public access.
4. Both buckets must use SSE-S3 encryption (upgradeable to SSE-KMS if required).

### DynamoDB Tables

1. All tables must use on-demand capacity mode.
2. ProcessingJobs must have byFolder GSI (partition: folderId, sort: createdAt).
3. ProcessingTasks must have byFileVersion GSI (partition: fileVersionId, sort: taskId).
4. WikiLocks must have TTL enabled on `ttl` attribute.
5. ProcessingTasks should have TTL enabled on `ttl` attribute (30-day retention).

### SQS Queues

1. Intake queue must have visibility timeout of 900s (15 min).
2. Intake queue must have DLQ with maxReceiveCount=3.
3. Wiki maintenance queue must be FIFO with content-based deduplication disabled.
4. Wiki maintenance queue must have visibility timeout of 900s.
5. Wiki maintenance queue must have DLQ with maxReceiveCount=3.

### Secrets

1. AppEnvSecret must contain `BEDROCK_MODEL_ID` with default value.
2. Secret must be accessible by Lambda execution roles.

### Alarms

1. DLQ message count alarm must trigger on > 0 messages.
2. Alarm must publish to SNS topic for ops notification.

### Lambda Layers

1. pymupdf4llm layer must include pymupdf and pymupdf4llm packages.
2. wiki-core layer must include wiki_core package.
3. wiki-llm layer must include wiki_llm, boto3, and tiktoken packages.
4. All layers must be compatible with python3.12 runtime.

---

## Failure Handling

| Failure               | Handling                                |
| --------------------- | --------------------------------------- |
| Stack creation fails  | Rollback, no partial resources          |
| Bucket already exists | Fail (bucket names are globally unique) |
| Table schema mismatch | Fail (require explicit migration)       |

---

## Security Requirements

1. S3 buckets must block public access.
2. S3 buckets must use server-side encryption.
3. DynamoDB tables must use encryption at rest.
4. Secrets Manager secret must use default KMS encryption.
5. IAM roles must follow least-privilege principle.

---

## Observability Requirements

| Metric/Alarm               | Threshold | Action        |
| -------------------------- | --------- | ------------- |
| IntakeDLQMessages          | > 0       | Alert via SNS |
| WikiMaintenanceDLQMessages | > 0       | Alert via SNS |

---

## Verification Plan

1. **Stack deploys successfully**
   - `aws cloudformation describe-stacks --stack-name processing-foundation-{env}`
   - Status: `CREATE_COMPLETE` or `UPDATE_COMPLETE`

2. **Buckets are accessible**

   ```bash
   aws s3 ls s3://processing-{env}-{account}/
   aws s3 ls s3://wikis-{env}-{account}/
   ```

3. **Tables accept read/write**

   ```bash
   aws dynamodb put-item --table-name processing-jobs-{env} \
     --item '{"fileVersionId": {"S": "test-123"}, "status": {"S": "test"}}'
   aws dynamodb get-item --table-name processing-jobs-{env} \
     --key '{"fileVersionId": {"S": "test-123"}}'
   aws dynamodb delete-item --table-name processing-jobs-{env} \
     --key '{"fileVersionId": {"S": "test-123"}}'
   ```

4. **Queues exist and are accessible**

   ```bash
   aws sqs get-queue-attributes --queue-url {intake-queue-url} --attribute-names All
   aws sqs get-queue-attributes --queue-url {wiki-maintenance-queue-url} --attribute-names All
   ```

5. **Layers are published**

   ```bash
   aws lambda list-layer-versions --layer-name pymupdf4llm-{env}
   aws lambda list-layer-versions --layer-name wiki-core-{env}
   aws lambda list-layer-versions --layer-name wiki-llm-{env}
   ```

6. **Secret exists**
   ```bash
   aws secretsmanager get-secret-value --secret-id processing-app-config-{env}
   ```

---

## Rollout / Rollback

### Rollout

1. Deploy to dev first, verify all postconditions.
2. Deploy to stage, run integration tests.
3. Deploy to prod with change set review.

### Rollback

- Stack deletion removes all resources except those with `DeletionPolicy: Retain`.
- Retained resources: ProcessingJobs table, ProcessingTasks table, Processing bucket, Wiki bucket.
- Manual cleanup required for retained resources if full rollback needed.

### Retained Resources

The following resources must have `DeletionPolicy: Retain`:

- ProcessingJobs table (audit trail)
- ProcessingTasks table (debugging data)
- Processing bucket (artifacts for in-flight jobs)
- Wiki bucket (wiki content)
- AppEnvSecret (configuration)

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
