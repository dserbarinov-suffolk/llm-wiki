# Data Models

> Centralized DynamoDB table schemas and status lifecycles.
> This file is the source of truth for table structure; CloudFormation implements these definitions.

---

## Tables Overview

| Table           | Purpose                                | Partition Key   | Sort Key |
| --------------- | -------------------------------------- | --------------- | -------- |
| ProcessingJobs  | Workflow state per FileVersion         | `fileVersionId` | -        |
| ProcessingTasks | Per-phase task tracking                | `taskId`        | -        |
| WikiLocks       | Defense-in-depth lock for Phase 3      | `folderId`      | -        |
| RateLimits      | Token bucket for Bedrock rate limiting | `modelId`       | -        |

---

## ProcessingJobs

> **Source of Truth**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)

Primary workflow tracking. One item per FileVersion.

### Schema

```
Table: processing-jobs-{env}
Partition Key: fileVersionId (String)

Attributes:
  fileVersionId         String    # PK, workflow idempotency key
  fileId                String    # Zarya file identity
  folderId              String    # Wiki target, Phase 3 serialization key
  sourceObjectKey       String    # Zarya committed object key: objects/{fileId}/{fileVersionId}
  sourceEtagOrChecksum  String    # S3 ETag or checksum for integrity validation
  sourceSizeBytes       Number    # For size-based routing
  sourceContentType     String    # application/pdf or text/markdown
  status                String    # See status model below
  currentPhase          String    # phase0 | phase1 | phase2 | phase3 | complete
  stepFunctionExecutionArn String
  attempts              Number
  createdAt             String    # ISO timestamp
  updatedAt             String    # ISO timestamp
  completedAt           String    # ISO timestamp (when terminal)
  errorCode             String    # Sanitized error code
  errorMessage          String    # Sanitized error message
  artifactPaths         Map       # Pointers to S3 artifacts
  synthesizedPageCount  Number    # Count for quick display
  claimCount            Number    # Total claims extracted
```

### artifactPaths Map

```
{
  normalizedMarkdownKey:  "processing/{fileVersionId}/phase-0/source.md"
  claimsRawKey:           "processing/{fileVersionId}/phase-1/claims-raw.jsonl"
  claimsNormalizedKey:    "processing/{fileVersionId}/phase-1/claims-normalized.json"
  candidatesKey:          "processing/{fileVersionId}/phase-1/candidates.json"
}
```

### GSI: byFolder

```
Partition Key: folderId
Sort Key: createdAt
Projection: ALL
```

Use case: List all jobs for a folder, ordered by creation time.

### Status Values

```
queued | running | blocked | partial | succeeded | failed | superseded
```

**Terminal states**: `partial`, `succeeded`, `failed`, `superseded`

**Status transition rules**:

- Only `MarkJobStatus` Lambda (invoked by Step Functions terminal states) may write terminal status.
- Phase workers may write `running` or `blocked`, never terminal states.

---

## ProcessingTasks

Per-phase task tracking for debugging and resume.

### Schema

```
Table: processing-tasks-{env}
Partition Key: taskId (String)  # Format: {fileVersionId}#{phase}#{slug}

Attributes:
  taskId            String    # PK
  fileVersionId     String    # Parent workflow
  phase             String    # normalizeDocument | extractClaims | synthesize | maintainWiki
  slug              String    # For phase1/phase2, null otherwise
  chunkIndex        Number    # For phase1 chunks
  status            String    # queued | running | succeeded | failed
  attempts          Number
  inputObjectKey    String
  outputObjectKey   String
  modelId           String    # Bedrock model used
  inputTokens       Number
  outputTokens      Number
  durationMs        Number
  createdAt         String
  updatedAt         String
  errorCode         String
  errorMessage      String
```

### GSI: byFileVersion

```
Partition Key: fileVersionId
Sort Key: taskId
Projection: ALL
```

Use case: List all tasks for a job, for debugging.

### taskId Format

| Phase   | Format                                  | Example                    |
| ------- | --------------------------------------- | -------------------------- |
| Phase 0 | `{fileVersionId}#normalize#-`           | `v-123#normalize#-`        |
| Phase 1 | `{fileVersionId}#extractClaims#{i:04d}` | `v-123#extractClaims#0000` |
| Phase 2 | `{fileVersionId}#synthesize#{slug}`     | `v-123#synthesize#topic-a` |
| Phase 3 | `{fileVersionId}#maintainWiki#-`        | `v-123#maintainWiki#-`     |

> Note: Phase 1 chunk indices use zero-padded 4-digit format (`0000`, `0001`, etc.)
> to match S3 chunk file naming and enable lexicographic sorting.

---

## WikiLocks

Defense-in-depth conditional lock for Phase 3 serialization.

### Schema

```
Table: wiki-locks-{env}
Partition Key: folderId (String)

Attributes:
  folderId    String    # PK
  lockedBy    String    # fileVersionId holding the lock
  lockedAt    String    # ISO timestamp
  ttl         Number    # DynamoDB TTL epoch seconds (auto-release after 10 min)
```

### Lock Semantics

- **Primary serialization**: SQS FIFO with `MessageGroupId=folderId`
- **Defense-in-depth**: WikiLocks handles edge cases where SQS visibility timeout expires before Lambda completes

### Acquire Lock

```python
dynamodb.put_item(
    TableName=WIKI_LOCKS_TABLE,
    Item={
        "folderId": folder_id,
        "lockedBy": file_version_id,
        "lockedAt": datetime.utcnow().isoformat(),
        "ttl": int(time.time()) + 600,  # 10 min TTL
    },
    ConditionExpression="attribute_not_exists(folderId) OR #ttl < :now",
    ExpressionAttributeNames={"#ttl": "ttl"},
    ExpressionAttributeValues={":now": int(time.time())},
)
```

### Release Lock

```python
dynamodb.delete_item(
    TableName=WIKI_LOCKS_TABLE,
    Key={"folderId": folder_id},
    ConditionExpression="lockedBy = :fvid",
    ExpressionAttributeValues={":fvid": file_version_id},
)
```

---

## RateLimits

Token bucket for Bedrock rate limiting.

### Schema

```
Table: rate-limits-{env}
Partition Key: modelId (String)

Attributes:
  modelId           String    # PK (e.g., "qwen.qwen3-coder-30b-a3b-v1:0")
  tokensAvailable   Number    # Current token bucket level
  requestsAvailable Number    # Current request bucket level
  lastRefillAt      String    # ISO timestamp of last refill
  maxTokens         Number    # Token bucket capacity
  maxRequests       Number    # Request bucket capacity
  tokensPerMinute   Number    # Refill rate
  requestsPerMinute Number    # Refill rate
```

### Refill Logic

Tokens and requests refill based on elapsed time since `lastRefillAt`:

```python
elapsed_minutes = (now - last_refill_at).total_seconds() / 60
tokens_to_add = elapsed_minutes * tokens_per_minute
requests_to_add = elapsed_minutes * requests_per_minute

new_tokens = min(max_tokens, tokens_available + tokens_to_add)
new_requests = min(max_requests, requests_available + requests_to_add)
```

---

## Required Capacity

| Table           | RCU (on-demand) | WCU (on-demand) | Notes                   |
| --------------- | --------------- | --------------- | ----------------------- |
| ProcessingJobs  | Auto            | Auto            | ~50 writes per PDF      |
| ProcessingTasks | Auto            | Auto            | ~20 writes per PDF      |
| WikiLocks       | Auto            | Auto            | 1-2 writes per Phase 3  |
| RateLimits      | Auto            | Auto            | ~10 writes per LLM call |

All tables use on-demand capacity mode. Reserved capacity can be configured for predictable workloads.

---

## Retention

| Table           | Retention     | Mechanism                  |
| --------------- | ------------- | -------------------------- |
| ProcessingJobs  | Indefinite    | No TTL (audit trail)       |
| ProcessingTasks | 30 days       | DynamoDB TTL on `ttl` attr |
| WikiLocks       | Auto-expiring | DynamoDB TTL (10 min)      |
| RateLimits      | Indefinite    | Persistent config          |

---

## Changelog

| Date       | Change                                |
| ---------- | ------------------------------------- |
| 2026-05-13 | Initial schema extraction from design |
