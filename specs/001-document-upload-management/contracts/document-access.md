# Document Access Contract

These authenticated browser-facing routes stream approved document bytes from private local storage. They are part of the Blazor application, not a public third-party API. Uploads and metadata mutations are initiated through authenticated Blazor pages and the document service.

## Queue Contract

The local .NET isolated Functions app consumes messages from the `document-scans` Azure Queue Storage queue. Offline training uses Azurite; no cloud account or service is required.

```json
{
	"scanJobId": 123,
	"generation": 1
}
```

- The database scan-job row is authoritative. The function loads the job/document and resolves the quarantine key from LocalDB; it never accepts a path or filename from a queue message.
- Queue delivery is at-least-once. Processing MUST be idempotent by scan-job ID and generation. Duplicate, stale-generation, or completed messages are acknowledged as no-ops.
- On an explicit infected verdict, mark the job/document rejected, remove quarantine content, and complete the queue message normally. On transient scanner/database errors, fail the invocation to request retry.
- Configure five maximum dequeue attempts. The runtime poison queue is `document-scans-poison`; its handler records failed status for operator review without making file bytes available.
- A stopped Functions host leaves queued jobs pending and files unavailable. The UI shows pending until a clean, rejected, or failed result.

## Authentication and Authorization

- Require the existing authenticated cookie and derive the requesting user ID from `NameIdentifier`; never accept a caller-supplied user ID as authority.
- Load the document metadata, then perform an imperative resource-access check before opening or streaming the file.
- Re-check current project membership and active share state on every request. Project/task documents are accessible to current project members, authorized project managers and the specific role-based managers in the feature rules. Direct shares do not bypass the project-member boundary.
- Administrators may read/download any document. Their deletion permission is enforced by the mutation service; these read routes do not permit edits, replacement, sharing, or deletion.
- Pending, rejected, missing, or otherwise unavailable documents are never streamed.
- Return the same not-found response for a nonexistent document and a document the caller is not allowed to discover, to avoid disclosing document existence.

## Routes

### `GET /documents/{documentId:int}/download`

Downloads an available document after authorization.

**Success**: `200 OK`, streamed body, validated `Content-Type`, `Content-Length`, and `Content-Disposition: attachment` with a safely encoded display filename.

### `GET /documents/{documentId:int}/preview`

Previews an available PDF or JPEG/PNG image after authorization.

**Success**: `200 OK`, streamed body, allowlisted `Content-Type`, `Content-Length`, and `Content-Disposition: inline` with a safely encoded display filename.

**Unsupported preview type**: `415 Unsupported Media Type`; the user can still use the download route.

## Common Responses and Headers

- Unauthenticated browser request: existing cookie-authentication login redirect; non-browser/API clients may receive `401 Unauthorized` according to the host's cookie configuration.
- Missing, unauthorized, pending, rejected, or deleted document: `404 Not Found`.
- Storage read failure: generic server error response; log the detailed cause server-side without exposing physical paths.
- Every response MUST set `X-Content-Type-Options: nosniff` and `Cache-Control: private, no-store`.
- The file system path and storage key MUST never be accepted from the route or returned to the browser. Physical storage MUST NOT be statically served.
- Successful download and preview requests MUST create the corresponding audit activity before or as bytes are streamed; the record includes actor, document, action, and UTC time.
