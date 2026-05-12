# Internal Wiki Authorization API Technical Design

## Purpose

This document is the implementation source of truth for the Zarya API endpoint that lets the
LLM-Wiki processing service authorize user-facing wiki reads without duplicating Zarya's permission
logic.

The processing service owns generated wiki artifacts. Zarya owns resource identity, folder/file
visibility, user identity, share grants, and authorization rules. The processing service must not
interpret Zarya grants directly.

## Goals

- Provide a narrow internal API for checking whether a Zarya user can view a folder wiki summary or
  read folder wiki content.
- Keep all resource authorization decisions in Zarya.
- Reuse Zarya's existing Cognito principal resolution, DynamoDB metadata, share visibility, and
  Verified Permissions integration.
- Return small, cacheable decisions that are safe for the processing service to enforce.
- Make the contract explicit enough for the processing service to implement against before the UI
  details are final.

## Non-Goals

- This API does not expose general-purpose arbitrary authorization checks.
- This API does not return share grants, Cedar policies, or raw permission internals.
- This API does not let the processing service mutate Zarya folders, files, shares, or wiki data.
- This API does not make folder wikis independently shareable. Wiki visibility follows the Zarya
  folder/file model until product requirements say otherwise.

## Existing Permission Model

Zarya currently models folder and file actions in `packages/core`:

Folder actions:

- `TraversePath`
- `ListChildren`
- `CreateFolder`
- `UploadIntoFolder`
- `DeleteFolder`
- `ShareFolder`

File actions:

- `ViewMetadata`
- `ReadContent`
- `ReplaceFile`
- `DeleteFile`
- `ShareFile`

Folder grants can include descendant file actions such as `ReadContent`. That means wiki content
authorization should not be implemented as a simple folder action check. A folder wiki is generated
from source files in that folder, so content reads require access to the folder's source content,
not merely visibility of the folder name.

## API Surface

Add one internal route:

```http
POST /internal/wiki-authorization/check
Authorization: Bearer <Zarya Cognito access token>
X-Internal-Service: processing-service
X-Request-Id: <optional request id>
Content-Type: application/json
```

The route uses the normal Zarya API envelope:

```ts
type ApiEnvelope<T> =
  | { ok: true; data: T; requestId: string }
  | {
      ok: false;
      error: { code: ApiErrorCode; message: string; details?: unknown };
      requestId: string;
    };
```

### Request DTO

```ts
type WikiAuthorizationAccess = "viewWikiSummary" | "readWikiContent";

type WikiAuthorizationCheckRequest = {
  checks: Array<{
    checkId: string;
    folderId: string;
    access: WikiAuthorizationAccess;
  }>;
};
```

Constraints:

- `checks` must contain at least one item.
- Initial maximum batch size is `30`, matching the current Verified Permissions batch size.
- `checkId` is caller-defined and echoed in the response.
- `folderId` is a Zarya folder id, not a wiki-specific id.

### Response DTO

```ts
type WikiAuthorizationDecisionReason =
  | "ALLOWED"
  | "DENIED"
  | "FOLDER_NOT_FOUND"
  | "FOLDER_DELETED"
  | "USER_DISABLED";

type WikiAuthorizationCheckResponse = {
  checks: Array<{
    checkId: string;
    folderId: string;
    access: WikiAuthorizationAccess;
    allowed: boolean;
    reason: WikiAuthorizationDecisionReason;
    decisionTtlSeconds: number;
  }>;
};
```

The processing service may cache allowed and denied decisions for at most `decisionTtlSeconds`.
Initial value: use the same configured authorization decision cache TTL as Zarya's Verified
Permissions authorizer.

## Access Semantics

### `viewWikiSummary`

Allows the processing service to return folder-level wiki shell data:

- wiki status
- counts
- recent ingest statuses
- contradiction count
- page catalog entries without page body excerpts

Decision rule:

```text
allowed when the user can see the folder in the Zarya explorer.
```

Implementation options, in order of preference:

1. Reuse the explorer visibility service to determine whether `getFolderChildrenForUser(userId,
folderId)` would not throw `ExplorerFolderNotVisibleError`.
2. Or implement an equivalent pure visibility check that handles owner visibility, direct folder
   grants, inherited folder grants, and partial ancestor visibility.

Do not require `ReadContent` for this level. A user who can see a folder can see that the wiki
exists and whether it is processing, but cannot necessarily read generated wiki bodies.

### `readWikiContent`

Allows the processing service to return generated Markdown page bodies or source-derived details:

- `index.md` body
- `log.md` entries if they reveal source-derived content
- wiki page Markdown
- contradiction page Markdown
- source summary pages
- answer pages that cite source content

Decision rule:

```text
allowed when the user can read content for every current, live source file represented in the wiki,
or has a folder grant/ownership path that grants descendant file ReadContent for the represented
source files.
```

Initial implementation:

```text
allowed when the user can read content for all current, live source files in that folder.
```

The conservative rule avoids leaking derived claims from a source file the user cannot read. If this
is too restrictive for partially shared folders, add page-level source filtering later. Until page
filtering exists, `readWikiContent` must be conservative.

Recommended initial behavior:

- For owned folders: allow.
- For shared folders: allow only when the effective folder grant includes descendant `ReadContent`
  for files in that folder, or every live source file represented in the wiki is independently
  readable by the requester.
- For partial ancestor visibility without content permission: deny.

## Processing Service Usage

Use `viewWikiSummary` for:

```http
GET /wikis/{folderId}
GET /processing/jobs?folderId={folderId}
GET /wikis/{folderId}/pages
```

`GET /wikis/{folderId}/pages` is summary-level only when it returns metadata such as page ids,
titles, categories, update timestamps, and source counts without Markdown bodies, excerpts,
evidence text, contradiction details, or other source-derived content.

Use `readWikiContent` for:

```http
GET /wikis/{folderId}/index
GET /wikis/{folderId}/log
GET /wikis/{folderId}/pages/{pageId}
GET /wikis/{folderId}/contradictions
```

`log.md` stays behind `readWikiContent` for the MVP. It may include file names, extracted topics,
contradiction details, or other source-derived information. Do not split it into a sanitized summary
view unless product needs a separate ingest-events surface later.

Use job-local lookup plus `viewWikiSummary` for:

```http
GET /processing/jobs/{fileVersionId}
```

The processing service should resolve `fileVersionId -> folderId` from its own `ProcessingJobs`
table, then authorize the folder with Zarya.

## Server Implementation Plan

Add a new server feature:

```text
packages/server/src/features/internal-authorization/
  domain/
    wiki-authorization-service.ts
    wiki-authorization-service.test.ts
  routes/
    wiki-authorization-router.ts
    wiki-authorization-router.test.ts
```

Add shared DTOs in `core`:

```text
packages/core/src/contracts/internal-authorization/wiki-authorization.ts
```

Export those DTOs from `packages/core/src/index.ts`.

### Route Responsibilities

The route should:

1. Parse and validate request body with the shared `core` schema.
2. Require an authenticated Cognito principal through existing authentication middleware.
3. Resolve the active Zarya user through the existing session user resolver.
4. Call `WikiAuthorizationService.checkWikiAuthorization(user.userId, request)`.
5. Validate the response with the shared `core` schema.
6. Return the normal Zarya API envelope.

The route must stay thin. It should not load folders, grants, files, or call Verified Permissions
directly.

### Service Responsibilities

The service should:

1. Load requested folders and reject missing/deleted folders with per-check reasons.
2. Evaluate `viewWikiSummary` using explorer visibility semantics.
3. Evaluate `readWikiContent` using conservative source-content semantics.
4. Return one decision per input check, preserving order.

Suggested dependency shape:

```ts
type WikiAuthorizationServiceDependencies = {
  visibility: {
    canViewFolder(
      userId: UserId,
      folderId: FolderId,
    ): Promise<FolderVisibilityDecision>;
  };
  contentAccess: {
    canReadFolderWikiContent(
      userId: UserId,
      folderId: FolderId,
    ): Promise<FolderContentDecision>;
  };
  config: {
    decisionTtlSeconds: number;
  };
};
```

Keep the visibility and content-access adapters behind interfaces so the first implementation can
reuse existing repositories/services and later be optimized without changing the route contract.

## Authorization Implementation Detail

### Visibility Adapter

The visibility adapter may initially wrap the explorer bootstrap service:

```ts
async function canViewFolder(
  userId: UserId,
  folderId: FolderId,
): Promise<FolderVisibilityDecision> {
  try {
    await explorerBootstrapService.getFolderChildrenForUser(userId, folderId);
    return { status: "visible" };
  } catch (error) {
    if (error instanceof ExplorerFolderNotVisibleError) {
      return { status: "notVisible" };
    }
    throw error;
  }
}
```

This is simple and correct, though it may load more data than needed. If it becomes hot, replace it
with a narrower visibility query that uses the same domain rules.

### Content Access Adapter

The content-access adapter needs to prevent leakage from generated wiki pages. Initial conservative
rule:

1. Load the folder.
2. If missing, return `FOLDER_NOT_FOUND`.
3. If deleted, return `FOLDER_DELETED`.
4. If `folder.ownerUserId === userId`, allow.
5. Load current live files under the folder that are represented in the wiki or, until wiki-source
   membership is available to Zarya, all current live direct child files.
6. For each file, authorize `ReadContent` on that file for the requester.
7. Allow only if every represented file is readable.

This can use the existing `BatchAuthorizer` and Cedar entity construction patterns from explorer
capability projection. If the folder has no represented files yet, deny `readWikiContent` unless the
user owns the folder.

Future improvement:

- Accept an optional `sourceFileIds` or `sourceFileVersionIds` list from the processing service for
  page-level authorization. Zarya can then require `ReadContent` only for sources cited by the page
  being returned.

## Error Handling

Transport-level failures:

| Situation                           | HTTP  | API code           |
| ----------------------------------- | ----- | ------------------ |
| Missing/invalid JWT                 | `401` | `UNAUTHENTICATED`  |
| Disabled current user               | `403` | `FORBIDDEN`        |
| Invalid request body                | `400` | `VALIDATION_ERROR` |
| Batch too large                     | `400` | `VALIDATION_ERROR` |
| Unexpected repository/authz failure | `500` | `INTERNAL_ERROR`   |

Per-check authorization failures must be represented as decisions in a successful `200` response,
not as route-level failures.

Example:

```json
{
  "ok": true,
  "data": {
    "checks": [
      {
        "checkId": "wiki-folder-1",
        "folderId": "folder-1",
        "access": "readWikiContent",
        "allowed": false,
        "reason": "DENIED",
        "decisionTtlSeconds": 60
      }
    ]
  },
  "requestId": "request-1"
}
```

## Security

- This route is internal but still requires a valid Zarya user JWT.
- The processing service must forward the end user's JWT when authorizing user-facing responses.
- Do not authorize user-facing wiki reads with only the processing service IAM role.
- The processing service may use IAM/service auth only for machine-to-machine health checks or
  non-user operations that do not return user content.
- Do not log JWTs, email addresses, wiki page bodies, source document text, or raw Cedar entities.
- Log `requestId`, `userId`, `folderId`, `access`, `allowed`, and `reason`.

## Performance and Caching

The route supports batching to reduce chatter from the processing service.

Recommended cache behavior:

- Processing service may cache decisions for `decisionTtlSeconds`.
- Use cache key `{userId}:{folderId}:{access}`.
- Default TTL should match Zarya's authorization decision cache TTL.
- Keep TTL short because folder sharing and revocation should take effect quickly.

Expected call patterns:

- Wiki summary route: one `viewWikiSummary` check.
- Wiki page route: one `readWikiContent` check.
- Page catalog route: one `viewWikiSummary` check when the catalog is metadata-only; require
  `readWikiContent` if it includes page bodies, excerpts, evidence text, or source-derived details.

## Testing Plan

### Core Contract Tests

Add tests for:

- valid request/response DTOs
- invalid access value
- empty checks array
- batch over maximum size
- missing `folderId`

### Service Unit Tests

Add tests for:

- owner can view summary and read content
- user with folder visibility but no content permission can view summary but cannot read content
- user with folder grant including descendant `ReadContent` can read content
- partial ancestor visibility allows summary only when explorer visibility says visible
- missing folder returns `FOLDER_NOT_FOUND`
- deleted folder returns `FOLDER_DELETED`
- decisions preserve input order

### Route Tests

Add tests for:

- authenticated request returns API envelope
- unauthenticated request returns `401`
- disabled user returns `403`
- invalid body returns `400`
- per-check denial returns `200` with `allowed: false`

### Integration/Fake Boundary Tests

Prefer fakes at repository and authorizer boundaries. Do not mock Verified Permissions directly in
service tests unless testing the adapter.

## IaC and Operations

No new AWS resources are required for the first implementation if the route is added to the existing
API Lambda.

The processing service must be allowed to call this API endpoint from its API runtime path. If API
Gateway resource policies or CORS are later tightened, include the processing service origin or
service role as appropriate. User-facing calls should still be authorized by forwarded user JWTs,
not by service role alone.

Operational metrics to add:

- `InternalWikiAuthorizationChecks`
- `InternalWikiAuthorizationDenied`
- `InternalWikiAuthorizationErrors`
- latency for the route

Suggested dimensions:

- `Environment`
- `Access`
- `Reason`
