# Data Model: Document Upload and Management

## Existing Entities Reused

- **User** (`UserId`, `Role`, `Department`, `DisplayName`): document owner/uploader, share recipient, actor, and source for existing role/department rules.
- **Project** (`ProjectId`, `ProjectManagerId`, `ProjectMembers`): project association and project-manager/member access.
- **TaskItem** (`TaskId`, `ProjectId`): optional task association. A task-linked document inherits its task's project association and access rules.
- **Notification** (`NotificationId`, `UserId`, `Type`, `CreatedDate`): in-app share and project-document notifications; extend `NotificationType` with document event values.

## New Entities

### Document

Represents a document's metadata and the key to its approved local file.

| Field | Type / Constraint | Meaning |
|---|---|---|
| `DocumentId` | `int`, primary key | Existing integer-key convention |
| `Title` | required, max 255 | User-facing title |
| `Description` | optional, max 2000 | User-facing description |
| `Category` | required text, max 50 | One of the six specified category labels; stored as text, not an enum integer |
| `OriginalFileName` | required, max 255 | Sanitized display name only; never used as a path |
| `FileType` | required, max 255 | Validated MIME type, determined from accepted content rather than trusted browser input |
| `FileSizeBytes` | required `long` | Actual size counted while streaming; range 1 to 25,000,000 bytes |
| `StorageKey` | nullable until clean, max 1024, unique when present | App-generated relative key for the approved file; never a client path or public URL |
| `UploadedByUserId` | required FK to `User` | Owner/uploader |
| `ProjectId` | optional FK to `Project` | Project context, if any |
| `TaskId` | optional FK to `TaskItem` | Task context, if any; when set, `ProjectId` must match the task's project |
| `ScanStatus` | required state | `Pending`, `Clean`, `Rejected`, or `Failed`; new files are available only in `Clean` state |
| `ScanGeneration` | required integer | Incremented for each upload or replacement scan; rejects stale/duplicate work for earlier file content |
| `CreatedDate` | required UTC timestamp | Upload time |
| `UpdatedDate` | required UTC timestamp | Last metadata or file replacement update |

**Relationships and indexes**:

- One user uploads many documents; one project or task can have many documents.
- Index `UploadedByUserId`, `ProjectId`, `TaskId`, `Category`, `CreatedDate`, and `ScanStatus` for scoped lists and dashboard queries.
- Search across title, description, uploader display name, project name, and tags is constrained to authorized records before results are returned.
- Sort/filter/pagination queries MUST remain scoped to authorized records. The expected per-user result set is up to 500 documents.

### DocumentTag

Stores each normalized search tag as a relational row rather than an opaque serialized list.

| Field | Type / Constraint | Meaning |
|---|---|---|
| `DocumentTagId` | `int`, primary key | Tag-row identifier |
| `DocumentId` | required FK to `Document` | Parent document |
| `Value` | required text, max 100 | Trimmed user-provided tag |

Enforce uniqueness on `(DocumentId, Value)` and cascade tag rows when a document is deleted. Tag count/input length limits are configured server-side and validated consistently in the UI and service.

### DocumentScanJob

Durable work item and transactional outbox row for each document upload or replacement scan.

| Field | Type / Constraint | Meaning |
|---|---|---|
| `DocumentScanJobId` | `int`, primary key | Queue-work identifier |
| `DocumentId` | required FK to `Document` | Document being scanned |
| `Generation` | required integer | Must match the document's current scan generation |
| `QuarantineKey` | required, max 1024 | App-generated relative quarantine key; never put the path in the queue message |
| `State` | required state | `PendingEnqueue`, `Queued`, `Processing`, `Completed`, `Rejected`, or `Failed` |
| `CreatedDate` | required UTC timestamp | Job creation time |
| `UpdatedDate` | required UTC timestamp | Last state transition |
| `AttemptCount` | integer | Observed attempts for diagnostics |
| `FailureCode` | optional, max 100 | Safe diagnostic category, not raw filesystem/scanner output |

Enforce uniqueness on `(DocumentId, Generation)`. The durable job row is inserted with document metadata before queue dispatch. A dispatcher publishes pending jobs and marks queue acceptance; a crash between send and marking can produce duplicates, so the function must be idempotent by job ID and generation.

### DocumentShare

Represents a revocable grant to a user or department. It never replaces a project's membership boundary.

| Field | Type / Constraint | Meaning |
|---|---|---|
| `DocumentShareId` | `int`, primary key | Share identifier |
| `DocumentId` | required FK to `Document` | Shared document |
| `SharedByUserId` | required FK to `User` | Owner initiating the share |
| `RecipientUserId` | nullable FK to `User` | Specific user recipient |
| `RecipientDepartment` | nullable text, max 100 | Department recipient |
| `CreatedDate` | required UTC timestamp | Share time |
| `RevokedDate` | nullable UTC timestamp | Revocation time; null means active |
| `RevokedByUserId` | nullable FK to `User` | Actor who revoked the share |

**Validation**:

- Exactly one of `RecipientUserId` or `RecipientDepartment` is supplied.
- A project- or task-associated document can only be shared when the recipient is a current member of its associated project. Access checks re-check membership on each request; an old share does not bypass lost membership.
- A recipient gains access only while the share is active. Revocation immediately denies subsequent list, search, preview, and download attempts.
- Administrators cannot create or revoke shares. Owners can create/revoke shares for documents they own; manager permissions do not imply sharing permission.

### DocumentActivity

An append-only audit record for document actions. It contains no file content.

| Field | Type / Constraint | Meaning |
|---|---|---|
| `DocumentActivityId` | `int`, primary key | Activity identifier |
| `DocumentId` | nullable FK to `Document` | Set null if the document is permanently deleted |
| `DocumentTitleSnapshot` | required text, max 255 | Non-content title retained for delete/audit reports |
| `ActorUserId` | nullable FK to `User` | Authenticated actor, if retained |
| `ActorDisplayNameSnapshot` | required text, max 255 | Display attribution for reports after later account changes |
| `Action` | required text/state | `Upload`, `Download`, `Delete`, `Share`, `RevokeShare`, `Replace`, `ScanRejected`, or `ScanFailed` |
| `CreatedDate` | required UTC timestamp | Action time |

Index `(CreatedDate)`, `(ActorUserId, CreatedDate)`, `(Action, CreatedDate)`, and `DocumentId`. Activity is append-only and available in administrative reporting; deletion removes the file and document/share/tag rows while retaining the minimal activity event.

## File Lifecycle

1. A file is streamed to private quarantine while actual bytes, extension, signature, and size are checked. Document metadata and a `PendingEnqueue` scan-job/outbox row are committed before queue dispatch.
2. The dispatcher sends `{scanJobId, generation}` to `document-scans` and marks queue acceptance. Message bodies contain no file bytes, client filename, or path. A new document remains pending, invisible, and unavailable until scanning finishes.
3. The Azure Functions worker loads authoritative job/document state from LocalDB, verifies its generation, scans the quarantine file, and atomically promotes a clean result. Stale, duplicate, or completed messages are acknowledged as no-ops.
4. An infected file is removed from quarantine and marked rejected; the queue message completes normally. Transient scanner/database errors are thrown for retry. After five failed deliveries, the runtime moves the message to `document-scans-poison`; a poison handler marks the job/document failed and keeps file bytes unavailable.
5. Replacement creates a new generation/job and stages a new file while the current clean file remains available. On a clean result, switch the current storage key and remove the prior file. On rejection or exhausted retries, leave the previous document unchanged.
6. On confirmed deletion, remove current/quarantined bytes, metadata, tags, shares, and scan jobs; retain the minimal deletion activity record. A filesystem deletion failure is reported and must not be presented as a completed permanent deletion.

## Access Rules

- **Employee**: own non-project documents, active direct shares, and documents available through current project membership.
- **TeamLead**: employee access plus document management for documents uploaded by users in their department, according to the feature's team-lead management requirements.
- **ProjectManager**: employee access plus management of documents associated with projects they manage.
- **Administrator**: view/download any document, read audit reports, and delete after confirmation; cannot edit metadata, replace files, or share.
- For project/task documents, active project membership is re-evaluated for normal member/share access. Project-manager and team-lead management permissions are separately checked for the specific operation.
- A task with no associated project can still have an owner-scoped attachment, but it has no project recipients and cannot be directly shared under the project-member-only rule.
- Every list/search result and every file byte request uses the same service-level access decision. UI visibility is not an authorization boundary.
