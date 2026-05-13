# S3 Layout

> Centralized S3 object key schemas and wiki structure.
> All components that read or write S3 must conform to these layouts.

---

## Buckets Overview

| Bucket                       | Purpose                           | Retention  | Versioning |
| ---------------------------- | --------------------------------- | ---------- | ---------- |
| `processing-{env}-{account}` | Intermediate processing artifacts | 30 days    | Enabled    |
| `wikis-{env}-{account}`      | Persistent wiki content           | Indefinite | Enabled    |

---

## Processing Bucket Layout

```
processing-{env}-{account}/
└── processing/
    └── {fileVersionId}/
        ├── phase-0/
        │   ├── source.md              # Normalized markdown
        │   └── metadata.json          # Page count, warnings, extraction info
        ├── phase-1/
        │   ├── chunks.json            # Chunk boundaries
        │   ├── chunks/
        │   │   ├── 0000.json          # Per-chunk extraction result
        │   │   ├── 0001.json
        │   │   └── ...
        │   ├── claims-raw.jsonl       # Append-only raw claims (all chunks)
        │   ├── claims-normalized.json # Deduped claims with stable IDs
        │   └── candidates.json        # Proposed pages for synthesis
        └── phase-2/
            ├── {slug}.md              # Synthesized page content
            └── {slug}.judge.json      # Judge verdict for the page
```

### Key Format Rules

- `processing/` is the top-level prefix for all artifacts
- `fileVersionId` is the partition within processing/ (enables lifecycle expiration)
- Phase directories are numbered: `phase-0`, `phase-1`, `phase-2`
- Chunk files use zero-padded 4-digit indices: `0000.json`, `0001.json`
- Slugs are kebab-case: `topic-name.md`

### Lifecycle Configuration

```yaml
LifecycleConfiguration:
  Rules:
    - Id: ExpireProcessingArtifacts
      Status: Enabled
      ExpirationInDays: 30
      # No Prefix: all objects are processing artifacts
```

---

## Wiki Bucket Layout

> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md) specifies
> the flat layout below. This is the authoritative wiki structure.

```
wikis-{env}-{account}/
└── {folderId}/
    ├── schema.md              # Wiki maintenance instructions
    ├── index.md               # Content catalog
    ├── log.md                 # Append-only history
    ├── _graph.json            # Machine-readable link graph
    ├── sources/
    │   └── {fileId}/
    │       └── {fileVersionId}.md
    ├── concepts/
    │   └── {slug}.md
    ├── entities/
    │   └── {slug}.md
    ├── procedures/
    │   └── {slug}.md
    ├── references/
    │   └── {slug}.md
    ├── analyses/
    │   └── {slug}.md
    └── contradictions/
        └── {slug}.md
```

### Key Format Rules

- `folderId` is the top-level partition
- Nested directory for sources: `sources/{fileId}/{fileVersionId}.md`
- Slugs are kebab-case: `topic-name.md`
- Wiki files are written directly (no manifest/versioning layer)

### Atomic Write Strategy

Since this is a flat layout without versioning:

- Phase 3 writes pages individually to their final paths
- `log.md` uses conditional write (append-only semantics)
- `index.md` and `_graph.json` are regenerated after each successful ingest
- WikiLocks table + FIFO queue ensure serialized writes per folder

---

## Source Page Frontmatter

Every source summary page in `sources/{fileId}/` must include:

```yaml
---
title: <Human-readable title from source>
type: source
source_id: <fileVersionId>
file_id: <fileId>
file_version_id: <fileVersionId>
source_type: pdf | markdown
status: draft | reviewed | stable
created_at: <ISO timestamp>
updated_at: <ISO timestamp>
tags: []
---
```

### Source Page Body Structure

```markdown
# {Title}

## Summary

Brief summary of the source content.

## Key Claims

| Claim | Evidence      | Locator        |
| ----- | ------------- | -------------- |
| ...   | "exact quote" | normalized:L12 |

## Topics

- [Topic A](../concepts/topic-a.md)
- [Topic B](../references/topic-b.md)

## Metadata

- **File**: {original filename}
- **Pages**: {page count}
- **Processed**: {timestamp}
```

---

## Wiki Page Frontmatter

Every synthesized page (concepts, entities, procedures, references) must include:

```yaml
---
title: <Page title>
type: concept | entity | procedure | reference
status: draft | reviewed | stable
created_at: <ISO timestamp>
updated_at: <ISO timestamp>
sources:
  - file_id: <fileId>
    file_version_id: <fileVersionId>
    source_path: sources/{fileId}-{fileVersionId}.md
tags: []
---
```

### Multi-Source Pages

When a page is derived from multiple sources:

```yaml
sources:
  - file_id: f-123
    file_version_id: v-456
    source_path: sources/f-123/v-456.md
  - file_id: f-789
    file_version_id: v-012
    source_path: sources/f-789/v-012.md
```

---

## Contradiction Page Frontmatter

```yaml
---
title: Contradiction - <topic>
type: contradiction
status: unresolved | resolved
created_at: <ISO timestamp>
updated_at: <ISO timestamp>
claims:
  - source_file_id: <fileId>
    claim: "<claim text>"
    evidence: "<evidence quote>"
  - source_file_id: <fileId>
    claim: "<conflicting claim text>"
    evidence: "<evidence quote>"
resolution:
  status: unresolved | auto_resolved | user_resolved
  winner: <fileId> | null
  reason: <resolution reason>
---
```

---

## Log Idempotency

Wiki log entries (`log.md`) use idempotency keys to prevent duplicates:

```markdown
## 2026-05-12T10:30:00Z | ingest | v-123

- **Source**: document.pdf
- **Pages created**: 3
- **Pages updated**: 2

<!-- idempotency_key: ingest:v-123 -->
```

**Append protocol**:

1. Check if `idempotency_key` exists in current log
2. If exists, skip (idempotent)
3. If not exists, append entry with key

---

## Object Metadata

All S3 objects should include metadata headers for tracing:

| Header                       | Value                  |
| ---------------------------- | ---------------------- |
| `x-amz-meta-file-version-id` | `{fileVersionId}`      |
| `x-amz-meta-folder-id`       | `{folderId}`           |
| `x-amz-meta-phase`           | `phase0`/`phase1`/etc. |

---

## Changelog

| Date       | Change                                |
| ---------- | ------------------------------------- |
| 2026-05-13 | Initial layout extraction from design |
