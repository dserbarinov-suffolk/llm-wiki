# Feature 01: Intake and Job Tracking

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)
> defines the canonical trigger, table access rules, and filtering requirements.
>
> Relevant invariants:
>
> - `fileVersionId` is the trace ID and workflow idempotency key.
> - `folderId` is the wiki aggregate key and FIFO MessageGroupId.
> - Top-level files (`parentFolderId === null`) are skipped in V1.

---

## Goal

Consume Zarya FileVersions stream, create job records, and start Step Functions executions idempotently. The workflow may stop after validation in early rollout.

---

## Scope

### Included

- Stream Dispatcher Lambda (DynamoDB stream trigger)
- Intake Consumer Lambda (SQS trigger)
- Step Functions state machine (minimal: ValidateAndEnrich → terminal states)
- Event source mapping for FileVersions stream
- Job creation in ProcessingJobs table
- Idempotent workflow start via execution name

### Excluded

- Files stream for deletion (Feature 06)
- PDF conversion (Feature 02)
- LLM processing (Features 03-04)
- Wiki maintenance (Feature 05)

---

## Preconditions

- Foundation (Feature 00) deployed
- Zarya FileVersions stream enabled and ARN available
- Zarya Files table readable for folderId and currentVersionId

---

## Postconditions

After deployment:

| Condition                               | Verification                            |
| --------------------------------------- | --------------------------------------- |
| Valid PDF/MD creates one job            | Query ProcessingJobs by fileVersionId   |
| Duplicate stream delivery is idempotent | Second event does not create execution  |
| Non-current versions are skipped        | Logs show skip message, no job created  |
| Top-level files are skipped             | Logs show skip message, no job created  |
| Unsupported content types are skipped   | Logs show skip message, metrics emitted |
| Job status is queryable                 | GET /processing/jobs/{fileVersionId}    |

---

## Interfaces Consumed

| Interface                    | Location                                                       |
| ---------------------------- | -------------------------------------------------------------- |
| FileVersions DynamoDB stream | Zarya infrastructure                                           |
| Files DynamoDB table         | Zarya infrastructure (for folderId lookup)                     |
| ProcessingJobs table schema  | [`../interfaces/data-models.md`](../interfaces/data-models.md) |

## Interfaces Produced

| Interface                 | Location                                                         |
| ------------------------- | ---------------------------------------------------------------- |
| FileVersionCommittedEvent | [`../interfaces/events.md`](../interfaces/events.md)             |
| StepFunctionsInput        | [`../interfaces/events.md`](../interfaces/events.md)             |
| Job status values         | [`../interfaces/status-model.md`](../interfaces/status-model.md) |

---

## Functional Requirements

### Stream Dispatcher

> Per the integration contract, the dispatcher must validate before routing.

1. Filter INSERT events only (not MODIFY or REMOVE).
2. Validate `objectKey` starts with `objects/` (defense against incomplete uploads).
3. Filter by content type: `application/pdf`, `text/markdown`, `text/x-markdown`.
4. Filter by size: ≤ 250MB.
5. Load `File` record from Zarya Files table to get `folderId` and `currentVersionId`.
6. Require `File.isDeleted === false` (skip deleted files).
7. Require `File.currentVersionId === fileVersionId` (skip non-current versions).
8. Skip files where `parentFolderId === null` (top-level files without folder/wiki target).
9. Emit FileVersionCommittedEvent to Intake Queue with resolved `folderId`.

### Intake Consumer

1. Parse FileVersionCommittedEvent from SQS message.
2. Create job record in ProcessingJobs table with status `queued`.
3. Start Step Functions execution with name = fileVersionId.
4. Handle `ExecutionAlreadyExists` gracefully (idempotent).
5. Delete SQS message on success (Lambda handles automatically).

### Step Functions (Minimal)

1. Accept StepFunctionsInput shape.
2. Validate source object exists (HeadObject).
3. Compare observed S3 ETag with `sourceEtagOrChecksum` (integrity check).
4. On validation failure, transition to MarkFailed.
5. On validation success, transition to terminal state (for now).

---

## Failure Handling

| Failure                      | Handling                                 |
| ---------------------------- | ---------------------------------------- |
| Files table lookup fails     | Retry via SQS visibility timeout         |
| File.isDeleted is true       | Skip (log and return success)            |
| Non-current version detected | Skip (log and return success)            |
| Execution already exists     | Idempotent success (job already tracked) |
| Source object missing        | Fail job with SOURCE_OBJECT_MISSING      |
| ETag mismatch                | Fail job with SOURCE_INTEGRITY_ERROR     |
| Transient AWS errors         | Retry with exponential backoff           |

---

## Security Requirements

1. Stream Dispatcher must have read access to Zarya FileVersions stream.
2. Stream Dispatcher must have read access to Zarya Files table.
3. Intake Consumer must have write access to ProcessingJobs table.
4. Intake Consumer must have StartExecution permission on state machine.
5. No sensitive data logged (file contents, user PII).

---

## Observability Requirements

| Metric                          | Dimension   | Trigger                     |
| ------------------------------- | ----------- | --------------------------- |
| `jobs.created`                  | environment | Job record created          |
| `jobs.skipped.non_current`      | environment | Non-current version skipped |
| `jobs.skipped.top_level`        | environment | Top-level file skipped      |
| `jobs.skipped.unsupported_type` | environment | Content type filtered       |
| `jobs.skipped.size_exceeded`    | environment | Size > 250MB                |

| Log Field       | Required | Example                      |
| --------------- | -------- | ---------------------------- |
| `fileVersionId` | Yes      | `"v-123"`                    |
| `folderId`      | Yes      | `"f-456"`                    |
| `action`        | Yes      | `"job_created"`, `"skipped"` |
| `skipReason`    | If skip  | `"non_current_version"`      |

---

## Verification Plan

1. **Inject synthetic stream event**
   - Create test FileVersion record in Zarya (or mock stream event)
   - Verify SQS message appears in Intake Queue

2. **Verify workflow starts**
   - Process SQS message
   - Verify Step Functions execution exists with name = fileVersionId
   - Verify job record exists with status `queued`

3. **Verify idempotency**
   - Inject duplicate stream event
   - Verify no second execution created
   - Verify `ExecutionAlreadyExists` logged

4. **Verify filtering**
   - Inject event with wrong content type → skipped
   - Inject event with size > 250MB → skipped
   - Inject event for non-current version → skipped
   - Inject event for top-level file → skipped

5. **Verify source validation**
   - Start workflow with valid source → proceeds
   - Start workflow with missing source → fails with SOURCE_OBJECT_MISSING

---

## Rollout / Rollback

### Rollout

1. Deploy Lambda functions and state machine (disabled mapping).
2. Run manual tests with synthetic events.
3. Enable event source mapping.
4. Monitor for 24 hours before proceeding to Feature 02.

### Rollback

1. Disable event source mapping.
2. Drain Intake Queue (move messages to DLQ or process manually).
3. Rollback Lambda/state machine if needed.

### Feature Flag

The event source mapping can be disabled without code deployment:

```bash
aws lambda update-event-source-mapping \
  --uuid {mapping-uuid} \
  --enabled false
```

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
