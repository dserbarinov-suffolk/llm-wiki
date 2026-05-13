# Feature 04: Phase 2 - Page Synthesis

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - `fileVersionId` is the trace ID and workflow idempotency key.
> - `taskId = {fileVersionId}#{phase}#{slug}` identifies slug-level work units.
> - Derived data has the same sensitivity classification as source content.

---

## Goal

Synthesize wiki pages from candidate topics, validate/repair pages, split results into success/failure, and proceed only with successful pages.

---

## Scope

### Included

- Phase 2 Synthesize Page Lambda (per-candidate)
- Filter Phase 2 Results Lambda
- Bedrock LLM integration for synthesis
- Page validation and repair
- Claim judge (optional quality gate)
- Success/failure result splitting

### Excluded

- Wiki maintenance (Feature 05)
- Contradiction detection (handled in Feature 05)

---

## Preconditions

- Feature 03 deployed (Phase 1 produces candidates)
- Claims normalized and stored
- Bedrock model access configured
- wiki-llm layer available

---

## Postconditions

After Phase 2 completes:

| Condition            | When                                                         |
| -------------------- | ------------------------------------------------------------ |
| All pages succeeded  | status = `succeeded`, proceed to Phase 3                     |
| Some pages failed    | status = `partial`, proceed to Phase 3 with successful pages |
| Zero pages succeeded | status = `failed`, workflow terminates                       |

| Artifact          | Location                                    |
| ----------------- | ------------------------------------------- |
| Synthesized pages | `{fileVersionId}/phase-2/{slug}.md`         |
| Judge verdicts    | `{fileVersionId}/phase-2/{slug}.judge.json` |

---

## Interfaces Consumed

| Interface               | Location                                                   |
| ----------------------- | ---------------------------------------------------------- |
| Phase1AggregationOutput | [`../interfaces/events.md`](../interfaces/events.md)       |
| Normalized Markdown     | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |
| Claims normalized       | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |

## Interfaces Produced

| Interface             | Location                                                   |
| --------------------- | ---------------------------------------------------------- |
| Phase2PageResult      | [`../interfaces/events.md`](../interfaces/events.md)       |
| FilteredPhase2Output  | [`../interfaces/events.md`](../interfaces/events.md)       |
| Wiki page frontmatter | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md) |

---

## Functional Requirements

### Per-Page Synthesis

1. Load claims for the candidate topic.
2. Build evidence bank from normalized source.
3. Acquire rate limit tokens before LLM call.
4. Call Bedrock with synthesis prompt.
5. Parse page content from LLM response.
6. Validate page against schema.
7. Repair page if validation fails.
8. Judge claims (optional quality gate).
9. Store page with metadata.
10. Return Phase2PageResult envelope.

### Page Validation

Each synthesized page must:

1. Have valid YAML frontmatter with required fields.
2. Have a title heading.
3. Have at least one claim with evidence.
4. Not have empty sections.
5. Not have duplicate headings.
6. Link to at least one source page (in `sources` frontmatter).

### Page Repair

If validation fails:

1. Attempt automatic repair for common issues.
2. Re-validate after repair.
3. If still invalid, mark page as failed.

### Claim Judge (Optional)

If `FEATURE_FLAG_JUDGE_QUALITY_GATE` is enabled:

1. Judge each claim against evidence.
2. Flag claims without grounding.
3. Store judge verdict.
4. Page passes if >80% claims grounded.

### Result Filtering

Filter Phase 2 Results Lambda:

1. Collect all Phase2PageResult from Map state.
2. Split into successfulPages and failedPages.
3. Determine overall status:
   - `succeeded`: All pages succeeded
   - `partial`: Some pages succeeded, some failed
   - `failed`: Zero pages succeeded
4. Return FilteredPhase2Output.

### Failure Behavior

1. Zero successful pages → workflow transitions to MarkFailed.
2. Some successful pages → workflow proceeds to Phase 3 with partial status.
3. Phase 3 receives only successful pages.

---

## Failure Handling

| Failure                       | Handling                               |
| ----------------------------- | -------------------------------------- |
| Rate limit exhausted          | Retry with backoff (RateLimitError)    |
| LLM response unparseable      | Mark page failed, continue             |
| Validation fails after repair | Mark page failed, continue             |
| Judge fails page              | Mark page failed (if gate enabled)     |
| Zero pages succeed            | Terminal failure: NO_PAGES_SYNTHESIZED |
| Lambda timeout                | Mark page failed, continue             |

---

## Security Requirements

1. Lambda must have InvokeModel permission on Bedrock.
2. Lambda must have read/write access to Processing bucket.
3. Lambda must have read/write access to RateLimits table.
4. LLM prompts must not include user PII.
5. LLM responses must be validated before storage.

---

## Observability Requirements

| Metric                           | Dimension            | Value                      |
| -------------------------------- | -------------------- | -------------------------- |
| `phase2.page.duration_ms`        | environment, model   | Per-page synthesis time    |
| `phase2.page.input_tokens`       | environment, model   | Tokens sent to LLM         |
| `phase2.page.output_tokens`      | environment, model   | Tokens received from LLM   |
| `phase2.page.claim_count`        | environment          | Claims in synthesized page |
| `phase2.page.status`             | environment, status  | succeeded/failed count     |
| `phase2.page.judge_verdict`      | environment, verdict | pass/fail/skip count       |
| `phase2.filter.status`           | environment, status  | succeeded/partial/failed   |
| `phase2.filter.successful_pages` | environment          | Count of successful pages  |
| `phase2.filter.failed_pages`     | environment          | Count of failed pages      |

| Log Field       | Required | Example                      |
| --------------- | -------- | ---------------------------- |
| `fileVersionId` | Yes      | `"v-123"`                    |
| `taskId`        | Yes      | `"v-123#synthesize#topic-a"` |
| `phase`         | Yes      | `"phase2"`                   |
| `slug`          | Yes      | `"topic-a"`                  |
| `category`      | Yes      | `"concept"`                  |
| `inputTokens`   | Yes      | `8000`                       |
| `outputTokens`  | Yes      | `4000`                       |
| `claimCount`    | Yes      | `12`                         |
| `judgeVerdict`  | If judge | `"pass"`                     |

---

## Verification Plan

1. **All pages succeed**
   - Process source with 3 clean topics
   - Verify all 3 pages synthesized
   - Verify filter status = `succeeded`
   - Verify workflow proceeds to Phase 3

2. **Partial success**
   - Simulate 1 of 3 pages failing validation
   - Verify filter status = `partial`
   - Verify 2 successful pages passed to Phase 3
   - Verify failed page recorded

3. **All pages fail**
   - Simulate all pages failing
   - Verify filter status = `failed`
   - Verify workflow transitions to MarkFailed

4. **Page validation**
   - Synthesize page with missing frontmatter
   - Verify repair attempts
   - Verify final validation result

5. **Judge quality gate** (if enabled)
   - Synthesize page with weak evidence
   - Verify judge verdict
   - Verify page fails if below threshold

---

## Rollout / Rollback

### Rollout

1. Deploy Phase 2 Lambda functions with judge disabled.
2. Update state machine with Phase2Synthesize Map state.
3. Test with fixtures before production traffic.
4. Enable judge quality gate after baseline established.

### Rollback

1. Update state machine to skip Phase 2.
2. In-flight workflows will fail at Phase 2.
3. Rollback Lambda if needed.

### Feature Flags

| Flag                              | Default | Effect                        |
| --------------------------------- | ------- | ----------------------------- |
| `FEATURE_FLAG_JUDGE_QUALITY_GATE` | false   | Enable claim judge as gate    |
| `JUDGE_GROUNDING_THRESHOLD`       | 0.8     | Min % claims grounded to pass |

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
