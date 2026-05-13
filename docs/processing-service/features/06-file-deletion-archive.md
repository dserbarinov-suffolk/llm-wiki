# Feature 06: File Deletion and Archive

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - `folderId` is the wiki aggregate key and FIFO MessageGroupId.
> - All wiki writes for a folder must serialize through `wiki-maintenance.fifo`.
> - Folder deletion is out of scope in V1.

---

## Goal

Consume Zarya Files stream deletion events, send ARCHIVE_SOURCE requests, and archive/rebuild affected wiki pages.

---

## Scope

### Included

- Files stream event source mapping
- Stream Dispatcher handling for file deletion
- ARCHIVE_SOURCE event type
- Archive logic in Phase 3 Lambda
- Sole-source page archiving
- Multi-source page rebuild

### Excluded

- Folder deletion cascade (V1 non-goal)
- Physical deletion of wiki content (audit trail preserved)

---

## Preconditions

- Feature 05 deployed (wiki maintenance works)
- Zarya Files stream enabled with NEW_AND_OLD_IMAGES
- Wiki pages have structured `sources` frontmatter

---

## Postconditions

After file deletion processing:

| Condition         | Result                       |
| ----------------- | ---------------------------- |
| Sole-source page  | Page status = `archived`     |
| Multi-source page | Source removed, page rebuilt |
| Wiki index        | Excludes archived pages      |
| Wiki graph        | Excludes archived pages      |
| Wiki log          | Archive entry added          |

---

## Interfaces Consumed

| Interface          | Location                                                   |
| ------------------ | ---------------------------------------------------------- |
| Zarya Files stream | Zarya infrastructure                                       |
| Wiki bucket layout | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |

## Interfaces Produced

| Interface            | Location                                             |
| -------------------- | ---------------------------------------------------- |
| ArchiveSourceRequest | [`../interfaces/events.md`](../interfaces/events.md) |

---

## Functional Requirements

### Stream Detection

1. Event source mapping on Zarya Files table stream.
2. Filter: `eventName = MODIFY` AND `NewImage.isDeleted = true` AND `OldImage.isDeleted = false`.
3. Requires `NEW_AND_OLD_IMAGES` stream view type.

### Stream Dispatcher

1. Detect stream source from event source ARN (parse table name).
2. For Files table deletion events:
   - Extract fileId, folderId from record
   - Send ARCHIVE_SOURCE to wiki maintenance FIFO queue
   - Use `MessageGroupId = folderId` (serializes with Phase 3)

### ARCHIVE_SOURCE Processing

1. Phase 3 Lambda dispatches by `eventType`.
2. For ARCHIVE_SOURCE:
   - No Step Functions task token
   - Retries via SQS visibility timeout / maxReceiveCount

### Archive Logic

1. Load current wiki.
2. Find all pages citing the deleted fileId in `sources` frontmatter.
3. For each affected page:
   - If sole source: mark page `status: archived`
   - If multi-source: remove source from `sources`, rebuild page
4. Rebuild index (excludes archived pages).
5. Rebuild graph (excludes archived pages).
6. Append archive entry to log.
7. Write updated wiki pages to S3.

### Idempotency

1. Archive operation is idempotent by fileId.
2. Re-running archive for same fileId produces same result.
3. Log entry deduped by `archive:{fileId}:{timestamp}`.

---

## Failure Handling

| Failure                | Handling                          |
| ---------------------- | --------------------------------- |
| Wiki lock contention   | Retry via SQS visibility timeout  |
| S3 transient error     | Retry via SQS visibility timeout  |
| Max retries exceeded   | Move to DLQ, alert                |
| File not found in wiki | Log warning, succeed (idempotent) |

---

## Security Requirements

1. Lambda must have read access to Zarya Files stream.
2. Lambda must have read/write access to Wiki bucket.
3. Lambda must have read/write access to WikiLocks table.
4. No user content logged.

---

## Observability Requirements

| Metric                   | Dimension   | Value                      |
| ------------------------ | ----------- | -------------------------- |
| `archive.triggered`      | environment | Archive events received    |
| `archive.pages_archived` | environment | Sole-source pages archived |
| `archive.pages_rebuilt`  | environment | Multi-source pages rebuilt |
| `archive.duration_ms`    | environment | Total execution time       |

| Log Field       | Required | Example            |
| --------------- | -------- | ------------------ |
| `fileId`        | Yes      | `"f-123"`          |
| `folderId`      | Yes      | `"folder-456"`     |
| `action`        | Yes      | `"archive_source"` |
| `pagesArchived` | Yes      | `2`                |
| `pagesRebuilt`  | Yes      | `1`                |

---

## Verification Plan

1. **Sole-source deletion**
   - Process file A into wiki (creates pages X, Y)
   - Delete file A in Zarya
   - Verify pages X, Y have status `archived`
   - Verify index excludes pages X, Y
   - Verify log has archive entry

2. **Multi-source deletion**
   - Process file A and file B into wiki
   - Page Z cites both A and B
   - Delete file A
   - Verify page Z rebuilt with only B as source
   - Verify page Z still in index

3. **Idempotency**
   - Replay same deletion event
   - Verify no duplicate archive entries
   - Verify wiki unchanged

4. **FIFO serialization**
   - Delete file while ingest in progress
   - Verify operations serialize correctly
   - Verify no race condition

5. **DLQ behavior**
   - Force archive to fail 3 times
   - Verify message moves to DLQ
   - Verify alert triggered

---

## Rollout / Rollback

### Rollout

1. Deploy updated Stream Dispatcher with Files stream handling.
2. Create event source mapping for Files stream.
3. Test with synthetic deletion events.
4. Enable production stream consumption.

### Rollback

1. Disable Files stream event source mapping.
2. Archive events stop processing.
3. Wiki content remains unchanged (no deletions).
4. Re-enable later to process queued deletions.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
