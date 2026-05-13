# Event Schemas

> Centralized event contracts for the processing service.
> All event producers and consumers must conform to these schemas.
> Feature specs reference this file; they do not duplicate schemas.

---

## Event Flow Overview

```
Zarya FileVersions Stream
    │
    ▼ FileVersionCommittedEvent
Intake Queue
    │
    ▼ StepFunctionsInput
Step Functions
    │
    ├── Phase0Output
    │
    ├── Phase1ChunkResult[]
    │
    ├── Phase1AggregationOutput
    │
    ├── Phase2PageResult[]
    │
    ├── FilteredPhase2Output
    │
    └── WikiMaintenanceRequest ──▶ FIFO Queue
                                        │
                                        ▼ StepFunctionsCallbackOutput
                                   Step Functions
```

---

## 1. FileVersionCommittedEvent

> **Source of Truth**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)

**Producer**: Stream Dispatcher  
**Consumer**: Intake Queue → Intake Consumer

```typescript
interface FileVersionCommittedEvent {
  eventType: "FileVersionCommitted";
  schemaVersion: 1;
  environment: string;
  fileVersionId: string; // Trace ID, workflow idempotency key
  fileId: string; // Stable file identity
  folderId: string; // Wiki aggregate key (resolved by dispatcher from Files table)
  versionNumber: number;
  objectKey: string; // Zarya committed S3 key: objects/{fileId}/{fileVersionId}
  etagOrChecksum: string; // S3 ETag or checksum from Zarya FileVersion
  sizeBytes: number;
  contentType: string; // "application/pdf" | "text/markdown" | "text/x-markdown"
  createdByUserId: string;
  createdAt: string; // ISO timestamp
}
```

> **Note**: `folderId` is resolved by the dispatcher via `GetItem` on Zarya's Files table.
> The raw DynamoDB stream event from FileVersions does not include `folderId`.

**Filtering rules** (applied by Stream Dispatcher):

- Content type must match `^(application/pdf|text/markdown|text/x-markdown)$`
- Size must be ≤ 250MB
- Must be current version (`fileRecord.currentVersionId === fileVersionId`)
- Must have a folder (`parentFolderId !== null`)

---

## 2. StepFunctionsInput

**Producer**: Intake Consumer  
**Consumer**: Step Functions state machine

```typescript
interface StepFunctionsInput {
  fileVersionId: string;
  fileId: string;
  folderId: string;
  sourceObjectKey: string; // objects/{fileId}/{fileVersionId}
  sourceEtagOrChecksum: string; // For integrity validation
  sourceSizeBytes: number;
  sourceContentType: string;
}
```

**Idempotency**: Execution name = `fileVersionId`. Duplicate starts return `ExecutionAlreadyExists`.

**Integrity validation**: Phase 0 should verify the observed S3 ETag matches `sourceEtagOrChecksum`
before processing. See [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md).

---

## 3. Phase0Output

**Producer**: Phase 0 Normalize Lambda  
**Consumer**: Step Functions (stored at `$.phase0`)

```typescript
interface Phase0Output {
  normalizedMarkdownKey: string; // S3 key: {fileVersionId}/phase-0/source.md
  metadataKey: string; // S3 key: {fileVersionId}/phase-0/metadata.json
  chunks: ChunkDefinition[];
}

interface ChunkDefinition {
  index: number;
  startLine: number;
  endLine: number;
  tokenEstimate: number;
}
```

---

## 4. Phase1ChunkResult

**Producer**: Phase 1 Extract Chunk Lambda (per chunk)  
**Consumer**: Step Functions Map state (collected at `$.phase1.chunkResults`)

```typescript
interface Phase1ChunkResult {
  chunkIndex: number;
  status: "succeeded" | "failed";
  claimCount?: number; // Present if succeeded
  error?: string; // Present if failed
  errorCode?: string; // Present if failed
}
```

**Failure handling**: Failed chunks are collected but do not fail the Map state.
The aggregator enforces the failure threshold.

---

## 5. Phase1AggregationOutput

**Producer**: Phase 1 Aggregate Claims Lambda  
**Consumer**: Step Functions (stored at `$.phase1.aggregation`)

```typescript
interface Phase1AggregationOutput {
  candidates: CandidatePage[];
  claimsNormalizedKey: string; // S3 key: {fileVersionId}/phase-1/claims-normalized.json
  totalClaims: number;
  failedChunks: number[]; // Indices of failed chunks (for debugging)
}

interface CandidatePage {
  topic: string;
  slug: string;
  pagePath: string; // e.g., "concepts/foo.md"
  category:
    | "source"
    | "concept"
    | "entity"
    | "procedure"
    | "reference"
    | "analysis";
  claimIds: string[]; // References to claims in claims-normalized.json
}
```

**Failure threshold**: If >20% of chunks failed, the aggregator throws `TooManyChunkFailures`
and the workflow transitions to `MarkFailed`.

---

## 6. Phase2PageResult

**Producer**: Phase 2 Synthesize Page Lambda (per candidate)  
**Consumer**: Step Functions Map state (collected at `$.phase2.synthesizedPages`)

```typescript
interface Phase2PageResult {
  status: "succeeded" | "failed";
  pagePath: string; // e.g., "concepts/foo.md"
  slug: string;
  category:
    | "source"
    | "concept"
    | "entity"
    | "procedure"
    | "reference"
    | "analysis";
  objectKey?: string; // S3 key if succeeded: processing/{fileVersionId}/phase-2/{slug}.md
  claimCount?: number; // Present if succeeded
  judgeVerdict?: string; // Present if succeeded
  error?: string; // Present if failed
  errorCode?: string; // Present if failed
}
```

---

## 7. FilteredPhase2Output

**Producer**: Filter Phase 2 Results Lambda  
**Consumer**: Step Functions (stored at `$.phase2Filtered`)

```typescript
interface FilteredPhase2Output {
  status: "succeeded" | "partial" | "failed";
  successfulPages: Phase2PageResult[]; // Only pages with status: "succeeded"
  failedPages: FailedPageSummary[];
}

interface FailedPageSummary {
  pagePath: string;
  slug: string;
  category: string;
  error: string;
}
```

**Status logic**:

- `succeeded`: All pages succeeded
- `partial`: Some pages succeeded, some failed
- `failed`: Zero pages succeeded

---

## 8. WikiMaintenanceRequest

**Producer**: Step Functions `QueueWikiMaintenance` task  
**Consumer**: Phase 3 Wiki Maintenance Lambda

```typescript
interface WikiMaintenanceRequest {
  eventType: "WIKI_MAINTENANCE";
  schemaVersion: 1;
  fileVersionId: string;
  fileId: string;
  folderId: string;
  normalizedMarkdownKey: string;
  claimsNormalizedKey: string;
  synthesizedPages: Phase2PageResult[]; // Only successful pages
  taskToken: string; // Step Functions callback token
  timestamp: string; // $$.State.EnteredTime
}
```

**Queue envelope**:

- Queue: `wiki-maintenance-{env}.fifo`
- `MessageGroupId`: `folderId` (serializes all wiki writes for a folder)
- `MessageDeduplicationId`: `{fileVersionId}:{timestamp}` (allows retries)

---

## 9. ArchiveSourceRequest

**Producer**: Stream Dispatcher (on file deletion)  
**Consumer**: Phase 3 Wiki Maintenance Lambda

```typescript
interface ArchiveSourceRequest {
  eventType: "ARCHIVE_SOURCE";
  schemaVersion: 1;
  fileId: string;
  folderId: string;
  deletedAt: string; // ISO timestamp
}
```

**Note**: `ARCHIVE_SOURCE` does not have a Step Functions task token.
Retries are handled by SQS visibility timeout / maxReceiveCount.

---

## 10. StepFunctionsCallbackOutput

**Producer**: Phase 3 Wiki Maintenance Lambda  
**Consumer**: Step Functions (via `send_task_success`)

```typescript
interface StepFunctionsCallbackOutput {
  status: "succeeded" | "superseded";
  fileVersionId: string;
  folderId: string;
  pagesCreated?: number; // Present if succeeded
  pagesUpdated?: number; // Present if succeeded
  currentVersionId?: string; // Present if superseded
}
```

**Callback contract**:

- Every handled path sends exactly one callback (success or failure).
- Handler returns normally after callback, allowing Lambda to delete the SQS message.
- `InvalidToken` exceptions indicate a prior successful callback (expected on replay).
- `TaskTimedOut` exceptions trigger reconciliation alerts.

---

## Schema Versioning

All events include `schemaVersion: 1`. When schemas change:

1. Increment `schemaVersion`.
2. Consumers must handle both old and new versions during migration.
3. Document changes in the changelog below.

### Changelog

| Date       | Version | Change                                |
| ---------- | ------- | ------------------------------------- |
| 2026-05-13 | 1       | Initial schema extraction from design |
