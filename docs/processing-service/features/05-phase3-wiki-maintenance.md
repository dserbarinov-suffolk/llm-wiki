# Feature 05: Phase 3 - Wiki Maintenance

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)
> defines wiki structure, required files, and maintenance rules.
>
> Relevant invariants:
>
> - `folderId` is the wiki aggregate key and FIFO MessageGroupId.
> - Step Functions owns terminal job status.
> - Phase 3 worker commits wiki state but does not decide final job status.
> - All wiki writes for a folder must serialize through `wiki-maintenance.fifo`.
> - All persisted artifacts must be idempotent by `fileVersionId`.

---

## Goal

Serialize wiki writes by folder, merge synthesized pages into wiki, send Step Functions callback, and handle superseded versions.

---

## Scope

### Included

- Phase 3 Wiki Maintenance Lambda
- SQS FIFO queue consumption
- WikiLocks defense-in-depth
- Flat wiki layout writes
- Page merge/upsert logic
- Contradiction detection
- Index/graph/log updates
- Step Functions callback
- Superseded version handling

### Excluded

- File deletion handling (Feature 06)
- Read API (Feature 07)

---

## Preconditions

- Feature 04 deployed (Phase 2 produces successful pages)
- Wiki bucket exists
- Wiki maintenance FIFO queue exists
- WikiLocks table exists

---

## Postconditions

After Phase 3 completes successfully:

| Artifact                | State                                            |
| ----------------------- | ------------------------------------------------ |
| Source summary page     | Created at `sources/{fileId}/{fileVersionId}.md` |
| Synthesized pages       | Merged/upserted at `{category}/{slug}.md`        |
| Wiki index              | Rebuilt with new pages                           |
| Wiki graph              | Rebuilt with new links                           |
| Wiki log                | Entry appended (idempotent)                      |

| Step Functions callback | Success or superseded                            |
| Job currentPhase        | `complete`                                       |

---

## Interfaces Consumed

| Interface              | Location                                                       |
| ---------------------- | -------------------------------------------------------------- |
| FilteredPhase2Output   | [`../interfaces/events.md`](../interfaces/events.md)           |
| WikiMaintenanceRequest | [`../interfaces/events.md`](../interfaces/events.md)           |
| Wiki bucket layout     | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md)     |
| WikiLocks table        | [`../interfaces/data-models.md`](../interfaces/data-models.md) |

## Interfaces Produced

| Interface                   | Location                                                   |
| --------------------------- | ---------------------------------------------------------- |
| StepFunctionsCallbackOutput | [`../interfaces/events.md`](../interfaces/events.md)       |
| Source page frontmatter     | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |
| Wiki bucket layout          | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |

---

## Functional Requirements

### FIFO Serialization

1. Wiki maintenance FIFO queue with `MessageGroupId=folderId`.
2. One message processed at a time per folder.
3. `MessageDeduplicationId` includes `$$.State.EnteredTime` (allows retries).

### Lock Acquisition

1. Acquire WikiLock as defense-in-depth.
2. Lock key = folderId, owner = fileVersionId.
3. TTL = 10 minutes (auto-release on crash).
4. If lock held by another, message returns to queue (visibility timeout).

### Superseded Version Check

1. Before mutating wiki, validate this is still the current version.
2. Get currentVersionId from Zarya Files table.
3. If currentVersionId != fileVersionId:
   - Archive processing artifacts
   - Send success callback with status `superseded`
   - Return normally (not an error)

### Wiki Mutation

1. Read existing wiki pages from S3 (if any).
2. Create source summary page.
3. For each synthesized page:
   - If page exists, detect contradictions, merge
   - If page new, create
4. Rebuild index.
5. Rebuild graph.
6. Append log entry (idempotent by `ingest:{fileVersionId}`).
7. Write all pages to their final S3 paths.

### Write Strategy

Since we use flat layout (no manifest):

1. Write pages directly to `wikis/{folderId}/{path}`.
2. FIFO queue ensures only one writer per folder at a time.
3. WikiLocks provides defense-in-depth.
4. S3 versioning provides per-object history.
5. Failure leaves partial state but next successful ingest regenerates index/graph.

### Contradiction Detection

1. Compare incoming claims with existing page claims.
2. If same-fileId conflict: newer version wins (auto-resolve).
3. If cross-file conflict: preserve both, create contradiction entry.
4. Update contradiction log.

### Step Functions Callback

1. On success: `send_task_success` with pagesCreated, pagesUpdated.
2. On superseded: `send_task_success` with status `superseded`.
3. On retryable error: `send_task_failure` with `RetryableError`.
4. On terminal error: `send_task_failure` with specific error code.
5. After callback, return normally (Lambda deletes message).

### Callback Error Handling

1. `InvalidToken`: Prior callback succeeded (expected on replay), log and return.
2. `TaskTimedOut`: Wiki committed but SF timed out, publish reconciliation alert.
3. Other errors: Log and raise (will retry).

---

## Failure Handling

| Failure             | Handling                                   |
| ------------------- | ------------------------------------------ |
| Lock contention     | Retryable: `send_task_failure`, SF retries |
| S3 transient error  | Retryable: `send_task_failure`, SF retries |
| Invalid claims      | Terminal: `send_task_failure`, job fails   |
| Source deleted      | Terminal: `send_task_failure`, job fails   |
| Callback timeout    | Alert for reconciliation                   |
| Pre-commit failure  | `send_task_failure`, job fails             |
| Post-commit failure | Log warning, wiki is committed             |

---

## Security Requirements

1. Lambda must have read/write access to Wiki bucket.
2. Lambda must have read/write access to WikiLocks table.
3. Lambda must have read access to Zarya Files table.
4. Lambda must have `states:SendTaskSuccess/Failure` permission.
5. No source content logged (only metadata).

---

## Observability Requirements

| Metric                    | Dimension           | Value                       |
| ------------------------- | ------------------- | --------------------------- |
| `phase3.duration_ms`      | environment         | Total execution time        |
| `phase3.pages_created`    | environment         | New pages created           |
| `phase3.pages_updated`    | environment         | Existing pages updated      |
| `phase3.contradictions`   | environment         | Contradictions detected     |
| `phase3.lock_wait_ms`     | environment         | Time waiting for lock       |
| `phase3.status`           | environment, status | succeeded/superseded/failed |
| `phase3.callback_timeout` | environment         | Callback timeout count      |

| Log Field        | Required | Example     |
| ---------------- | -------- | ----------- |
| `fileVersionId`  | Yes      | `"v-123"`   |
| `folderId`       | Yes      | `"f-456"`   |
| `phase`          | Yes      | `"phase3"`  |
| `pagesCreated`   | Yes      | `3`         |
| `pagesUpdated`   | Yes      | `2`         |
| `contradictions` | Yes      | `0`         |
| `callbackStatus` | Yes      | `"success"` |

---

## Verification Plan

1. **Single file ingest**
   - Process one file into empty folder
   - Verify source page created
   - Verify synthesized pages created
   - Verify index/graph/log updated

2. **Second file same folder**
   - Process second file into folder with existing wiki
   - Verify pages merged correctly
   - Verify FIFO serialization (second waits for first)
   - Verify log has both entries

3. **Superseded version**
   - Start processing version A
   - Complete version B while A in queue
   - Verify A becomes superseded
   - Verify callback has status `superseded`

4. **Lock contention**
   - Simulate lock held by another worker
   - Verify message returns to queue
   - Verify retry succeeds after lock released

5. **Partial failure recovery**
   - Simulate failure after some pages written
   - Verify next successful ingest regenerates index/graph
   - Verify no orphan pages accumulate

6. **Callback replay**
   - Replay Phase 3 message after success
   - Verify no duplicate wiki entries
   - Verify `InvalidToken` handled gracefully

---

## Rollout / Rollback

### Rollout

1. Deploy Phase 3 Lambda.
2. Ensure FIFO queue exists from Feature 00.
3. Update state machine with QueueWikiMaintenance task.
4. Test with single file before multi-file batch.

### Rollback

1. Disable state machine Wiki Maintenance step.
2. In-flight workflows will wait for timeout, then fail.
3. Wiki state is preserved (flat layout, FIFO serialization).
4. Rollback Lambda if needed.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
