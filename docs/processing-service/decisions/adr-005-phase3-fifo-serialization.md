# ADR-005: FIFO Queue for Phase 3 Serialization

## Status

Accepted

## Context

Phase 3 (wiki maintenance) must serialize writes per folder to prevent:

- Concurrent index/graph rebuilds
- Lost updates from race conditions
- Inconsistent wiki state

Options considered:

1. **SQS FIFO with MessageGroupId** - Queue-based serialization
2. **DynamoDB lock only** - Optimistic locking
3. **Step Functions Choice state** - Workflow-level coordination
4. **Redis/ElastiCache lock** - External lock service

## Decision

Use **SQS FIFO queue** with `MessageGroupId=folderId` as the primary serialization mechanism, with **DynamoDB WikiLocks** as defense-in-depth.

## Rationale

### Why SQS FIFO

- **Built-in ordering**: FIFO guarantees one message per group processes at a time
- **No lock management**: Queue handles serialization automatically
- **waitForTaskToken**: Integrates with Step Functions callback pattern
- **Serverless**: No infrastructure to manage
- **Retry semantics**: Visibility timeout + DLQ for failures

### Why MessageGroupId = folderId

- Each folder has one wiki
- Parallel processing across folders is safe
- Serialization only needed within a folder

### Why Defense-in-Depth with DynamoDB Lock

FIFO provides primary serialization, but edge cases exist:

- Lambda timeout before processing completes
- Visibility timeout expires during long operation
- SQS message becomes visible again while first processor still running

WikiLocks table handles these cases:

- TTL-based lock with conditional writes
- Lock holder identified by fileVersionId
- Auto-release after 10 minutes

### Trade-offs Accepted

- **Latency**: FIFO has slightly higher latency than standard queues
- **Throughput**: 300 msg/s per MessageGroupId (sufficient for our scale)
- **Complexity**: Two serialization mechanisms to maintain

## Alternatives Rejected

### DynamoDB Lock Only

- No queue semantics (must poll or use streams)
- Complex retry logic
- No natural ordering

### Step Functions Coordination

- Would require global state across executions
- Complex to implement workflow-to-workflow coordination
- Doesn't scale across concurrent workflows

### Redis Lock

- Additional infrastructure to operate
- Single point of failure
- Overkill for our throughput

## Consequences

- Wiki maintenance queue must be FIFO
- MessageGroupId must be folderId (not fileVersionId)
- MessageDeduplicationId must include timestamp to allow retries
- WikiLocks table provides secondary protection
- Phase 3 Lambda must handle lock acquisition and release

## Related

- [System Contract](../system-contract.md) - FIFO serialization invariant
- [Feature 05](../features/05-phase3-wiki-maintenance.md) - Lock implementation
- [Data Models](../interfaces/data-models.md) - WikiLocks schema

## Date

2026-05-13
