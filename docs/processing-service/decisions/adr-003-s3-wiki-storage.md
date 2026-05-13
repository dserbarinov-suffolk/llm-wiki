# ADR-003: S3 for Wiki Storage

## Status

Accepted (Updated 2026-05-13: Flat layout adopted from handoff contract)

## Context

The processing service needs persistent storage for wiki content with:

- Markdown files (pages, index, log)
- JSON files (graph)
- Serialized writes (via FIFO + WikiLocks)
- Version history
- Cross-region durability

Options considered:

1. **S3** - Object storage
2. **DynamoDB** - Document database
3. **EFS** - File system
4. **Git repository** - Version control

## Decision

Use **S3** for wiki storage with a **flat layout** as specified in the
[SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md).

**Update (2026-05-13)**: The original decision included a manifest-based atomic commit
protocol. After reviewing the integration handoff, we adopt the simpler flat layout
where wiki files are written directly to their final paths. Serialization is achieved
through SQS FIFO queues (`MessageGroupId=folderId`) and the WikiLocks DynamoDB table,
not through a manifest protocol.

## Rationale

### Why S3

- **Cost**: $0.023/GB/month (cheapest durable storage)
- **Durability**: 99.999999999% (11 nines)
- **Versioning**: Built-in object versioning for history
- **Simplicity**: Files are just objects
- **Scale**: No capacity planning
- **Access patterns**: Read-heavy with occasional writes (matches S3)

### Flat Layout with Serialized Writes

Since S3 doesn't have native transactions and the handoff contract specifies a flat layout,
we use serialization instead of staging/manifest:

1. SQS FIFO queue with `MessageGroupId=folderId` ensures one wiki write at a time per folder
2. WikiLocks DynamoDB table provides defense-in-depth locking
3. Files are written directly to their final paths
4. S3 versioning provides history for individual objects

This is simpler than a manifest approach and matches the handoff contract.

### Trade-offs Accepted

- **Consistency**: Eventually consistent (acceptable for wiki reads)
- **No queries**: Can't query file contents (use DynamoDB for metadata)
- **Latency**: Higher than in-memory (acceptable for wiki access patterns)
- **No atomic multi-file writes**: Serialization ensures only one writer at a time

## Alternatives Rejected

### DynamoDB

- 400KB item limit too small for wiki pages
- Complex pagination for large documents
- Higher cost for our data volume

### EFS

- Requires VPC and mount targets
- Higher cost than S3
- Unnecessary for our access patterns

### Git repository

- Complex automation for serverless writes
- Merge conflicts with concurrent updates
- Overkill for folder-isolated wikis

## Consequences

- Wiki bucket must have versioning enabled (for object-level history)
- Lifecycle rules manage old versions
- Wiki writes must go through FIFO queue for serialization
- Writers must acquire WikiLock before mutating wiki
- Server-side encryption required (SSE-S3 minimum)

## Related

- [S3 Layout](../interfaces/s3-layout.md) - Object key schema
- [Feature 05](../features/05-phase3-wiki-maintenance.md) - Wiki maintenance implementation

## Date

2026-05-13
