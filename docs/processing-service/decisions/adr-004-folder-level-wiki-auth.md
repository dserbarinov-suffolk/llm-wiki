# ADR-004: Folder-Level Wiki Authorization

## Status

Accepted

## Context

The processing service exposes wiki content via API. Authorization must:

- Integrate with Zarya's existing permission model
- Support folder-level access control
- Distinguish between summary and content access
- Avoid leaking derived content from restricted sources

Options considered:

1. **Zarya delegation** - Call Zarya API for authorization
2. **JWT claims** - Derive permissions from token claims
3. **Page-level ACLs** - Fine-grained per-page permissions
4. **Copy Zarya model** - Replicate permission data locally

## Decision

Delegate authorization to Zarya via API with two access levels:

- **viewWikiSummary**: Status, counts, page catalog (no body)
- **readWikiContent**: Full wiki content including page bodies

## Rationale

### Why Zarya Delegation

- **Single source of truth**: Zarya owns folder permissions
- **Real-time**: Permission changes take effect immediately (with short cache)
- **No sync**: Don't need to replicate or sync permission data
- **Consistency**: Same rules as Zarya file explorer

### Why Two Access Levels

- **viewWikiSummary**: Matches Zarya folder visibility in explorer
- **readWikiContent**: Conservative rule requiring access to ALL sources

The conservative readWikiContent rule prevents leaking derived claims from a source the user cannot read. A user who can see the folder but can't read a specific file shouldn't see claims derived from that file.

### Trade-offs Accepted

- **Network call**: Every request calls Zarya auth API (mitigated by caching)
- **No partial access**: User sees all or nothing per folder
- **Title exposure**: Page titles visible with viewWikiSummary (accepted—titles not considered sensitive)

## Alternatives Rejected

### JWT Claims

- Claims become stale (folder sharing changes)
- Token contains too much permission data
- Zarya would need to issue wiki-aware tokens

### Page-Level ACLs

- Complex to implement and maintain
- Performance overhead for per-page checks
- Product doesn't require this granularity in V1

### Local Permission Copy

- Sync complexity and staleness
- Data duplication
- Consistency bugs

## Consequences

- API handlers must call Zarya auth API
- Authorization decisions cached with short TTL
- readWikiContent requires access to all live sources
- Page-level filtering is future work if needed
- Must monitor Zarya auth API latency and errors

## Related

- [Auth Contract](../interfaces/auth-contract.md) - Authorization API contract
- [Feature 07](../features/07-read-api-and-auth.md) - API authorization implementation

## Date

2026-05-13
