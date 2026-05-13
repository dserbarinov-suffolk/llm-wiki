# ADR-002: Step Functions for Workflow Orchestration

## Status

Accepted

## Context

The processing service needs to orchestrate multi-phase workflows (Phase 0 → 1 → 2 → 3) with:

- Parallel processing within phases (Map states)
- Error handling and retries
- Long-running async operations (Phase 3 wiki maintenance)
- Idempotent execution

Options considered:

1. **AWS Step Functions** - Managed state machine
2. **Temporal** - Open-source workflow engine
3. **Custom orchestration** - Lambda chains with SQS
4. **AWS EventBridge Pipes** - Event-driven orchestration

## Decision

Use **AWS Step Functions Standard Workflows** for workflow orchestration.

## Rationale

### Why Step Functions

- **Visual workflow**: State machine definition is self-documenting
- **Built-in retries**: Configurable retry with exponential backoff
- **Map state**: Native parallel processing with concurrency control
- **waitForTaskToken**: Async callback pattern for Phase 3 serialization
- **AWS integration**: Native Lambda, SQS, DynamoDB integrations
- **Audit trail**: Execution history preserved for debugging
- **No infrastructure**: Serverless, pay per state transition

### Why Standard (not Express)

- **Duration**: Workflows can run hours (60 PDFs = ~4 hours)
- **waitForTaskToken**: Required for Phase 3 FIFO queue pattern
- **Execution history**: Need full audit trail for debugging

### Trade-offs Accepted

- **Cost**: $25 per million state transitions (acceptable at our scale)
- **Cold start**: First state has ~100ms overhead
- **ASL complexity**: Amazon States Language has learning curve

## Alternatives Rejected

### Temporal

- Additional infrastructure to operate
- Overkill for our workflow complexity
- Team unfamiliar with Temporal

### Custom Lambda chains

- No built-in retry logic
- Complex error handling
- No visual debugging
- Reinventing the wheel

### EventBridge Pipes

- Cannot set execution name for idempotency
- Limited orchestration capabilities
- Better for simple event routing

## Consequences

- Workflow definition lives in ASL JSON files
- CI must validate ASL before deployment
- State transitions are billable events
- Must design for Map state partial failure handling
- Execution name = fileVersionId provides idempotency

## Related

- [System Contract](../system-contract.md) - Orchestration authority invariants
- [Feature 01](../features/01-intake-and-job-tracking.md) - Workflow start idempotency

## Date

2026-05-13
