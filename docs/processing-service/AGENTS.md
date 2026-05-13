# Processing Service Implementation Guide

> Operating contract for agents implementing the serverless wiki processing service.
> Read this file completely before writing any infrastructure or Lambda code.

---

## Read Order

1. **This file** — implementation rules and boundaries
2. **[SCALABLE-FILE-PROCESSING-HANDOFF.md](../SCALABLE-FILE-PROCESSING-HANDOFF.md)** — Zarya integration contract (source of truth for external interfaces)
3. **[system-contract.md](system-contract.md)** — non-negotiable invariants (must not violate)
4. **[architecture.md](architecture.md)** — system diagram and component inventory
5. **Feature specs** — detailed requirements per feature
6. **[reference/cli-mapping.md](reference/cli-mapping.md)** — CLI tools to learn from

---

## Mission

Implement the serverless processing service on AWS as specified in the design documents.

The CLI tools in `tools/` are **reference implementations** — they solve the same problems locally. Study them for:

- Edge cases already discovered
- Data structures that work
- Validation logic
- Error handling patterns

Do **not** copy them verbatim. The paradigms differ (see Translation Rules below).

---

## Sacred Rules

1. **Handoff doc is source of truth for Zarya integration.** External interfaces, table access, and trigger patterns come from the handoff.
2. **Design docs are requirements.** Feature specs define what to build. Do not invent features.
3. **System contract is inviolable.** Every invariant in system-contract.md must hold.
4. **CLI tools are reference, not specification.** Learn patterns; adapt to serverless.
5. **Interface schemas are contracts.** Events, data models, and S3 layout are exact.
6. **ADRs explain why.** If you want to change an architectural decision, propose an ADR first.
7. **Tests before merge.** Every Lambda must have unit tests. Integration tests for workflows.

---

## Implementation Targets

| Component                  | Location                                    | Language    |
| -------------------------- | ------------------------------------------- | ----------- |
| CloudFormation templates   | `infra/processing-service/cfn/`             | YAML        |
| Step Functions ASL         | `infra/processing-service/sfn/`             | JSON        |
| Lambda functions (Node.js) | `infra/processing-service/lambda/node/`     | TypeScript  |
| Lambda functions (Python)  | `infra/processing-service/lambda/python/`   | Python 3.12 |
| Lambda layers              | `infra/processing-service/layers/`          | Mixed       |
| Shared Python packages     | `packages/wiki_core/`, `packages/wiki_llm/` | Python      |
| Tests                      | `infra/processing-service/tests/`           | Mixed       |

---

## Translation Rules

The CLI tools run locally with human-in-the-loop. The service runs serverlessly with full automation. Apply these translations:

| CLI Pattern                              | Serverless Equivalent                        |
| ---------------------------------------- | -------------------------------------------- |
| Local file system (`raw/`, `wiki/`)      | S3 buckets with prefixes                     |
| `Path.read_text()` / `Path.write_text()` | `s3.get_object()` / `s3.put_object()`        |
| `.wiki-extraction-state/` directory      | S3 `processing/{fileVersionId}/` prefix      |
| `subprocess.run()` for external tools    | Lambda layers with imported modules          |
| Print statements                         | Structured JSON logging                      |
| Command-line arguments                   | Lambda event payload / Step Functions input  |
| Exit codes                               | Lambda response / Step Functions task output |
| Human reviews worktree                   | Automated validation + quality gates         |
| `pnpm wiki:*` commands                   | Step Functions states                        |
| Resume via `--resume` flag               | Step Functions Map state error handling      |
| Local Ollama/Bedrock toggle              | Always Bedrock (via `wiki_llm` package)      |

### S3 Path Translation

| CLI Path                                               | S3 Key                                                      |
| ------------------------------------------------------ | ----------------------------------------------------------- |
| `raw/imported/{slug}/`                                 | N/A (source in Zarya bucket)                                |
| `raw/normalized/{slug}/source.md`                      | `processing/{fileVersionId}/normalized/source.md`           |
| `.wiki-extraction-state/{slug}/claims-raw.jsonl`       | `processing/{fileVersionId}/phase-1/claims-raw.jsonl`       |
| `.wiki-extraction-state/{slug}/claims-normalized.json` | `processing/{fileVersionId}/phase-1/claims-normalized.json` |
| `.wiki-extraction-state/{slug}/candidates.json`        | `processing/{fileVersionId}/phase-1/candidates.json`        |
| `wiki/sources/{slug}.md`                               | `wikis/{folderId}/sources/{fileId}/{fileVersionId}.md`      |
| `wiki/concepts/*.md`                                   | `wikis/{folderId}/concepts/*.md`                            |
| `wiki/index.md`                                        | `wikis/{folderId}/index.md`                                 |
| `wiki/_graph.json`                                     | `wikis/{folderId}/_graph.json`                              |

---

## What to Reuse Directly

These packages are designed for both CLI and Lambda use:

| Package               | Purpose                                   | Import As    |
| --------------------- | ----------------------------------------- | ------------ |
| `packages/wiki_core/` | Types, parsing, validation (no I/O)       | Lambda layer |
| `packages/wiki_llm/`  | Bedrock client, prompts, response parsing | Lambda layer |

These are CLI-only (do not import into Lambda):

| Module                          | Why Not                                                   |
| ------------------------------- | --------------------------------------------------------- |
| `tools/wiki_ingest.py`          | Orchestrates via subprocess; use Step Functions instead   |
| `tools/wiki_phase0_import.py`   | Uses local `marker_single`; use pymupdf4llm layer instead |
| `tools/wiki_phase3_finalize.py` | Runs sequential CLI commands; use Step Functions states   |

---

## Feature Implementation Order

Implement features in dependency order:

| Phase | Features                              | Dependencies |
| ----- | ------------------------------------- | ------------ |
| 1     | 00-foundation                         | None         |
| 2     | 01-intake, 02-phase0                  | 00           |
| 3     | 03-phase1                             | 02           |
| 4     | 04-phase2                             | 03           |
| 5     | 05-phase3                             | 04           |
| 6     | 06-deletion, 07-api, 08-observability | 05           |

For each feature:

1. Read the feature spec completely
2. Check [reference/cli-mapping.md](reference/cli-mapping.md) for CLI equivalents
3. Study the CLI code for patterns and edge cases
4. Write CloudFormation / Lambda / ASL
5. Write tests
6. Validate against feature spec's Verification Plan

---

## Validation Checkpoints

Before marking a feature complete:

- [ ] CloudFormation template passes `cfn-lint`
- [ ] ASL passes `aws stepfunctions validate-state-machine-definition`
- [ ] Lambda unit tests pass
- [ ] All invariants from system-contract.md are preserved
- [ ] All postconditions from feature spec are satisfied
- [ ] Interface schemas match exactly (events.md, data-models.md, s3-layout.md)

---

## Error Handling Patterns

Study these CLI implementations for error handling:

| Pattern                  | CLI Reference                                  | Apply To          |
| ------------------------ | ---------------------------------------------- | ----------------- |
| Chunk failure threshold  | `wiki_deep_extract.py` — 20% failure threshold | Phase 1 Map state |
| Rate limit retry         | `wiki_deep_extract.py` — exponential backoff   | Bedrock calls     |
| Validation + repair loop | `wiki_phase2_single.py` — 3 repair attempts    | Phase 2 synthesis |
| Idempotent writes        | `wiki_log.py` — idempotency_key check          | Wiki log entries  |
| Lock acquisition         | N/A (local is single-process)                  | WikiLocks table   |

---

## Testing Strategy

| Test Type         | Coverage                | Location             |
| ----------------- | ----------------------- | -------------------- |
| Unit tests        | Each Lambda handler     | `tests/unit/`        |
| Integration tests | Step Functions workflow | `tests/integration/` |
| Contract tests    | Event schema validation | `tests/contracts/`   |
| Smoke tests       | Post-deployment health  | `tests/smoke/`       |

Use `wiki_core` test utilities for parsing and validation tests.

---

## Prompt Templates

The CLI tools use prompt templates in `tools/prompts/`. These are directly reusable:

| Prompt              | CLI Location                                        | Lambda Use                |
| ------------------- | --------------------------------------------------- | ------------------------- |
| Claim extraction    | `tools/prompts/` (inline in `wiki_deep_extract.py`) | Phase 1 Extract           |
| Page synthesis      | `tools/prompts/phase2-synthesis.md`                 | Phase 2 Synthesize        |
| Reference synthesis | `tools/prompts/phase2-reference-synthesis.md`       | Phase 2 (reference pages) |

Package these prompts in the `wiki_llm` layer or embed in Lambda code.

---

## When to Deviate from CLI

The CLI has some patterns that don't translate:

| CLI Pattern                | Why It Doesn't Translate | Serverless Approach            |
| -------------------------- | ------------------------ | ------------------------------ |
| Disposable git worktrees   | No git in Lambda         | S3 staging prefix              |
| Interactive repair prompts | No human in loop         | Automated retry + quality gate |
| `--dry-run` flags          | Lambda always executes   | Feature flags in config        |
| Local model fallback       | Bedrock only in prod     | Single backend                 |
| Curator review packets     | No human approval        | Automated validation           |

---

## Observability Mapping

| CLI Report                    | Serverless Equivalent                 |
| ----------------------------- | ------------------------------------- |
| `wiki/_linter-report.md`      | CloudWatch dashboard + S3 report      |
| `wiki/_grounding-report.md`   | Per-job grounding metrics             |
| `wiki/_maintenance-report.md` | Scheduled Lambda + SNS alerts         |
| `wiki/log.md`                 | DynamoDB ProcessingJobs + S3 wiki log |

---

## Questions Protocol

If the design docs are ambiguous:

1. Check if CLI tools already handle the case
2. Check ADRs for rationale
3. If still unclear, add to feature spec's "Open Questions" section
4. Propose a resolution before implementing

Do not guess. Document uncertainty.

---

## Related Documents

| Document                                             | Purpose                    |
| ---------------------------------------------------- | -------------------------- |
| [README.md](README.md)                               | Design document navigation |
| [system-contract.md](system-contract.md)             | Inviolable invariants      |
| [reference/cli-mapping.md](reference/cli-mapping.md) | CLI → serverless mapping   |
| [/AGENTS.md](/AGENTS.md)                             | Repo-level operating rules |
