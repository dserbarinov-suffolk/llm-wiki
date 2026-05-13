# Feature 02: Phase 0 - Source Normalization

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)
> defines source object paths and integrity validation requirements.
>
> Relevant invariants:
>
> - `fileVersionId` is the trace ID and workflow idempotency key.
> - Derived data has the same sensitivity classification as source content.
> - All persisted artifacts must be idempotent by `fileVersionId`.

---

## Goal

Convert source PDF/Markdown to normalized Markdown, store artifacts, and produce chunk list for Phase 1.

---

## Scope

### Included

- Phase 0 Normalize Lambda
- PDF → Markdown conversion using pymupdf4llm
- Markdown passthrough (for .md sources)
- Source integrity validation with ETag
- Chunk boundary calculation
- Artifact storage in Processing bucket

### Excluded

- Large PDF handling via ECS (deferred to future enhancement)
- LLM calls (Feature 03)
- Wiki maintenance (Feature 05)

---

## Preconditions

- Feature 01 deployed (workflow can reach Phase 0)
- Source object exists at `objects/{fileId}/{fileVersionId}`
- Processing bucket exists
- pymupdf4llm layer available

---

## Postconditions

After Phase 0 completes successfully:

| Artifact                     | Location                                |
| ---------------------------- | --------------------------------------- |
| Normalized Markdown          | `{fileVersionId}/phase-0/source.md`     |
| Extraction metadata          | `{fileVersionId}/phase-0/metadata.json` |
| Chunk list in workflow state | `$.phase0.chunks`                       |
| Job currentPhase             | `phase1`                                |

---

## Interfaces Consumed

| Interface                 | Location                                             |
| ------------------------- | ---------------------------------------------------- |
| StepFunctionsInput        | [`../interfaces/events.md`](../interfaces/events.md) |
| Source object in Zarya S3 | Zarya infrastructure                                 |
| Processing bucket         | Foundation (Feature 00)                              |

## Interfaces Produced

| Interface                | Location                                                   |
| ------------------------ | ---------------------------------------------------------- |
| Phase0Output             | [`../interfaces/events.md`](../interfaces/events.md)       |
| Processing bucket layout | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |

---

## Functional Requirements

### Source Validation

> Per the integration contract, validate integrity before processing.

1. Call HeadObject on `sourceObjectKey` (e.g., `objects/{fileId}/{fileVersionId}`).
2. If HeadObject fails, fail with SOURCE_OBJECT_MISSING.
3. Compare observed S3 ETag with `sourceEtagOrChecksum` from workflow input.
4. If ETag mismatch, fail with SOURCE_INTEGRITY_ERROR.
5. Download source using GetObject.

### PDF Conversion

1. Use pymupdf4llm to convert PDF to Markdown.
2. Clean PDF artifacts (headers, footers, page numbers).
3. Preserve document structure (headings, lists, tables).
4. Extract images to `{fileVersionId}/phase-0/assets/` if configured.

### Markdown Passthrough

1. For text/markdown sources, copy content without conversion.
2. Normalize line endings (CRLF → LF).
3. Validate UTF-8 encoding.

### Chunking

1. Split normalized Markdown into chunks of ~6500 tokens each.
2. Respect heading boundaries when possible.
3. Respect code block and table boundaries.
4. Return chunk list with startLine, endLine, tokenEstimate.

### Artifact Storage

1. Store normalized Markdown at `{fileVersionId}/phase-0/source.md`.
2. Store metadata at `{fileVersionId}/phase-0/metadata.json`.
3. Include S3 metadata header `x-amz-meta-file-version-id`.

### Size Routing (Future)

For V1, all files go through Lambda. Future enhancement:

- Files > 100MB route to ECS Fargate
- ECS uses same logic, more memory/storage

---

## Failure Handling

| Failure                | Handling                                 |
| ---------------------- | ---------------------------------------- |
| Source object missing  | Terminal failure: SOURCE_OBJECT_MISSING  |
| ETag mismatch          | Terminal failure: SOURCE_INTEGRITY_ERROR |
| PDF conversion fails   | Terminal failure: PDF_CONVERSION_ERROR   |
| Invalid encoding       | Terminal failure: ENCODING_ERROR         |
| Lambda timeout (5 min) | Workflow catch → MarkFailed              |
| S3 transient error     | Retry with backoff                       |

---

## Security Requirements

1. Lambda must have read access to Zarya source bucket.
2. Lambda must have write access to Processing bucket.
3. Temporary files in /tmp must be cleaned up.
4. No source content logged (only metadata).

---

## Observability Requirements

| Metric                         | Dimension   | Value                      |
| ------------------------------ | ----------- | -------------------------- |
| `phase0.duration_ms`           | environment | Execution time             |
| `phase0.source_size_bytes`     | environment | Input file size            |
| `phase0.normalized_size_bytes` | environment | Output Markdown size       |
| `phase0.chunk_count`           | environment | Number of chunks generated |
| `phase0.page_count`            | environment | PDF pages (PDFs only)      |

| Log Field           | Required | Example             |
| ------------------- | -------- | ------------------- |
| `fileVersionId`     | Yes      | `"v-123"`           |
| `phase`             | Yes      | `"phase0"`          |
| `sourceContentType` | Yes      | `"application/pdf"` |
| `sourceSizeBytes`   | Yes      | `1234567`           |
| `chunkCount`        | Yes      | `8`                 |
| `durationMs`        | Yes      | `45000`             |

---

## Verification Plan

1. **Small PDF conversion**
   - Upload 10-page PDF fixture
   - Verify normalized markdown exists
   - Verify chunk count > 0
   - Verify metadata.json contains page count

2. **Markdown passthrough**
   - Upload markdown fixture
   - Verify content preserved
   - Verify chunk boundaries correct

3. **Version validation**
   - Start workflow with valid VersionId → proceeds
   - Start workflow with wrong VersionId → fails cleanly

4. **Large file handling**
   - Upload 50MB PDF
   - Verify Lambda completes within timeout
   - Verify chunks are reasonable size

5. **Edge cases**
   - PDF with no text (images only) → empty or minimal markdown
   - PDF with complex tables → table structure preserved
   - Markdown with code blocks → blocks not split across chunks

---

## Rollout / Rollback

### Rollout

1. Deploy Phase 0 Lambda.
2. Update state machine to include Phase0Normalize task.
3. Test with fixture PDFs before enabling production traffic.

### Rollback

1. Update state machine to skip Phase 0 (go to terminal).
2. In-flight workflows will fail at Phase 0 and go to MarkFailed.
3. Rollback Lambda if needed.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
