# CLI Tool Mapping

> Maps processing service features to existing CLI tools.
> Study these implementations for patterns, edge cases, and proven solutions.

---

## How to Use This Document

For each feature you implement:

1. Find the feature in the table below
2. Read the linked CLI tool code
3. Note the "Learn From" column — these are specific patterns to study
4. Note the "Differs Because" column — these require adaptation

---

## Feature → CLI Mapping

### Feature 02: Phase 0 Normalization

| Design Component     | CLI Equivalent                | Learn From                                       | Differs Because                                                      |
| -------------------- | ----------------------------- | ------------------------------------------------ | -------------------------------------------------------------------- |
| PDF → Markdown       | `tools/wiki_phase0_import.py` | `clean_pdf_artifacts()` — PDF extraction cleanup | CLI uses `marker_single` subprocess; Lambda uses `pymupdf4llm` layer |
| Markdown passthrough | `tools/wiki_phase0_import.py` | Suffix detection, copy logic                     | Same pattern works                                                   |
| Idempotent import    | `tools/wiki_phase0_import.py` | `--reuse-imported` hash check                    | S3 VersionId replaces hash check                                     |
| Chunking             | `tools/wiki_deep_extract.py`  | `chunk_normalized_source()`                      | Same logic, different I/O                                            |

**Key code to study:**

```python
# tools/wiki_phase0_import.py:17-35
def clean_pdf_artifacts(text: str) -> str:
    """Clean up PDF extraction artifacts from normalized markdown.

    PDF extractors sometimes soft-wrap long lines with backslash-space or
    backslash-newline sequences. These cause JSON parse errors when the
    model quotes them as evidence.
    """
    text = re.sub(r'\\ ', '', text)
    text = re.sub(r'\\\n', '', text)
    return text
```

**Reuse directly:** This function should be in `wiki_core` and used by Lambda.

---

### Feature 03: Phase 1 Claim Extraction

| Design Component   | CLI Equivalent               | Learn From                                | Differs Because                        |
| ------------------ | ---------------------------- | ----------------------------------------- | -------------------------------------- |
| Chunked extraction | `tools/wiki_deep_extract.py` | `run_deep_extraction()` main loop         | CLI is sequential; Lambda is Map state |
| Claim schema       | `packages/wiki_io/state.py`  | `RawClaim`, `NormalizedClaim` dataclasses | Reuse directly                         |
| Extraction prompt  | `tools/wiki_deep_extract.py` | `EXTRACTION_PROMPT` template              | Reuse directly                         |
| Claim parsing      | `tools/wiki_deep_extract.py` | `parse_extraction_response()`             | Reuse directly                         |
| Deduplication      | `tools/wiki_deep_extract.py` | `normalize_and_cluster_claims()`          | Reuse directly                         |
| Topic clustering   | `tools/wiki_deep_extract.py` | Topic assignment logic                    | Reuse directly                         |
| Failure threshold  | `tools/wiki_deep_extract.py` | 20% chunk failure threshold               | Same threshold, Step Functions check   |
| Resume support     | `tools/wiki_deep_extract.py` | Chunk status tracking in state file       | S3 markers + Map state ResultSelector  |
| Rate limiting      | `tools/wiki_deep_extract.py` | Bedrock retry with backoff                | Use RateLimits DynamoDB table          |

**Key code to study:**

```python
# packages/wiki_io/state.py
@dataclass
class RawClaim:
    text: str
    evidence: str
    locator: str
    topic: str
    confidence: float
    chunk_index: int
```

```python
# tools/wiki_deep_extract.py — extraction prompt structure
EXTRACTION_PROMPT = """Extract factual claims from this text chunk...
Return JSON array of claims with: text, evidence, locator, topic, confidence
"""
```

**Reuse directly:**

- `RawClaim` and `NormalizedClaim` from `wiki_io.state`
- Extraction prompt template
- `parse_extraction_response()` logic

---

### Feature 04: Phase 2 Page Synthesis

| Design Component      | CLI Equivalent                                | Learn From                            | Differs Because            |
| --------------------- | --------------------------------------------- | ------------------------------------- | -------------------------- |
| Synthesis prompt      | `tools/prompts/phase2-synthesis.md`           | Full prompt template                  | Reuse directly             |
| Reference page prompt | `tools/prompts/phase2-reference-synthesis.md` | Reference-specific template           | Reuse directly             |
| Evidence bank         | `tools/wiki_phase2_benchmark.py`              | `build_evidence_bank()`               | Reuse pattern              |
| Claim clustering      | `tools/wiki_phase2_single.py`                 | `select_claims_for_page()`            | Reuse pattern              |
| Page validation       | `tools/wiki_check_synthesis.py`               | `check_synthesis_structured()`        | Reuse directly via layer   |
| Repair loop           | `tools/wiki_phase2_single.py`                 | 3-attempt repair with re-prompt       | Same pattern, Lambda retry |
| Quality filtering     | `tools/wiki_phase2_single.py`                 | Min claims threshold                  | Same logic                 |
| Schema validation     | `tools/wiki_page_schema.py`                   | `WikiPageSchema`, `validate_schema()` | Reuse directly via layer   |

**Key code to study:**

```python
# tools/wiki_phase2_single.py — repair loop pattern
for attempt in range(max_repair_attempts):
    result = synthesize_page(claims, prompt)
    validation = run_validation(result)
    if validation.passed:
        return result
    result = repair_page(result, validation.errors)
```

```python
# tools/wiki_check_synthesis.py — validation checks
def check_synthesis_structured(page_path, source_slug):
    """Returns structured validation result with:
    - frontmatter_valid
    - has_source_backed_details
    - evidence_ids_valid
    - backlinks_valid
    """
```

**Reuse directly:**

- Prompt templates from `tools/prompts/`
- `WikiPageSchema` from `wiki_page_schema.py`
- Validation logic from `wiki_check_synthesis.py`

---

### Feature 05: Phase 3 Wiki Maintenance

| Design Component   | CLI Equivalent        | Learn From              | Differs Because        |
| ------------------ | --------------------- | ----------------------- | ---------------------- |
| Index rebuild      | `tools/wiki_index.py` | `rebuild_index()`       | Same logic, S3 I/O     |
| Graph rebuild      | `tools/wiki_graph.py` | `rebuild_graph()`       | Same logic, S3 I/O     |
| Log append         | `tools/wiki_log.py`   | `append_log_entry()`    | Same format, S3 append |
| Idempotent log     | `tools/wiki_log.py`   | `idempotency_key` check | Same pattern           |
| Lint validation    | `tools/wiki_lint.py`  | `run_lint()`            | Run post-commit        |
| FIFO serialization | N/A (single process)  | N/A                     | New for serverless     |
| WikiLocks          | N/A (single process)  | N/A                     | New for serverless     |

**Key code to study:**

```python
# tools/wiki_index.py — index generation
def rebuild_index():
    """Scan wiki/**/*.md, extract frontmatter, generate index.md"""
    entries = []
    for page in iter_content_pages():
        fm = parse_frontmatter(page)
        entries.append(IndexEntry(
            title=fm.data.get('title'),
            path=page,
            type=fm.data.get('type'),
            summary=extract_summary(fm.body)
        ))
    return render_index(entries)
```

```python
# tools/wiki_log.py — idempotent append
def append_log_entry(operation, summary, details, idempotency_key=None):
    log = Path("wiki/log.md").read_text()
    if idempotency_key and idempotency_key in log:
        return  # Already logged
    entry = render_entry(operation, summary, details)
    Path("wiki/log.md").write_text(log + entry)
```

**Reuse directly:**

- Index generation logic from `wiki_index.py`
- Graph generation logic from `wiki_graph.py`
- Log entry format from `wiki_log.py`

**New for serverless:**

- FIFO queue serialization (no CLI equivalent)
- DynamoDB locking (no CLI equivalent)

---

### Feature 06: File Deletion / Archive

| Design Component      | CLI Equivalent           | Learn From                      | Differs Because    |
| --------------------- | ------------------------ | ------------------------------- | ------------------ |
| Source removal        | `tools/wiki_uningest.py` | `remove_source_contributions()` | Same logic, S3 I/O |
| Sole-source detection | `tools/wiki_uningest.py` | Page source counting            | Same logic         |
| Index/graph update    | `tools/wiki_uningest.py` | Rebuild after removal           | Same pattern       |

**Key code to study:**

```python
# tools/wiki_uningest.py
def remove_source_contributions(source_slug):
    """Remove pages where source is sole contributor.
    Update pages where source is one of multiple contributors.
    """
    for page in find_pages_citing_source(source_slug):
        sources = page.frontmatter.get('sources', [])
        if len(sources) == 1:
            delete_page(page)
        else:
            remove_source_from_page(page, source_slug)
```

---

### Feature 08: Observability

| Design Component   | CLI Equivalent              | Learn From                | Differs Because    |
| ------------------ | --------------------------- | ------------------------- | ------------------ |
| Lint checks        | `tools/wiki_lint.py`        | All check functions       | Adapt for S3 paths |
| Grounding checks   | `tools/wiki_grounding.py`   | Token overlap scoring     | Reuse directly     |
| Maintenance runner | `tools/wiki_maintenance.py` | Orchestration of checks   | Scheduled Lambda   |
| Report format      | `wiki/_linter-report.md`    | Markdown report structure | S3 + CloudWatch    |

**Key code to study:**

```python
# tools/wiki_lint.py — issue structure
@dataclass(frozen=True)
class Issue:
    level: str  # PASS, WARN, FAIL, TODO
    path: str
    message: str
```

---

## Packages to Include in Lambda Layers

### wiki_core layer

| Module                    | Size  | Purpose                       |
| ------------------------- | ----- | ----------------------------- |
| `wiki_core/types.py`      | ~5KB  | Dataclasses for claims, pages |
| `wiki_core/parsing.py`    | ~10KB | Frontmatter, markdown parsing |
| `wiki_core/validation.py` | ~8KB  | Schema validation             |

### wiki_llm layer

| Module                  | Size  | Purpose          |
| ----------------------- | ----- | ---------------- |
| `wiki_llm/backends.py`  | ~15KB | Bedrock client   |
| `wiki_llm/prompts.py`   | ~5KB  | Prompt templates |
| `wiki_llm/responses.py` | ~10KB | Response parsing |

---

## Data Structures to Reuse

These dataclasses from CLI are directly usable in Lambda:

```python
# From packages/wiki_io/state.py
RawClaim          # Per-chunk extraction output
NormalizedClaim   # Deduplicated claim
Candidate         # Page candidate from clustering
Manifest          # Extraction state tracking

# From tools/wiki_page_schema.py
WikiPageSchema    # Validated page structure
ParseResult       # LLM response parsing result

# From tools/wiki_lint.py
Issue             # Lint/validation issue
```

---

## Prompt Templates to Reuse

| Template            | Location                                      | Used By         |
| ------------------- | --------------------------------------------- | --------------- |
| Claim extraction    | `wiki_deep_extract.py` (inline)               | Phase 1 Lambda  |
| Page synthesis      | `tools/prompts/phase2-synthesis.md`           | Phase 2 Lambda  |
| Reference synthesis | `tools/prompts/phase2-reference-synthesis.md` | Phase 2 Lambda  |
| Source page repair  | `tools/prompts/phase1-source-repair.md`       | Phase 2a Lambda |

---

## Edge Cases Discovered in CLI

These edge cases are already handled in CLI tools — ensure Lambda handles them too:

| Edge Case                     | CLI Handling                 | Location                  |
| ----------------------------- | ---------------------------- | ------------------------- |
| PDF backslash-space artifacts | `clean_pdf_artifacts()`      | `wiki_phase0_import.py`   |
| Empty chunk extraction        | Skip, don't fail             | `wiki_deep_extract.py`    |
| LLM returns non-JSON          | Retry with structured prompt | `wiki_deep_extract.py`    |
| Claim without evidence        | Filter out                   | `wiki_deep_extract.py`    |
| Circular wiki links           | Detect and warn              | `wiki_lint.py`            |
| Missing frontmatter           | Fail validation              | `wiki_check_synthesis.py` |
| Evidence ID not found         | Repair or fail               | `wiki_fill_evidence.py`   |
| Rate limit (429)              | Exponential backoff          | `wiki_llm/backends.py`    |

---

## Anti-Patterns from CLI

These CLI patterns should NOT be copied:

| CLI Pattern            | Why Not             | Better Approach       |
| ---------------------- | ------------------- | --------------------- |
| `subprocess.run()`     | No shell in Lambda  | Direct module import  |
| `Path().read_text()`   | Local filesystem    | S3 client             |
| `git init` / worktrees | No git in Lambda    | S3 staging prefix     |
| Interactive `input()`  | No stdin            | Automated with config |
| `sys.exit()`           | Kills Lambda        | Return/raise          |
| Global state mutation  | Lambda reuse issues | Pass state explicitly |
| `print()` for logging  | Not structured      | `structlog` or JSON   |
