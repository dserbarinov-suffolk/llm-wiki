# Feature 08: Observability and Operations

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - `fileVersionId` is the trace ID for all ingest operations.
> - `folderId` is the wiki aggregate key.

---

## Goal

Make the service operable with structured logging, metrics, alarms, and runbooks.

---

## Scope

### Included

- Structured logging with trace IDs
- CloudWatch metrics (custom and Lambda)
- DLQ alarms
- Failure rate alarm with volume guard
- Reconciliation alert for callback timeouts
- Runbooks for common issues

### Excluded

- X-Ray distributed tracing (use fileVersionId as correlation ID)
- Custom dashboards (CloudWatch automatic dashboards sufficient for V1)

---

## Preconditions

- Features 00-07 deployed
- SNS alert topic exists
- CloudWatch Logs configured

---

## Postconditions

After deployment:

| Capability            | State                                     |
| --------------------- | ----------------------------------------- |
| Structured logs       | All Lambdas emit JSON logs with trace IDs |
| Terminal job metrics  | Emitted once per job completion           |
| DLQ alarms            | Fire on > 0 messages                      |
| Failure rate alarm    | Fire on > 10% failures with volume guard  |
| Reconciliation alerts | Fire on callback timeout                  |
| Runbooks              | Published to wiki or ops docs             |

---

## Interfaces Consumed

| Interface            | Location                                                       |
| -------------------- | -------------------------------------------------------------- |
| All Lambda functions | Features 01-07                                                 |
| SNS alert topic      | Foundation (Feature 00)                                        |
| ProcessingJobs table | [`../interfaces/data-models.md`](../interfaces/data-models.md) |

## Interfaces Produced

None (observability is cross-cutting).

---

## Functional Requirements

### Structured Logging

Every log line must be JSON with these fields:

| Field          | Required  | Description                      |
| -------------- | --------- | -------------------------------- |
| `timestamp`    | Yes       | ISO 8601 timestamp               |
| `level`        | Yes       | INFO, WARN, ERROR                |
| `message`      | Yes       | Human-readable message           |
| `traceId`      | If ingest | `fileVersionId`                  |
| `folderId`     | If known  | Wiki aggregate key               |
| `phase`        | If known  | `phase0`, `phase1`, etc.         |
| `taskId`       | If task   | `{fileVersionId}#{phase}#{slug}` |
| `durationMs`   | If timed  | Operation duration               |
| `errorCode`    | If error  | Structured error code            |
| `errorMessage` | If error  | Human-readable error             |

### Custom Metrics

| Metric                       | Unit  | Dimensions                    | When Emitted          |
| ---------------------------- | ----- | ----------------------------- | --------------------- |
| `jobs.created`               | Count | environment                   | Job record created    |
| `jobs.terminal`              | Count | environment, status           | Job reaches terminal  |
| `phase0.duration_ms`         | Ms    | environment                   | Phase 0 completes     |
| `phase1.chunk.duration_ms`   | Ms    | environment, model            | Chunk extraction done |
| `phase1.chunk.input_tokens`  | Count | environment, model            | Chunk extraction done |
| `phase1.chunk.output_tokens` | Count | environment, model            | Chunk extraction done |
| `phase2.page.duration_ms`    | Ms    | environment, model            | Page synthesis done   |
| `phase3.duration_ms`         | Ms    | environment                   | Wiki maintenance done |
| `api.requests`               | Count | environment, endpoint, status | API request handled   |
| `api.latency_ms`             | Ms    | environment, endpoint         | API request handled   |

### Metric Emission Rule

**Terminal job metrics must be emitted exactly once per job**, when the job reaches a terminal state (succeeded, failed, partial, superseded). This is done by `MarkJobStatus` Lambda.

### Alarms

| Alarm                      | Metric                            | Threshold            | Period | Action       |
| -------------------------- | --------------------------------- | -------------------- | ------ | ------------ |
| IntakeDLQMessages          | ApproximateNumberOfMessages       | > 0                  | 1 min  | Alert SNS    |
| WikiMaintenanceDLQMessages | ApproximateNumberOfMessages       | > 0                  | 1 min  | Alert SNS    |
| HighFailureRate            | jobs.terminal (status=failed)     | > 10% of terminal    | 1 hour | Page on-call |
| LLMRateLimitExhaustion     | phase1.chunk.duration_ms (p99)    | > 60000 (1 min wait) | 5 min  | Alert Slack  |
| QueueBacklog               | ApproximateNumberOfMessages + Age | > 100, age > 30 min  | 5 min  | Alert Slack  |

### Failure Rate Alarm Volume Guard

The HighFailureRate alarm must have a volume guard:

```
ALARM when:
  (failed_jobs / total_terminal_jobs) > 0.10
  AND total_terminal_jobs >= 10
```

This prevents alerting on 1 failure out of 2 jobs.

### Reconciliation Alerts

When `TaskTimedOut` exception occurs in Phase 3 after wiki commit:

1. Publish to SNS alert topic.
2. Include fileVersionId, folderId, timestamp.
3. Alert message indicates wiki is committed but Step Functions failed.
4. Requires manual reconciliation (mark job as succeeded).

Implementation:

```python
def publish_reconciliation_alert(file_version_id: str, folder_id: str, reason: str):
    sns.publish(
        TopicArn=ALERT_TOPIC_ARN,
        Subject=f"[RECONCILIATION] Wiki committed but callback failed: {file_version_id}",
        Message=json.dumps({
            "type": "reconciliation_required",
            "fileVersionId": file_version_id,
            "folderId": folder_id,
            "reason": reason,
            "timestamp": datetime.utcnow().isoformat(),
            "action": "Manual intervention required. Wiki is committed. Mark job as succeeded.",
        }),
    )
```

---

## Failure Handling

| Situation             | Detection                | Response                    |
| --------------------- | ------------------------ | --------------------------- |
| DLQ message           | DLQ alarm fires          | Investigate, replay or drop |
| High failure rate     | Failure rate alarm fires | Investigate root cause      |
| Callback timeout      | Reconciliation alert     | Manual job status update    |
| Rate limit exhaustion | Long chunk duration p99  | Increase limits or throttle |

---

## Runbooks

### DLQ Message Handling

1. Query DLQ for message attributes.
2. Identify fileVersionId and phase from message.
3. Check job status and Step Functions execution.
4. Options:
   - Replay: Move message back to main queue
   - Drop: Delete message if job already terminal
   - Investigate: Check logs for root cause

### Reconciliation (Callback Timeout)

1. Receive reconciliation alert.
2. Verify wiki files are present in S3.
3. Update ProcessingJobs table:
   ```bash
   aws dynamodb update-item \
     --table-name processing-jobs-{env} \
     --key '{"fileVersionId": {"S": "{fvid}"}}' \
     --update-expression "SET #s = :s, completedAt = :t" \
     --expression-attribute-names '{"#s": "status"}' \
     --expression-attribute-values '{":s": {"S": "succeeded"}, ":t": {"S": "{timestamp}"}}'
   ```
4. Close alert ticket.

### High Failure Rate

1. Query recent failed jobs:
   ```bash
   aws dynamodb query \
     --table-name processing-jobs-{env} \
     --index-name byFolder \
     --key-condition-expression "status = :s" \
     --filter-expression "updatedAt > :t"
   ```
2. Check common errorCode patterns.
3. If Bedrock throttling: reduce concurrency or increase limits.
4. If source issues: check Zarya for problematic files.
5. If code bug: hotfix and deploy.

---

## Security Requirements

1. No PII in metrics or alarms.
2. Alert messages include trace IDs, not content.
3. SNS topic encrypted with KMS.
4. Log retention configured (30 days default).

---

## Observability Requirements

This feature IS the observability requirements. See above.

---

## Verification Plan

1. **Structured logging**
   - Process a file
   - Query CloudWatch Logs Insights by fileVersionId
   - Verify all phases logged with correct fields

2. **Metric emission**
   - Process files to terminal states
   - Query CloudWatch Metrics for jobs.terminal
   - Verify one metric per terminal job

3. **DLQ alarm**
   - Force a message to DLQ
   - Verify alarm fires
   - Verify SNS notification received

4. **Failure rate alarm**
   - Process 20 jobs, fail 5 (25%)
   - Verify alarm fires
   - Verify does NOT fire with 1/2 failures (volume guard)

5. **Reconciliation alert**
   - Simulate callback timeout
   - Verify SNS alert received
   - Follow runbook to resolve

---

## Rollout / Rollback

### Rollout

1. Deploy metric emission in Lambda functions.
2. Create alarms in CloudFormation.
3. Test alarm paths with synthetic failures.
4. Subscribe ops team to SNS topic.

### Rollback

1. Alarms can be disabled without code change.
2. Metric emission is low-cost, can remain.
3. Remove alarm resources if needed.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
