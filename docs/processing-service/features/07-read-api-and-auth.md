# Feature 07: Read API and Authorization

> This feature inherits all invariants from [`../system-contract.md`](../system-contract.md).
>
> Relevant invariants:
>
> - `folderId` is the wiki aggregate key.
> - Derived data has the same sensitivity classification as source content.

---

## Goal

Expose read APIs for job status and wiki content, delegate authorization to Zarya, and enforce the summary/content permission distinction.

---

## Scope

### Included

- API Gateway HTTP API
- API authorizer (JWT validation)
- Job status endpoints
- Wiki summary endpoint
- Wiki content endpoints
- Zarya authorization integration
- Authorization decision caching

### Excluded

- Write endpoints (contradiction resolution deferred)
- Internal/admin endpoints

---

## Preconditions

- Features 00-05 deployed (jobs and wikis populated)
- Zarya wiki authorization API available
- Cognito User Pool shared with Zarya

---

## Postconditions

After deployment:

| Endpoint                               | Available | Authorization   |
| -------------------------------------- | --------- | --------------- |
| `GET /processing/jobs/{fileVersionId}` | Yes       | viewWikiSummary |
| `GET /processing/jobs?folderId={id}`   | Yes       | viewWikiSummary |
| `GET /wikis/{folderId}`                | Yes       | viewWikiSummary |
| `GET /wikis/{folderId}/index`          | Yes       | readWikiContent |
| `GET /wikis/{folderId}/log`            | Yes       | readWikiContent |
| `GET /wikis/{folderId}/pages`          | Yes       | viewWikiSummary |
| `GET /wikis/{folderId}/pages/{pageId}` | Yes       | readWikiContent |
| `GET /wikis/{folderId}/contradictions` | Yes       | readWikiContent |

---

## Interfaces Consumed

| Interface            | Location                                                           |
| -------------------- | ------------------------------------------------------------------ |
| ProcessingJobs table | [`../interfaces/data-models.md`](../interfaces/data-models.md)     |
| Wiki bucket          | [`../interfaces/s3-layout.md`](../interfaces/s3-layout.md)         |
| Zarya auth API       | [`../interfaces/auth-contract.md`](../interfaces/auth-contract.md) |

## Interfaces Produced

| Interface             | Location      |
| --------------------- | ------------- |
| ProcessingJobResponse | Defined below |
| WikiSummaryResponse   | Defined below |
| WikiPageResponse      | Defined below |

---

## Functional Requirements

### Authentication

1. API Gateway Lambda authorizer validates JWT.
2. JWT must be from shared Cognito User Pool.
3. Extract user ID from JWT claims.
4. Attach user context to request.

### Authorization

1. For each request, call Zarya auth API with folderId and required access.
2. Cache authorization decisions (key: `sha256(jwt):folderId:access`).
3. Cache TTL from Zarya response (typically 60-300s).
4. On authorization failure, return 403 with structured error.

### Job Status Endpoints

**GET /processing/jobs/{fileVersionId}**

1. Load job from ProcessingJobs table.
2. Resolve folderId from job record.
3. Check viewWikiSummary access for folderId.
4. Return ProcessingJobResponse.

**GET /processing/jobs?folderId={id}**

1. Check viewWikiSummary access for folderId.
2. Query ProcessingJobs by folderId GSI.
3. Return array of ProcessingJobResponse.

### Wiki Endpoints

**GET /wikis/{folderId}**

1. Check viewWikiSummary access.
2. List wiki contents from S3.
3. Compute stats (source count, page counts, etc.).
4. Return WikiSummaryResponse.

**GET /wikis/{folderId}/pages**

1. Check viewWikiSummary access.
2. List pages from wiki (metadata only).
3. Return array of page metadata (title, type, path, updatedAt).

**GET /wikis/{folderId}/pages/{pageId}**

1. Check readWikiContent access.
2. Load page content from wiki.
3. Return WikiPageResponse with full Markdown body.

**GET /wikis/{folderId}/index**

1. Check readWikiContent access.
2. Load index.md from wiki.
3. Return raw Markdown.

**GET /wikis/{folderId}/log**

1. Check readWikiContent access.
2. Load log.md from wiki.
3. Return raw Markdown (or paginated entries).

**GET /wikis/{folderId}/contradictions**

1. Check readWikiContent access.
2. Load contradictions from wiki.
3. Return array of ContradictionResponse.

---

## Response Schemas

### ProcessingJobResponse

```typescript
interface ProcessingJobResponse {
  fileVersionId: string;
  fileId: string;
  folderId: string;
  status:
    | "queued"
    | "running"
    | "blocked"
    | "partial"
    | "succeeded"
    | "failed"
    | "superseded";
  currentPhase: string;
  progress: {
    phase0: PhaseProgress;
    phase1: Phase1Progress;
    phase2: Phase2Progress;
    phase3: PhaseProgress;
  };
  createdAt: string;
  updatedAt: string;
  completedAt?: string;
  error?: { code: string; message: string };
}
```

### WikiSummaryResponse

```typescript
interface WikiSummaryResponse {
  folderId: string;
  wikiStatus: "empty" | "ready" | "updating" | "degraded";
  latestUpdatedAt?: string;
  stats: {
    sourceCount: number;
    conceptCount: number;
    entityCount: number;
    procedureCount: number;
    referenceCount: number;
    contradictionCount: number;
  };
  activeFileVersionIds: string[];
  failedFileVersionIds: string[];
  recentIngests: Array<{
    fileVersionId: string;
    fileId: string;
    status: "succeeded" | "failed" | "partial" | "superseded";
    completedAt: string;
    pagesCreated: number;
    pagesUpdated: number;
  }>;
}
```

### WikiPageResponse

```typescript
interface WikiPageResponse {
  pageId: string;
  path: string;
  title: string;
  category: "source" | "concept" | "entity" | "procedure" | "reference";
  status: "draft" | "reviewed" | "stable" | "archived";
  body: string; // Full Markdown content
  sources: Array<{ fileId: string; fileVersionId: string }>;
  createdAt: string;
  updatedAt: string;
}
```

---

## Failure Handling

| Failure              | HTTP | Response               |
| -------------------- | ---- | ---------------------- |
| Missing/invalid JWT  | 401  | UNAUTHENTICATED        |
| Authorization denied | 403  | FORBIDDEN              |
| Job not found        | 404  | NOT_FOUND              |
| Wiki not found       | 404  | NOT_FOUND              |
| Page not found       | 404  | NOT_FOUND              |
| Zarya auth API error | 502  | UPSTREAM_ERROR (retry) |
| Internal error       | 500  | INTERNAL_ERROR         |

---

## Security Requirements

1. All endpoints require valid JWT.
2. No endpoint exposes content without authorization.
3. Authorization decisions cached with short TTL.
4. No user PII in logs beyond user ID.
5. Rate limiting on API Gateway (per user).

---

## Observability Requirements

| Metric               | Dimension        | Value                  |
| -------------------- | ---------------- | ---------------------- |
| `api.requests`       | endpoint, status | Request count          |
| `api.latency_ms`     | endpoint         | Response time          |
| `api.auth.cache_hit` | environment      | Cache hit rate         |
| `api.auth.denied`    | environment      | Authorization failures |

| Log Field    | Required | Example                   |
| ------------ | -------- | ------------------------- |
| `userId`     | Yes      | `"user-123"`              |
| `endpoint`   | Yes      | `"GET /wikis/{folderId}"` |
| `folderId`   | If any   | `"folder-456"`            |
| `statusCode` | Yes      | `200`                     |
| `latencyMs`  | Yes      | `45`                      |

---

## Verification Plan

1. **Authenticated access**
   - Request with valid JWT → succeeds
   - Request without JWT → 401
   - Request with expired JWT → 401

2. **Authorization**
   - User with viewWikiSummary → can access summary
   - User with viewWikiSummary → cannot access page content
   - User with readWikiContent → can access page content
   - User with no access → 403

3. **Cache behavior**
   - First request calls Zarya auth
   - Second request uses cache
   - Cache expires, next request calls Zarya again

4. **Job endpoints**
   - Get job by fileVersionId
   - List jobs by folderId
   - Verify authorization uses resolved folderId

5. **Wiki endpoints**
   - Get wiki summary
   - List pages (metadata only)
   - Get page content
   - Get index
   - Get log
   - Get contradictions

---

## Rollout / Rollback

### Rollout

1. Deploy API Gateway and Lambda handlers.
2. Configure Cognito authorizer.
3. Test with authorized users.
4. Route traffic from Zarya frontend.

### Rollback

1. Update Zarya frontend to disable wiki features.
2. API remains deployed but unused.
3. Rollback API Gateway if needed.

---

## Open Questions

None.

---

## Changelog

| Date       | Change               |
| ---------- | -------------------- |
| 2026-05-13 | Initial feature spec |
