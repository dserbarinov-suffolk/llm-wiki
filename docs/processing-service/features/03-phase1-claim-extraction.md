# Feature 03: Phase 1 - Claim Extraction

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - `fileVersionId` is the trace ID and workflow idempotency key.
> - `taskId = {fileVersionId}#{phase}#{slug}` identifies slug-level work units.
> - Derived data has the same sensitivity classification as source content.

---

## Goal

Run LLM extraction per chunk, store per-chunk outputs, aggregate and normalize claims, and generate candidate pages for Phase 2.

---

## Scope

### Included

- Phase 1 Extract Chunk Lambda (per-chunk)
- Phase 1 Aggregate Claims Lambda
- Bedrock LLM integration for extraction
- Rate limiting via token bucket
- Chunk result collection (Step Functions Map)
- Failure threshold enforcement
- Candidate page generation

### Excluded

- Page synthesis (Feature 04)
- Wiki maintenance (Feature 05)

---

## Preconditions

- Feature 02 deployed (Phase 0 produces chunks)
- Rate limiter table initialized with model config
- Bedrock model access configured
- Processing bucket exists

---

## Postconditions

After Phase 1 completes successfully:

| Artifact                     | Location                                         |
| ---------------------------- | ------------------------------------------------ |
| Per-chunk extraction results | `{fileVersionId}/phase-1/chunks/{index}.json`    |
| Raw claims (all chunks)      | `{fileVersionId}/phase-1/claims-raw.jsonl`       |
| Normalized claims            | `{fileVersionId}/phase-1/claims-normalized.json` |
| Candidate pages              | `{fileVersionId}/phase-1/candidates.json`        |
| Job currentPhase             | `phase2`                                         |

---

## Interfaces Consumed

| Interface           | Location                                                       |
| ------------------- | -------------------------------------------------------------- |
| Phase0Output        | [`../interfaces/events.md`](../interfaces/events.md)           |
| Normalized Markdown | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md)     |
| RateLimits table    | [`../interfaces/data-models.md`](../interfaces/data-models.md) |

## Interfaces Produced

| Interface               | Location                                             |
| ----------------------- | ---------------------------------------------------- |
| Phase1ChunkResult       | [`../interfaces/events.md`](../interfaces/events.md) |
| Phase1AggregationOutput | [`../interfaces/events.md`](../interfaces/events.md) |

---

## Functional Requirements

### Per-Chunk Extraction

1. Load chunk text from normalized source using line boundaries.
2. Acquire rate limit tokens before LLM call.
3. Call Bedrock with extraction prompt.
4. Parse structured claims from LLM response.
5. Store chunk result with status, claimCount, or error.
6. Return Phase1ChunkResult envelope.

### Claim Structure

Each extracted claim must include:

```typescript
interface RawClaim {
  text: string; // Claim statement
  evidence: string; // Exact quote from source
  locator: string; // Line reference (e.g., "normalized:L12-L15")
  topic: string; // Suggested topic grouping
  confidence: number; // 0.0-1.0
}
```

### Failure Handling in Map State

1. Chunk failures do not fail the Map state.
2. Failed chunks return `{status: "failed", error: "..."}`.
3. Aggregator receives all results (successful and failed).

### Failure Threshold

1. If >20% of chunks failed extraction, fail the job.
2. Throw `TooManyChunkFailures` error.
3. Workflow transitions to MarkFailed.

### Claim Aggregation

1. Collect claims from successful chunks only.
2. Deduplicate claims by semantic similarity.
3. Normalize claim text and evidence.
4. Assign stable claim IDs for reference.
5. Group claims by topic.
6. Generate candidate pages with minimum claim threshold.

### Candidate Generation

1. Group claims by topic.
2. Require minimum 3 claims per topic to create a candidate.
3. Determine page category (concept, entity, procedure, reference).
4. Generate slug and pagePath.
5. Include claimIds for traceability.

---

## Failure Handling

| Failure                  | Handling                                  |
| ------------------------ | ----------------------------------------- |
| Rate limit exhausted     | Retry with backoff (RateLimitError)       |
| Bedrock throttling       | Retry with exponential backoff            |
| LLM response unparseable | Mark chunk failed, continue               |
| >20% chunks failed       | Terminal failure: TOO_MANY_CHUNK_FAILURES |
| Lambda timeout           | Mark chunk failed, continue               |

---

## Security Requirements

1. Lambda must have InvokeModel permission on Bedrock.
2. Lambda must have read/write access to Processing bucket.
3. Lambda must have read/write access to RateLimits table.
4. LLM prompts must not include user PII.
5. LLM responses must be validated before storage.

---

## Observability Requirements

| Metric                            | Dimension           | Value                      |
| --------------------------------- | ------------------- | -------------------------- |
| `phase1.chunk.duration_ms`        | environment, model  | Per-chunk extraction time  |
| `phase1.chunk.input_tokens`       | environment, model  | Tokens sent to LLM         |
| `phase1.chunk.output_tokens`      | environment, model  | Tokens received from LLM   |
| `phase1.chunk.claim_count`        | environment         | Claims extracted per chunk |
| `phase1.chunk.status`             | environment, status | succeeded/failed count     |
| `phase1.aggregation.total_claims` | environment         | Total claims after dedup   |
| `phase1.aggregation.candidates`   | environment         | Candidate pages generated  |
| `phase1.rate_limit.wait_ms`       | environment, model  | Time waiting for tokens    |

| Log Field       | Required | Example                   |
| --------------- | -------- | ------------------------- |
| `fileVersionId` | Yes      | `"v-123"`                 |
| `taskId`        | Yes      | `"v-123#extractClaims#0"` |
| `phase`         | Yes      | `"phase1"`                |
| `chunkIndex`    | Yes      | `0`                       |
| `inputTokens`   | Yes      | `5000`                    |
| `outputTokens`  | Yes      | `2000`                    |
| `claimCount`    | Yes      | `15`                      |

---

## Verification Plan

1. **Basic extraction**
   - Process 3-chunk fixture
   - Verify all chunks produce results
   - Verify claims-normalized.json exists
   - Verify candidates.json has entries

2. **Partial failure**
   - Simulate 1 of 10 chunks failing
   - Verify workflow continues
   - Verify failed chunk logged
   - Verify aggregator receives 9 successful results

3. **Threshold failure**
   - Simulate 3 of 10 chunks failing (30%)
   - Verify workflow fails with TOO_MANY_CHUNK_FAILURES

4. **Rate limiting**
   - Exhaust token bucket
   - Verify chunks wait and retry
   - Verify rate_limit.wait_ms metric emitted

5. **Claim deduplication**
   - Process source with repeated content
   - Verify duplicate claims are merged
   - Verify stable IDs assigned

---

## Rollout / Rollback

### Rollout

1. Initialize rate limiter with conservative limits.
2. Deploy Phase 1 Lambda functions.
3. Update state machine with Phase1ExtractClaims Map state.
4. Test with small fixtures before production traffic.
5. Gradually increase rate limits based on observed throttling.

### Rollback

1. Update state machine to skip Phase 1.
2. In-flight workflows will fail at Phase 1.
3. Rollback Lambda if needed.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
