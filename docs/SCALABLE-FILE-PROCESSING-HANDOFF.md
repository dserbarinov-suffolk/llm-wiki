# File Processing Integration Handoff

## Purpose

This document is the integration contract between Zarya's file-manager application and a separate
document processing service. Zarya owns upload authorization, S3 object promotion, canonical file
metadata, and user-facing file identity. The processing service consumes committed file-version
events, reads committed objects, and maintains one LLM Wiki per Zarya folder.

The initial target workload is a user uploading about 60 PDFs at once. Markdown files are also in
scope. Processing is expected to run in four phases:

| Phase | Work                                   | Parallelism                | Primary constraint |
| ----- | -------------------------------------- | -------------------------- | ------------------ |
| 0     | PDF or Markdown to normalized Markdown | per source document        | CPU and memory     |
| 1     | Extract claims                         | per slug                   | LLM rate limits    |
| 2     | Synthesize                             | per slug                   | LLM rate limits    |
| 3     | Maintain wiki                          | serialized per folder/wiki | shared wiki writes |

The consuming application implements the LLM Wiki pattern: raw sources are immutable, the wiki is a
persistent and compounding directory of LLM-generated Markdown pages, and a schema document tells the
LLM how to maintain the wiki. In Zarya terms, each folder is one wiki. Uploading or replacing a
supported file in a folder should incrementally update that folder's wiki instead of forcing the
user to re-query raw documents from scratch.

## Source System Contract

### Canonical Trigger

The processing service should treat a committed `FileVersion` row as the business event that a new
processable object exists.

Do not use S3 object-created events as the primary trigger. Zarya uploads bytes to
`pending-uploads/`, copies valid uploads to `objects/`, and only then commits metadata. A committed
S3 object without a `FileVersion` row is an incomplete upload-promotion state and is not
user-visible.

Recommended event source:

```text
DynamoDB stream on zarya-<env>-file-versions
stream view: NEW_IMAGE
consumer filter: INSERT events
```

Current implementation note: the checked-in foundation template defines the `FileVersionsTable`
without a DynamoDB stream. Enabling this trigger is a required foundation-stack change, not an
already-deployed table capability.

`FileVersion` rows are immutable once committed. The processor should ignore `MODIFY` and `REMOVE`
events unless a future product change explicitly adds processing behavior for deletion or retention.

### Required Zarya Tables

The processor may read these Zarya-owned tables:

| Table                       | Purpose                                                            | Access                                               |
| --------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------- |
| `zarya-<env>-file-versions` | immutable committed blob metadata                                  | stream read, `GetItem`, optional `Query` on `byFile` |
| `zarya-<env>-files`         | stable file identity, display name, owner, current version pointer | `GetItem` by `fileId`                                |

The processor must not write to Zarya-owned tables. Processing status, intermediate results, and
wiki outputs belong in processor-owned tables and S3 prefixes or buckets.

The processor must resolve `File.parentFolderId` before starting wiki work. A supported top-level
file with `parentFolderId === null` needs an explicit product decision before processing because
there is no concrete folder/wiki target.

### FileVersion Item Shape

The stream `NewImage` and `GetItem` result contain:

| Attribute                    | Type           | Meaning                                              |
| ---------------------------- | -------------- | ---------------------------------------------------- |
| `fileVersionId`              | string         | stable idempotency key for this uploaded version     |
| `fileId`                     | string         | stable user-facing file identity                     |
| `versionNumber`              | number         | version sequence for `fileId`                        |
| `objectKey`                  | string         | committed S3 key for the bytes                       |
| `sizeBytes`                  | number         | committed object size                                |
| `contentType`                | string         | committed object content type from upload completion |
| `etagOrChecksum`             | string         | S3 ETag or checksum captured during commit           |
| `createdByUserId`            | string         | user that initiated the upload                       |
| `createdFromUploadSessionId` | string         | upload session that produced this version            |
| `isDeleted`                  | boolean        | deletion marker; expected `false` on insert          |
| `deletedAt`                  | string or null | deletion timestamp; expected `null` on insert        |
| `createdAt`                  | string         | ISO timestamp for metadata commit                    |

The committed object key currently follows:

```text
objects/{fileId}/{fileVersionId}
```

The processor should validate that `objectKey` starts with `objects/` before reading from S3.

### File Item Shape

Load the `File` row only when processing needs file-level context such as display name, owner, or
current-version checks.

Relevant fields:

| Attribute          | Type           | Meaning                               |
| ------------------ | -------------- | ------------------------------------- |
| `fileId`           | string         | stable user-facing file identity      |
| `parentFolderId`   | string or null | canonical folder parent               |
| `name`             | string         | display name; not unique              |
| `ownerUserId`      | string         | owner                                 |
| `currentVersionId` | string         | current visible version for this file |
| `isDeleted`        | boolean        | file tombstone marker                 |
| `updatedAt`        | string         | latest file metadata update           |

The processor should only process the current user-visible version. It must load `File` and require
`File.isDeleted === false` and `File.currentVersionId === FileVersion.fileVersionId` before starting
expensive work. If a newer version arrives while older work is running, downstream wiki maintenance
must let the newer version win.

## Event Semantics

DynamoDB stream delivery is at least once. The processor must be idempotent.

Use `fileVersionId` as the natural idempotency key for a whole-file processing workflow. Use
`fileVersionId + phase + slug` as the idempotency key for slug-level tasks. Use `folderId` as the
serialization key for wiki-maintenance work.

Recommended event envelope produced by a dispatcher:

```json
{
  "eventType": "FileVersionCommitted",
  "schemaVersion": 1,
  "environment": "dev",
  "fileVersionId": "version-1",
  "fileId": "file-1",
  "versionNumber": 1,
  "objectKey": "objects/file-1/version-1",
  "sizeBytes": 2048,
  "contentType": "application/pdf",
  "etagOrChecksum": "checksum-1",
  "createdByUserId": "user-1",
  "createdAt": "2026-05-12T00:00:00.000Z"
}
```

The dispatcher should drop unsupported versions when:

- `contentType` is not `application/pdf`, `text/markdown`, or another explicitly approved Markdown
  media type
- `isDeleted` is true
- `objectKey` does not start with `objects/`
- required attributes are missing or fail validation

## Recommended Processing Architecture

Use the DynamoDB stream only to start controlled work. Do not let 60 simultaneous uploads become
unbounded Lambda or LLM concurrency.

Recommended AWS shape:

```text
FileVersion INSERT
-> stream dispatcher Lambda or EventBridge Pipe
-> processing intake queue
-> per-file workflow/orchestrator
-> phase-specific queues or Step Functions Map states
-> serialized folder/wiki maintenance
-> processor-owned wiki objects, result tables, and APIs
```

Step Functions is a good first orchestrator if the team wants AWS-native workflow visibility:

```text
Phase 0: convert one source document to normalized Markdown
Phase 1: map over Markdown slugs with MaxConcurrency
Phase 2: map over slugs with MaxConcurrency
Phase 3: update the folder wiki once all slug work succeeds
```

SQS-backed workers are a good fit when LLM rate limits need global control across many workflows:

| Queue                          | Unit of work                 | Concurrency control                                 |
| ------------------------------ | ---------------------------- | --------------------------------------------------- |
| `document-normalization-queue` | one file version             | Lambda reserved concurrency or ECS worker count     |
| `claim-extraction-queue`       | one slug or small slug batch | queue worker concurrency plus provider rate limiter |
| `synthesis-queue`              | one slug or small slug batch | queue worker concurrency plus provider rate limiter |
| `wiki-maintenance.fifo`        | one folder/wiki              | FIFO `MessageGroupId = folderId` or DynamoDB lock   |

For the 60-PDF case, expected behavior should be backlog-driven:

```text
60 file-version events arrive quickly
dispatcher creates or confirms 60 processing jobs
Phase 0 drains at CPU-safe concurrency
Phase 1 and Phase 2 drain at LLM-rate-safe concurrency
Phase 3 runs only after prerequisites are complete and the folder wiki lock is available
```

Initial concurrency should be configurable, not hard-coded. A conservative starting point:

| Phase | Suggested starting limit      | Notes                                                                  |
| ----- | ----------------------------- | ---------------------------------------------------------------------- |
| 0     | 5 to 10 PDFs                  | raise only after measuring CPU, memory, timeout, and ephemeral storage |
| 1     | 20 to 100 slug tasks globally | derive from LLM requests-per-minute and tokens-per-minute              |
| 2     | 20 to 100 slug tasks globally | separate limit from Phase 1 if model or prompt differs                 |
| 3     | 1 per folder/wiki             | use a lock or FIFO message group keyed by `folderId`                   |

If Phase 0 receives a Markdown source file, it should normalize and register the Markdown artifact
without running PDF conversion. If PDF conversion regularly exceeds Lambda timeout, memory, or
ephemeral storage limits, run conversion on ECS Fargate or AWS Batch while keeping the same
queue/workflow contract.

The processor should process only the latest version of each file. Older versions may have
completed historical jobs, but current derived outputs should be reconciled from the latest live
version so newer source documents win when they conflict with earlier source documents. For example,
if the latest building-code document changes a driveway-width requirement from `X` feet to `X + 1`
feet, folder-level outputs should converge on the newer requirement.

The processor should not silently overwrite incompatible claims between different latest source
files in the same folder when neither source clearly supersedes the other. It should preserve the
conflict in the wiki with citations to the competing sources, update relevant pages to mention the
contradiction, and expose the conflict for later review or linting.

## Large Source Size Guidance

Large PDFs should be handled as object references, never as queue, Lambda, or Step Functions payloads.
Queues and workflow state should carry `fileVersionId`, `fileId`, `folderId`, `objectKey`, size,
content type, checksum, and phase pointers only.

The proposed product maximum for automatically processed PDFs is 250 MB. Current Zarya upload
initiation validates `sizeBytes` as a non-negative number and records the client-supplied content
type, but it does not enforce a PDF-specific maximum before issuing a presigned upload URL. If the
250 MB processing bound becomes product policy, add that validation to upload initiation and keep a
matching processor-side `FileVersion.sizeBytes` check as defense in depth before Phase 0.

Current Zarya uploads use a single presigned S3 `PUT` to `pending-uploads/{uploadSessionId}/{fileVersionId}`,
then the server copies matching bytes to `objects/{fileId}/{fileVersionId}` during upload
completion. The bucket lifecycle aborts incomplete multipart uploads, but the application does not
currently expose a multipart upload initiation/completion API. Supporting PDFs at or above 100 MB as
a first-class workflow should add an explicit multipart upload path.

Under the proposed large-file processing policy, S3 multipart upload should be the default upload
strategy for PDFs at or above 100 MB. A 125 MB PDF is expected to be processable. A 250 MB PDF is
the upper supported bound and should use the large-file path: multipart upload, streaming reads where
possible, enough `/tmp` storage for conversion, and conservative Phase 0 concurrency until measured.

PDFs above 250 MB are out of scope for automatic processing under that proposed policy. The
preferred behavior is to reject them at upload initiation once the upload limit exists. If an
oversize object is already present because of an older client, manual import, or inconsistent
configuration, the dispatcher should ignore it or mark the processing job failed with an
unsupported-size error without attempting conversion.

Do not send 125 MB or 250 MB source bytes through API Gateway, SQS, Step Functions state, Lambda
invoke payloads, or DynamoDB items. Store bytes in S3 and pass pointers.

Recommended initial thresholds:

| Source size        | Upload path                                                         | Phase 0 path                                                    | Notes                                                 |
| ------------------ | ------------------------------------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------- |
| `< 100 MB`         | current single presigned `PUT` may be acceptable                    | Lambda or container worker                                      | still avoid loading whole files into memory           |
| `100 MB to 250 MB` | add an application multipart path before treating this as supported | Lambda only after measured; container worker remains acceptable | set explicit `/tmp`, memory, timeout, and concurrency |
| `> 250 MB`         | reject at upload initiation once the product limit is implemented   | do not process automatically                                    | report unsupported size if already committed          |

Phase 0 should record preflight metadata before conversion:

- source size
- page count when cheaply available
- encrypted/password-protected status
- detected text vs image-heavy pages when cheaply available
- estimated conversion strategy
- selected worker class

## LLM Wiki Contract

Each Zarya folder maps to one persistent wiki. The folder's uploaded files are the raw source layer,
and processor-owned Markdown objects are the generated wiki layer.

Required wiki files:

| File        | Purpose                                                                                       |
| ----------- | --------------------------------------------------------------------------------------------- |
| `schema.md` | wiki-maintenance instructions, page conventions, frontmatter conventions, and ingest workflow |
| `index.md`  | content-oriented catalog of wiki pages with links, summaries, categories, and source counts   |
| `log.md`    | append-only chronological record of ingests, wiki updates, queries, and lint passes           |

Recommended wiki directories:

| Directory         | Purpose                                                                       |
| ----------------- | ----------------------------------------------------------------------------- |
| `sources/`        | one summary page per latest source document version                           |
| `entities/`       | entity pages maintained across ingests                                        |
| `concepts/`       | topic and concept pages maintained across ingests                             |
| `comparisons/`    | cross-source comparisons and analyses                                         |
| `contradictions/` | unresolved or historically important conflicts between source claims          |
| `answers/`        | filed answers from future query workflows, if the consuming app supports them |

`index.md` is the primary navigation surface for both users and later LLM runs. `log.md` should use a
parseable heading prefix, for example:

```markdown
## [2026-05-12] ingest | fileVersionId=version-1 | Building Code 2026
```

The wiki is an accumulated artifact. Phase 3 should update multiple pages when a new source affects
existing knowledge: source summary, index, log, relevant entity pages, relevant concept pages,
comparisons, and contradiction pages as needed.

## Processor-Owned State

Create processor-owned persistence instead of adding processing fields to Zarya tables.

Suggested `ProcessingJobs` table:

| Attribute                   | Purpose                                                           |
| --------------------------- | ----------------------------------------------------------------- |
| `fileVersionId`             | partition key and workflow id                                     |
| `fileId`                    | source file id                                                    |
| `folderId`                  | wiki id and Phase 3 serialization key                             |
| `sourceObjectKey`           | committed Zarya object key                                        |
| `sourceEtagOrChecksum`      | source integrity marker                                           |
| `status`                    | `queued`, `running`, `blocked`, `succeeded`, `failed`, `canceled` |
| `currentPhase`              | `0`, `1`, `2`, `3`, or `complete`                                 |
| `attempts`                  | workflow-level attempts                                           |
| `createdAt`, `updatedAt`    | audit timestamps                                                  |
| `errorCode`, `errorMessage` | sanitized final or latest failure                                 |

Suggested `ProcessingTasks` table:

| Attribute                | Purpose                                                            |
| ------------------------ | ------------------------------------------------------------------ |
| `taskId`                 | stable id such as `{fileVersionId}#{phase}#{slug}`                 |
| `fileVersionId`          | parent workflow id                                                 |
| `phase`                  | `normalizeDocument`, `extractClaims`, `synthesize`, `maintainWiki` |
| `slug`                   | slug id for phase 1 and 2, `null` otherwise                        |
| `status`                 | task status                                                        |
| `attempts`               | retry count                                                        |
| `inputObjectKey`         | processor-owned or source input pointer                            |
| `outputObjectKey`        | processor-owned result pointer                                     |
| `createdAt`, `updatedAt` | audit timestamps                                                   |

Suggested processor S3 layout:

```text
processing/{fileVersionId}/phase-0/document.md
processing/{fileVersionId}/phase-1/{slug}.json
processing/{fileVersionId}/phase-2/{slug}.json
wikis/{folderId}/schema.md
wikis/{folderId}/index.md
wikis/{folderId}/log.md
wikis/{folderId}/sources/{fileId}/{fileVersionId}.md
wikis/{folderId}/entities/{pageSlug}.md
wikis/{folderId}/concepts/{pageSlug}.md
wikis/{folderId}/contradictions/{pageSlug}.md
```

Keep processor outputs separate from Zarya committed upload objects. If results should later appear
inside Zarya as user-visible files, add an explicit API/import flow rather than writing directly to
Zarya's metadata tables. The consuming application is expected to expose APIs that Zarya can call or
render from; the exact Zarya-visible data contract is still TBD.

Processor-owned Markdown artifacts, intermediate JSON, wiki outputs, and failed-attempt artifacts
should be retained indefinitely in S3 unless a future retention policy supersedes this contract.

## Zarya Integration API Starting Point

The consuming application should expose API endpoints for Zarya to inspect wiki state without
requiring Zarya to know the processor's internal S3 layout.

Suggested read APIs:

| Endpoint                               | Purpose                                                                        |
| -------------------------------------- | ------------------------------------------------------------------------------ |
| `GET /wikis/{folderId}`                | wiki summary, status, updated timestamp, page counts, and contradiction counts |
| `GET /wikis/{folderId}/index`          | rendered or raw `index.md`                                                     |
| `GET /wikis/{folderId}/log`            | recent `log.md` entries, with pagination                                       |
| `GET /wikis/{folderId}/pages`          | searchable/listable page catalog derived from `index.md` and metadata          |
| `GET /wikis/{folderId}/pages/{pageId}` | one wiki page as Markdown plus metadata                                        |
| `GET /processing/jobs/{fileVersionId}` | processing status for a source version                                         |

Suggested DTO fields for wiki pages:

| Field                  | Purpose                                                                   |
| ---------------------- | ------------------------------------------------------------------------- |
| `pageId`               | stable processor id for API routing                                       |
| `path`                 | wiki-relative Markdown path, such as `concepts/fire-rating.md`            |
| `title`                | display title                                                             |
| `category`             | `source`, `entity`, `concept`, `comparison`, `contradiction`, or `answer` |
| `markdown`             | page body when requesting a page                                          |
| `sourceFileVersionIds` | source versions cited by the page                                         |
| `outboundLinks`        | wiki page links referenced by the page                                    |
| `updatedAt`            | latest processor update timestamp                                         |

Exact UI behavior in Zarya is still TBD, but the API should preserve the LLM Wiki abstraction:
folders expose wikis, wikis expose Markdown pages, and pages cite source versions.

## Phase Contracts

### Phase 0: PDF or Markdown to Normalized Markdown

Input:

- `fileVersionId`
- Zarya file bucket
- committed `objectKey`
- `sizeBytes`
- `etagOrChecksum`

Output:

- Markdown object key
- extracted slug list
- conversion metadata such as page count, warnings, and parser version

The worker should verify the source object still exists before conversion or normalization. It may
compare the observed S3 ETag or checksum with `etagOrChecksum` when available.

### Phase 1: Extract Claims

Input:

- `fileVersionId`
- Markdown object key
- one slug or a small bounded slug batch

Output:

- per-slug claims object
- model name, prompt version, token usage, and sanitized failure details

The worker must be retry-safe. Re-running the same slug task should overwrite the same
processor-owned result key only after the replacement result is complete, or write a new attempt key
and atomically update the task pointer.

### Phase 2: Synthesize

Input:

- `fileVersionId`
- slug
- Phase 1 result pointer

Output:

- per-slug synthesis object
- model name, prompt version, token usage, and sanitized failure details

Use a separate concurrency/rate-limit budget from Phase 1 if Phase 2 uses different prompts,
models, or token volume.

### Phase 3: Maintain Wiki

Input:

- `folderId`
- latest `fileVersionId`
- normalized source Markdown pointer
- all successful Phase 2 result pointers for the source document
- current wiki page pointers for `schema.md`, `index.md`, `log.md`, and relevant existing pages

Output:

- updated wiki Markdown objects
- appended `log.md` entry
- updated `index.md`
- wiki maintenance status update

This phase must serialize writes to shared folder-level wiki files. Use one of:

- FIFO SQS with `MessageGroupId = folderId`
- DynamoDB conditional lock keyed by `folderId`
- a single reserved-concurrency wiki-maintenance worker per target class

Maintenance rules:

- create or update one source summary page for the latest `fileVersionId`
- update `index.md` on every successful ingest
- append to `log.md` on every successful ingest, failed maintenance attempt, query filing, or lint pass
- update existing entity and concept pages when the new source confirms, refines, or supersedes their claims
- create contradiction pages when current sources conflict without a clear precedence rule
- cite source summary pages or source ids from changed wiki pages
- avoid editing raw Zarya source objects

Future query and lint workflows should use the same wiki layout. Query answers that are worth
keeping can be filed under `answers/` and added to `index.md` and `log.md`. Lint passes should check
for stale claims, unresolved contradictions, missing cross-references, orphan pages, and important
concepts that lack pages.

## Failure Handling

Use retries for transient failures and explicit terminal states for validation failures.

Recommended policies:

- unsupported content type: ignore without creating a processing job
- missing source object: retry briefly, then fail with a source-object error
- Phase 0 conversion failure: retry with backoff, then fail the job
- LLM rate limit: retry with backoff and jitter; do not increase concurrency automatically
- per-slug permanent validation failure: record task failure and mark job failed or partially failed
- wiki maintenance lock contention: retry or requeue, not fail

Every queue should have a DLQ. DLQ messages must include `fileVersionId`, phase, slug when
applicable, attempt count, and a sanitized error code.

## Security and IAM

Grant the processor the minimum cross-system access:

- read the Zarya app secret or stack outputs needed to discover table and bucket names
- read DynamoDB stream records for `file-versions`
- `GetItem` on `file-versions` and `files`
- `GetObject` on committed `objects/*`
- no access to `pending-uploads/*` unless a separate reconciliation job is approved
- no write access to Zarya DynamoDB tables
- no delete access to Zarya file objects

Processor-owned roles should write only to processor-owned queues, tables, logs, and output
prefixes or buckets.

## Deployment and Environment Rules

Follow Zarya's normal environment boundaries:

| Environment | AWS CLI profile |
| ----------- | --------------- |
| `dev`       | `sdai-dev`      |
| `stage`     | `sdai-stage`    |
| `prod`      | `sdai-prod`     |

Enable and test the stream consumer in `dev` first, then `stage`, then `prod`. Do not manually
mutate `stage` or `prod` resources outside the guarded deployment path unless explicitly approved.

Enabling streams on the retained `FileVersionsTable` is a foundation-stack change and should be
reviewed in the CloudFormation change set. The processor service may live in its own stack, but it
should import or parameterize the Zarya table stream ARN, table names, bucket name, and environment.

## Observability

Minimum metrics:

- file-version events received, accepted, ignored, and rejected
- processing jobs queued, running, succeeded, and failed
- per-phase queue depth and oldest message age
- per-phase task duration and failure count
- LLM requests, tokens, rate-limit retries, and cost estimate
- wiki maintenance lock wait time
- wiki pages created, updated, and left unchanged
- contradiction pages created or updated
- DLQ message count

Minimum log fields:

- `fileVersionId`
- `fileId`
- `phase`
- `slug` when applicable
- `taskId`
- `attempt`
- `requestId` or workflow execution id
- sanitized `errorCode`

Do not log raw PDF text, Markdown text, claims, prompts, model responses, user emails, tokens, or
presigned URLs unless a separate privacy review approves it.

## Open Decisions

- Which LLM provider and rate-limit budget controls Phase 1 and Phase 2?
- What exact Zarya UI surfaces should render wiki summaries, pages, contradictions, and processing status?
- Should supported top-level files be rejected, assigned to a synthetic personal wiki, or require
  Zarya to create/select a folder before upload?
