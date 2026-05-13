# Status Model

> Centralized status values, transitions, and display mappings.
> This file is the authoritative reference for all status-related logic.

---

## Job Status

### Values

| Status       | Terminal | Description                                    |
| ------------ | -------- | ---------------------------------------------- |
| `queued`     | No       | Job created, waiting for workflow to start     |
| `running`    | No       | Workflow actively processing                   |
| `blocked`    | No       | Temporary failure, will retry                  |
| `partial`    | Yes      | Completed with some pages failed               |
| `succeeded`  | Yes      | All phases completed successfully              |
| `failed`     | Yes      | Unrecoverable error                            |
| `superseded` | Yes      | Newer version completed, this version archived |

### State Diagram

```
              ┌───────────────────────────────────────────────────┐
              │                                                   │
              ▼                                                   │
         ┌────────┐                                              │
         │ queued │                                              │
         └────┬───┘                                              │
              │ workflow starts                                  │
              ▼                                                   │
         ┌────────┐         retryable error         ┌─────────┐ │
         │running │ ─────────────────────────────▶  │ blocked │ │
         └────┬───┘                                 └────┬────┘ │
              │                                          │       │
              │ ◀────────────────────────────────────────┘       │
              │         retry succeeds                           │
              │                                                   │
    ┌─────────┼─────────┬─────────────┬───────────────┐         │
    │         │         │             │               │         │
    ▼         ▼         ▼             ▼               ▼         │
┌───────┐ ┌───────┐ ┌───────┐ ┌───────────┐ ┌───────────────┐   │
│partial│ │success│ │failed │ │superseded │ │ (still running)│───┘
└───────┘ └───────┘ └───────┘ └───────────┘ └───────────────┘
    │         │         │             │
    └─────────┴─────────┴─────────────┘
              │
         TERMINAL
```

### Transition Rules

| From      | To           | Trigger                                   | Writer          |
| --------- | ------------ | ----------------------------------------- | --------------- |
| -         | `queued`     | Job created                               | Intake Consumer |
| `queued`  | `running`    | Workflow starts                           | Phase worker    |
| `running` | `blocked`    | Retryable error (lock, throttle)          | Phase worker    |
| `blocked` | `running`    | Retry succeeds                            | Phase worker    |
| `running` | `partial`    | Phase 3 success, some pages failed        | MarkJobStatus   |
| `running` | `succeeded`  | Phase 3 success, all pages succeeded      | MarkJobStatus   |
| `running` | `failed`     | Terminal error in any phase               | MarkJobStatus   |
| `running` | `superseded` | Newer version completed during processing | MarkJobStatus   |
| `blocked` | `failed`     | Max retries exceeded                      | MarkJobStatus   |

**Invariant**: Only `MarkJobStatus` Lambda (invoked by Step Functions terminal ASL states) may write terminal status values.

---

## Phase Values

### Values

| Phase      | Description                         |
| ---------- | ----------------------------------- |
| `phase0`   | Normalizing source (PDF → Markdown) |
| `phase1`   | Extracting claims from chunks       |
| `phase2`   | Synthesizing wiki pages             |
| `phase3`   | Updating wiki (serialized)          |
| `complete` | All phases finished                 |

### Phase Progression

```
phase0 → phase1 → phase2 → phase3 → complete
```

Each phase updates `currentPhase` when it starts. Terminal states set `currentPhase = complete`.

---

## Wiki Status

### Values

| Status     | Description                                  |
| ---------- | -------------------------------------------- |
| `empty`    | No sources processed yet                     |
| `ready`    | All known sources processed, wiki up to date |
| `updating` | One or more sources currently processing     |
| `degraded` | Some sources failed, wiki partially complete |

### Derivation Logic

```python
def derive_wiki_status(folder_id: str) -> str:
    jobs = query_jobs_by_folder(folder_id)

    if not jobs:
        return "empty"

    active = [j for j in jobs if j.status in ("queued", "running", "blocked")]
    failed = [j for j in jobs if j.status == "failed"]

    if active:
        return "updating"
    elif failed:
        return "degraded"
    else:
        return "ready"
```

---

## User-Facing Mapping

Recommended Zarya UI labels:

| Internal Status | UI Label             | UI State    |
| --------------- | -------------------- | ----------- |
| `queued`        | "Waiting"            | Pending     |
| `running`       | Phase-specific text  | In Progress |
| `blocked`       | "Retrying"           | In Progress |
| `partial`       | "Partially complete" | Warning     |
| `succeeded`     | "Complete"           | Success     |
| `failed`        | "Failed"             | Error       |
| `superseded`    | "Replaced"           | Info        |

### Phase-Specific Labels

| Current Phase | UI Label              |
| ------------- | --------------------- |
| `phase0`      | "Extracting text"     |
| `phase1`      | "Analyzing content"   |
| `phase2`      | "Building wiki pages" |
| `phase3`      | "Updating wiki"       |
| `complete`    | "Complete"            |

---

## Progress Reporting

### Job Progress Response

```typescript
interface JobProgress {
  phase0: PhaseProgress;
  phase1: Phase1Progress;
  phase2: Phase2Progress;
  phase3: PhaseProgress;
}

interface PhaseProgress {
  status: "pending" | "running" | "complete" | "failed";
  durationMs?: number;
}

interface Phase1Progress extends PhaseProgress {
  chunksTotal: number;
  chunksComplete: number;
}

interface Phase2Progress extends PhaseProgress {
  pagesTotal: number;
  pagesComplete: number;
}
```

### Example Response

```json
{
  "phase0": { "status": "complete", "durationMs": 45000 },
  "phase1": { "status": "complete", "chunksTotal": 8, "chunksComplete": 8 },
  "phase2": { "status": "running", "pagesTotal": 5, "pagesComplete": 2 },
  "phase3": { "status": "pending" }
}
```

---

## Changelog

| Date       | Change                          |
| ---------- | ------------------------------- |
| 2026-05-13 | Initial status model extraction |
