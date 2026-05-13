# Authorization Contract

> Centralized authorization model for the processing service.
> All API endpoints must conform to this contract.
>
> **Integration Contract**: [SCALABLE-FILE-PROCESSING-HANDOFF.md](../../SCALABLE-FILE-PROCESSING-HANDOFF.md)
> defines the API endpoint shapes and authorization responsibilities.

---

## Authorization Model

The processing service delegates authorization to Zarya. It does not maintain its own user model or access control lists.

### Flow

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Client    │──JWT───▶│  API GW     │──JWT───▶│  Lambda     │
│  (Zarya FE) │         │  Authorizer │         │  Handler    │
└─────────────┘         └─────────────┘         └─────────────┘
                               │                       │
                        Validate JWT            Call Zarya Auth
                        (authentication)        (authorization)
                               │                       │
                               ▼                       ▼
                        ┌─────────────┐         ┌─────────────┐
                        │  Cognito    │         │ Zarya API   │
                        │  User Pool  │         │ Auth Check  │
                        └─────────────┘         └─────────────┘
```

### Authentication vs Authorization

| Concern        | Who Handles      | What It Does                           |
| -------------- | ---------------- | -------------------------------------- |
| Authentication | API GW + Cognito | Validates JWT signature and expiration |
| Authorization  | Zarya API        | Determines folder access rights        |

**Important**: Do not derive folder permissions from local authorizer claims. Zarya is authoritative for access decisions.

---

## Access Types

| Access            | Allows                                                       | Decision Rule                                             |
| ----------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| `viewWikiSummary` | Wiki status, counts, ingest statuses, page catalog (no body) | User can see folder in Zarya explorer                     |
| `readWikiContent` | Wiki page Markdown, index, log, contradictions, source pages | User can read all current live source files in the folder |

### Security Invariant (MVP)

`readWikiContent` requires the user to be able to read **all current live source files** in the folder (not just those "represented in the wiki").

**Rationale**: This conservative rule avoids leaking derived claims from a source the user cannot read.

**Future**: Page-level source filtering may be added later if product requirements demand partial-folder access.

---

## Endpoint Mapping

| Endpoint                               | Required Access                                                |
| -------------------------------------- | -------------------------------------------------------------- |
| `GET /wikis/{folderId}`                | `viewWikiSummary`                                              |
| `GET /processing/jobs?folderId={id}`   | `viewWikiSummary`                                              |
| `GET /processing/jobs/{fileVersionId}` | `viewWikiSummary` (resolve folderId from ProcessingJobs first) |
| `GET /wikis/{folderId}/index`          | `readWikiContent`                                              |
| `GET /wikis/{folderId}/log`            | `readWikiContent`                                              |
| `GET /wikis/{folderId}/pages`          | `viewWikiSummary` (metadata only: titles, types, timestamps)   |
| `GET /wikis/{folderId}/pages/{pageId}` | `readWikiContent`                                              |
| `GET /wikis/{folderId}/contradictions` | `readWikiContent`                                              |

### Decision: Page Title Sensitivity

`viewWikiSummary` exposes page titles via `/wikis/{folderId}/pages`. This is acceptable
because page titles are not considered sensitive for this product. If this assumption
changes, the endpoint can be updated to require `readWikiContent` instead.

---

## Zarya Auth API

### Request

```typescript
interface WikiAuthorizationRequest {
  checks: Array<{
    folderId: string;
    access: "viewWikiSummary" | "readWikiContent";
  }>;
}
```

### Response

```typescript
interface WikiAuthorizationResponse {
  results: Array<{
    folderId: string;
    access: "viewWikiSummary" | "readWikiContent";
    allowed: boolean;
    reason?: string; // Present if denied
    decisionTtlSeconds: number;
  }>;
}
```

### Example Call

```typescript
const response = await zaryaAuthClient.checkWikiAccess(
  {
    checks: [{ folderId: "folder-123", access: "readWikiContent" }],
  },
  {
    headers: { Authorization: `Bearer ${userJwt}` },
  },
);

if (!response.results[0].allowed) {
  throw new ForbiddenError(response.results[0].reason);
}
```

---

## Caching

### Cache Key Format

```
sha256(jwt):folderId:access
```

**Note**: Hash the JWT to avoid trusting unvalidated claims in the cache key.

### Cache TTL

Use `decisionTtlSeconds` from the Zarya response. Keep TTL short because folder sharing/revocation should take effect quickly.

### Recommended TTL

| Environment | TTL  | Rationale                         |
| ----------- | ---- | --------------------------------- |
| dev         | 60s  | Fast iteration during testing     |
| stage/prod  | 300s | Balance performance and freshness |

---

## Error Handling

### Transport-Level Errors

| Situation                   | HTTP  | API Code           |
| --------------------------- | ----- | ------------------ |
| Missing/invalid JWT         | `401` | `UNAUTHENTICATED`  |
| Disabled current user       | `403` | `FORBIDDEN`        |
| Invalid request body        | `400` | `VALIDATION_ERROR` |
| Unexpected internal failure | `500` | `INTERNAL_ERROR`   |

### Authorization Failures

Authorization failures return HTTP `403` with a structured error:

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Access denied to folder folder-123",
    "details": {
      "folderId": "folder-123",
      "requiredAccess": "readWikiContent"
    }
  }
}
```

---

## Implementation Checklist

- [ ] API Gateway authorizer validates JWT signature and expiration
- [ ] Lambda handlers call Zarya auth API for folder access
- [ ] Authorization decisions are cached with appropriate TTL
- [ ] Denied requests return structured 403 errors
- [ ] Job status lookups resolve folderId before checking access
- [ ] No folder permissions derived from JWT claims

---

## Changelog

| Date       | Change                           |
| ---------- | -------------------------------- |
| 2026-05-13 | Initial auth contract extraction |
