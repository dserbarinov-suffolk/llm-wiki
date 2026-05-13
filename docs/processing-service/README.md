# Wiki Processing Service

> Design documentation for the serverless wiki processing service.

---

## Overview

The wiki processing service transforms uploaded documents into persistent, structured wiki content. Each Zarya folder maintains its own wiki—a collection of interlinked Markdown pages synthesized from source documents.

**Key capabilities:**

- PDF-to-Markdown normalization
- LLM-powered claim extraction and page synthesis
- Automated wiki maintenance (index, graph, cross-links)
- API for wiki content access
- Folder-level authorization via Zarya

---

## For Implementers

> **Start here if you're implementing this service.**

| Document                                             | Purpose                               | Read Time |
| ---------------------------------------------------- | ------------------------------------- | --------- |
| [**AGENTS.md**](AGENTS.md)                           | Operating contract for implementation | 10 min    |
| [reference/cli-mapping.md](reference/cli-mapping.md) | CLI tools to learn from               | 15 min    |

The existing CLI tools in `tools/` solve the same problems locally. Study them for patterns, edge cases, and proven solutions before implementing Lambda equivalents.

---

## Quick Navigation

| Document                                 | Purpose                               | Read Time |
| ---------------------------------------- | ------------------------------------- | --------- |
| [Architecture Overview](architecture.md) | System diagram, components, data flow | 10 min    |
| [System Contract](system-contract.md)    | Non-negotiable invariants and rules   | 5 min     |

### Feature Specifications

| Feature                                                                 | Scope                              |
| ----------------------------------------------------------------------- | ---------------------------------- |
| [00 - Foundation](features/00-foundation.md)                            | Durable infrastructure resources   |
| [01 - Intake and Job Tracking](features/01-intake-and-job-tracking.md)  | Stream consumption, workflow start |
| [02 - Phase 0 Normalization](features/02-phase0-normalization.md)       | PDF → Markdown conversion          |
| [03 - Phase 1 Claim Extraction](features/03-phase1-claim-extraction.md) | LLM claim extraction               |
| [04 - Phase 2 Page Synthesis](features/04-phase2-page-synthesis.md)     | LLM page synthesis                 |
| [05 - Phase 3 Wiki Maintenance](features/05-phase3-wiki-maintenance.md) | FIFO serialization, wiki writes    |
| [06 - File Deletion and Archive](features/06-file-deletion-archive.md)  | Source removal handling            |
| [07 - Read API and Auth](features/07-read-api-and-auth.md)              | API endpoints, Zarya auth          |
| [08 - Observability and Ops](features/08-observability-and-ops.md)      | Logging, metrics, alarms           |

### Interface Definitions

| Interface                                    | Defines                              |
| -------------------------------------------- | ------------------------------------ |
| [Events](interfaces/events.md)               | Event schemas for all integrations   |
| [Data Models](interfaces/data-models.md)     | DynamoDB table schemas               |
| [S3 Layout](interfaces/s3-layout.md)         | Bucket structure, object key schemas |
| [Status Model](interfaces/status-model.md)   | Job and phase status transitions     |
| [Auth Contract](interfaces/auth-contract.md) | Authorization API with Zarya         |

### Operations

| Document                                                  | Purpose                               |
| --------------------------------------------------------- | ------------------------------------- |
| [Deployment Requirements](ops/deployment-requirements.md) | Stack boundaries, retention, ordering |
| [CI/CD Requirements](ops/ci-cd-requirements.md)           | Jobs, gates, artifact naming          |
| [Drift Detection](ops/drift-detection.md)                 | Infrastructure drift monitoring       |
| [Cost Model](ops/cost-model.md)                           | Per-PDF and monthly estimates         |

### Architecture Decisions

| ADR                                                          | Decision                         |
| ------------------------------------------------------------ | -------------------------------- |
| [ADR-001](decisions/adr-001-bedrock-model.md)                | AWS Bedrock with Qwen3-Coder 30b |
| [ADR-002](decisions/adr-002-step-functions-orchestration.md) | Step Functions for orchestration |
| [ADR-003](decisions/adr-003-s3-wiki-storage.md)              | S3 for wiki storage              |
| [ADR-004](decisions/adr-004-folder-level-wiki-auth.md)       | Folder-level authorization       |
| [ADR-005](decisions/adr-005-phase3-fifo-serialization.md)    | FIFO + lock for serialization    |

---

## Key Concepts

### fileVersionId (Trace ID)

Every processing job is identified by `fileVersionId` from Zarya. This ID:

- Becomes the Step Functions execution name
- Appears in all logs and metrics
- Links artifacts across S3, DynamoDB, and CloudWatch

### folderId (Aggregate Key)

Wiki content is scoped to Zarya folders:

- Each folder has one wiki
- Authorization checked per folder
- Wiki writes serialized per folder

### Processing Phases

| Phase   | Input           | Output            | LLM? |
| ------- | --------------- | ----------------- | ---- |
| Phase 0 | PDF             | Markdown + chunks | No   |
| Phase 1 | Markdown chunks | Extracted claims  | Yes  |
| Phase 2 | Claims          | Synthesized pages | Yes  |
| Phase 3 | Pages           | Updated wiki      | No   |

---

## Reading Order

For implementers:

1. [AGENTS.md](AGENTS.md) - Operating contract and translation rules
2. [System Contract](system-contract.md) - Understand invariants
3. [Architecture](architecture.md) - See the big picture
4. [reference/cli-mapping.md](reference/cli-mapping.md) - CLI tools to study
5. Feature specs in dependency order (see AGENTS.md)
6. Interface definitions as needed

For reviewers:

1. [System Contract](system-contract.md) - Verify invariants are complete
2. Feature specs - Each should be reviewable in < 15 minutes
3. ADRs - Validate decision rationale

---

## Document Conventions

### Status Badges

| Badge     | Meaning                               |
| --------- | ------------------------------------- |
| 🔴 Draft  | In progress, may change significantly |
| 🟡 Review | Ready for review                      |
| 🟢 Stable | Approved, changes require ADR         |

Currently all documents are **🔴 Draft** pending initial review.

### Cross-References

Documents reference each other using relative Markdown links:

```markdown
See [Events](../interfaces/events.md) for the full schema.
```

### Change Tracking

Each document includes a changelog at the bottom. Breaking changes should be accompanied by an ADR.

---

## Contributing

### Adding a Feature

1. Create `features/NN-feature-name.md` using the template in existing features
2. Update this README's navigation table
3. Add interface definitions if needed
4. Create ADR if the feature requires architectural decisions

### Modifying Interfaces

1. Update the interface document
2. Update all features that consume the interface
3. Consider backward compatibility for deployed systems

### Proposing Architecture Changes

1. Create an ADR in `decisions/`
2. Reference the ADR in affected documents
3. Update architecture.md if the change affects the system diagram

---

## Related

### Integration Contract

- **[SCALABLE-FILE-PROCESSING-HANDOFF.md](../SCALABLE-FILE-PROCESSING-HANDOFF.md)** - Zarya integration contract (source of truth for external interfaces)

### Implementation Resources

- [AGENTS.md](AGENTS.md) - Operating contract for implementation
- [reference/cli-mapping.md](reference/cli-mapping.md) - CLI tools → serverless mapping
- [tools/](../../tools/) - CLI reference implementations
- [packages/wiki_core/](../../packages/wiki_core/) - Reusable types and parsing (for Lambda layers)
- [packages/wiki_llm/](../../packages/wiki_llm/) - Bedrock client and prompts (for Lambda layers)

### Background Documents

- [DESIGN-scalable-processing.md](../DESIGN-scalable-processing.md) - Original monolithic design (deprecated)
- [INGESTION-FLOW.md](../INGESTION-FLOW.md) - Local wiki ingestion workflow
- [MODEL-BACKENDS.md](../MODEL-BACKENDS.md) - LLM backend configuration
- [/AGENTS.md](../../AGENTS.md) - Repo-level operating rules

---

## Changelog

| Date       | Change                                         |
| ---------- | ---------------------------------------------- |
| 2026-05-13 | Integrated SCALABLE-FILE-PROCESSING-HANDOFF.md |
| 2026-05-13 | Added AGENTS.md and reference/cli-mapping.md   |
| 2026-05-13 | Initial split from monolithic design           |
