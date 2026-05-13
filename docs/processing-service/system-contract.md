# System Contract

> This file defines stable cross-cutting invariants for the processing service.
> Every feature specification must read and conform to this contract.
> Changes to this contract require explicit review and approval.
>
> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../SCALABLE-FILE-PROCESSING-HANDOFF.md)
> is the source of truth for Zarya integration points.

---

## Zarya Integration Rules

> These rules come from the Zarya integration handoff and are non-negotiable.

### Canonical Trigger

The processing service treats a committed `FileVersion` row as the business event.

- **Do not use S3 object-created events** as the primary trigger.
- Zarya uploads to `pending-uploads/`, copies to `objects/`, then commits metadata.
- A committed S3 object without a FileVersion row is an incomplete state.

Event source: DynamoDB stream on `zarya-{env}-file-versions` (INSERT events only).

### Zarya Table Access

| Table                       | Access                 | Purpose                                      |
| --------------------------- | ---------------------- | -------------------------------------------- |
| `zarya-{env}-file-versions` | Stream read, `GetItem` | Trigger, validation                          |
| `zarya-{env}-files`         | `GetItem` only         | Resolve `folderId`, check `currentVersionId` |

**Forbidden**: Writing to any Zarya table.

### Source Object Path

Committed objects follow: `objects/{fileId}/{fileVersionId}`

The dispatcher must validate `objectKey` starts with `objects/` before processing.

### Source Bucket Resolution

The Zarya source bucket name is a deployment parameter, not derived from DynamoDB.
The FileVersion record provides the `objectKey` (path within bucket), but the bucket
name itself must be configured at deployment time via the `ZaryaFilesBucket` parameter
in the foundation stack. This allows the same Lambda code to work across environments
without hardcoding bucket names.

### Current Version Rule

Process only the current user-visible version:

1. Load `File` record
2. Require `File.isDeleted === false`
3. Require `File.currentVersionId === FileVersion.fileVersionId`

If a newer version arrives during processing, downstream wiki maintenance lets the newer version win.

### Integrity Validation

Before Phase 0 conversion:

1. Read source with the committed `objectKey`
2. Compare observed S3 ETag with `etagOrChecksum` from FileVersion
3. If mismatch, fail with integrity error

---

## Non-Negotiable Invariants

### Identity Model

- **One wiki per Zarya folder.** `folderId` is the wiki aggregate key.
- **`fileVersionId`** is the ingest trace ID and workflow idempotency key.
- **`fileId`** is stable source identity across versions.
- **`folderId`** is the wiki aggregate key and FIFO MessageGroupId for serialization.
- **`taskId = {fileVersionId}#{phase}#{slug}`** identifies slug-level work units.

### Scope Boundaries (V1)

- Top-level files (`parentFolderId === null`) are skipped.
- Folder deletion is out of scope.
- User cancellation of jobs is not supported.
- User resolution of contradictions is not supported.

### Data Sensitivity

- Wiki content (normalized markdown, claims, wiki pages) is **user-content-derived**.
- Derived data has the **same sensitivity classification as source content**.
- If source files require SSE-KMS, derived data storage must match or exceed that level.

### Orchestration Authority

- **Step Functions owns terminal job status.** Only `MarkJobStatus` (invoked by terminal ASL states) may write terminal status values.
- **Phase 3 worker commits wiki state** but does not decide final job status—it reports success/failure via callback.
- **SQS FIFO with `MessageGroupId=folderId`** serializes all wiki writes for a folder.
- **WikiLocks DynamoDB table** provides defense-in-depth for edge cases.

### Idempotency

- All persisted artifacts must be idempotent by `fileVersionId` or deterministic event ID.
- Step Functions execution name = `fileVersionId` (prevents duplicate workflows).
- FIFO `MessageDeduplicationId` includes `$$.State.EnteredTime` (allows retries).
- Wiki log entries use `idempotency_key = ingest:{fileVersionId}`.

---

## Status Model

### Job Status Values

```
queued | running | blocked | partial | succeeded | failed | superseded
```

### Terminal States

```
partial | succeeded | failed | superseded
```

### Status Lifecycle

```
queued ──┬── running ──┬── succeeded
         │             ├── partial
         │             ├── failed
         │             └── superseded
         └── blocked ──┴── (retried → running)
```

### Who Writes Status

| Status       | Writer                     | Trigger                                       |
| ------------ | -------------------------- | --------------------------------------------- |
| `queued`     | Intake Consumer            | Job created                                   |
| `running`    | Step Functions task states | Workflow starts phase                         |
| `blocked`    | Phase 3 worker             | Retryable error (lock contention, etc.)       |
| `partial`    | MarkJobStatus (ASL)        | Phase 3 callback success, some pages failed   |
| `succeeded`  | MarkJobStatus (ASL)        | Phase 3 callback success, all pages succeeded |
| `failed`     | MarkJobStatus (ASL)        | Terminal error in any phase                   |
| `superseded` | MarkJobStatus (ASL)        | Newer version completed during processing     |

---

## Source of Truth Table

| Concept               | Source of Truth          | Notes                                                 |
| --------------------- | ------------------------ | ----------------------------------------------------- |
| Job exists            | ProcessingJobs table     | Created by Intake Consumer                            |
| Job terminal status   | Step Functions execution | MarkJobStatus writes to DynamoDB                      |
| Wiki content          | S3 `wikis/{folderId}/`   | Flat layout per handoff contract                      |
| Processing artifacts  | S3 `processing/{fvId}/`  | Retained 30 days                                      |
| Folder access rights  | Zarya auth API           | Processing service does not cache decisions long-term |
| Source file existence | Zarya Files table        | `isDeleted` = soft delete                             |

---

## Security Classification

| Data                | Classification                    | Storage       | Encryption |
| ------------------- | --------------------------------- | ------------- | ---------- |
| Source PDFs         | User content                      | Zarya S3      | SSE-S3     |
| Normalized Markdown | User-content-derived (same class) | Processing S3 | SSE-S3     |
| Claims/Evidence     | User-content-derived (same class) | Processing S3 | SSE-S3     |
| Wiki pages          | User-content-derived (same class) | Wiki S3       | SSE-S3     |
| Job metadata        | Operational                       | DynamoDB      | Encrypted  |

> **Note**: If source files require stronger encryption (e.g., SSE-KMS with CMK),
> derived data storage must match or exceed that level.

---

## Deployment Order

| Step | Stack/Change                | Depends On             |
| ---- | --------------------------- | ---------------------- |
| 1    | Zarya FileVersions stream   | Zarya foundation stack |
| 2    | Zarya Files stream          | Zarya foundation stack |
| 3    | Zarya stream ARN exports    | Steps 1-2              |
| 4    | Processing foundation stack | AWS account exists     |
| 5    | Processing app stack        | Steps 3-4              |

> **Note**: The foundation stack can be deployed before Zarya streams are enabled.
> The app stack requires the streams because it creates event source mappings.

---

## Explicit Non-Goals (V1)

- **Folder deletion cascade**: Only individual file deletion is supported.
- **Job cancellation**: Jobs cannot be cancelled by users.
- **Contradiction resolution UI**: Contradictions are read-only.
- **Personal wikis**: Top-level files without a folder are skipped.
- **Cross-folder wiki linking**: Each folder has an isolated wiki.
- **Partial-folder access**: `readWikiContent` requires access to all live sources.
- **Page-level source filtering**: All or nothing per folder.

---

## Contract Versioning

When this contract must change:

1. Propose the change in a PR description or design doc.
2. Identify all feature specs affected.
3. Update affected specs in the same PR.
4. Add a dated changelog entry below.

### Changelog

| Date       | Change                                 | Affected Specs |
| ---------- | -------------------------------------- | -------------- |
| 2026-05-13 | Initial contract extracted from design | All            |
