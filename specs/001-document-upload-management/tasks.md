---

description: "Dependency-ordered tasks for Document Upload and Management"

---

# Tasks: Document Upload and Management

**Input**: Design documents in `specs/001-document-upload-management/`  
**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/document-access.md`, `quickstart.md`

**Verification**: The repository currently has no test project. Add the planned .NET 8 xUnit harness and focused tests for behavior and authorization. LocalDB/ClamAV/Azurite end-to-end checks, UI usability/performance measurements, and the operator backup/restore exercise require local prerequisites and remain repeatable manual checks; they do not replace automated service and integration tests.

**Organization**: Tasks are ordered by priority user story. Shared schema, storage, scanner, queue, and authorization foundations block every story.

## Phase 1: Setup

**Purpose**: Create the feature's shared processing, worker, and test project scaffolding without changing unrelated application behavior.

- [ ] T001 Create the .NET 8 shared processing class library project in `ContosoDashboard.DocumentProcessing/ContosoDashboard.DocumentProcessing.csproj`
- [ ] T002 Create the .NET 8 isolated Azure Functions worker project in `ContosoDashboard.DocumentScan.Functions/ContosoDashboard.DocumentScan.Functions.csproj`
- [ ] T003 [P] Create the .NET 8 xUnit test project with EF Core SQLite, Functions worker test support, and project test dependencies in `ContosoDashboard.Tests/ContosoDashboard.Tests.csproj`
- [ ] T004 Add project references from `ContosoDashboard/ContosoDashboard.csproj`, `ContosoDashboard.DocumentScan.Functions/ContosoDashboard.DocumentScan.Functions.csproj`, and `ContosoDashboard.Tests/ContosoDashboard.Tests.csproj` to `ContosoDashboard.DocumentProcessing/ContosoDashboard.DocumentProcessing.csproj`

## Phase 2: Foundational

**Purpose**: Establish shared entities, safe persistence, private file handling, scanner integration, queue processing, and service-boundary authorization before any story work.

**⚠️ CRITICAL**: All foundational tasks must be complete before user-story implementation begins.

- [ ] T005 [P] Add `Document`, `DocumentTag`, `DocumentScanJob`, `DocumentShare`, and append-only `DocumentActivity` entities with the constraints in `specs/001-document-upload-management/data-model.md` to `ContosoDashboard.DocumentProcessing/Models/Document.cs`, `ContosoDashboard.DocumentProcessing/Models/DocumentTag.cs`, `ContosoDashboard.DocumentProcessing/Models/DocumentScanJob.cs`, `ContosoDashboard.DocumentProcessing/Models/DocumentShare.cs`, and `ContosoDashboard.DocumentProcessing/Models/DocumentActivity.cs`
- [ ] T006 Configure `DocumentDbContext` and add the idempotent document-only tables, foreign keys, uniqueness constraints, and indexes in `ContosoDashboard.DocumentProcessing/Data/DocumentDbContext.cs` and `ContosoDashboard.DocumentProcessing/Data/Schema/DocumentSchema.sql`
- [ ] T007 [P] Add schema-initializer tests for fresh and existing `EnsureCreated` databases, repeat initialization, and preservation of existing rows in `ContosoDashboard.Tests/Data/DocumentSchemaInitializerTests.cs`
- [ ] T008 Implement the configured private filesystem storage abstraction with generated relative keys, quarantine/approved roots, streaming, promotion, and cleanup in `ContosoDashboard.DocumentProcessing/Services/LocalDocumentStorage.cs`; test containment and cleanup in `ContosoDashboard.Tests/Storage/LocalDocumentStorageTests.cs`
- [ ] T009 Implement the loopback-only ClamAV adapter that accepts only explicit clean verdicts and fails closed on timeout, unavailable service, malformed response, or scan error in `ContosoDashboard.DocumentProcessing/Services/ClamAvScanner.cs`; cover verdicts and failures in `ContosoDashboard.Tests/Services/ClamAvScannerTests.cs`
- [ ] T010 Implement app-specific resource-level document authorization for owner, department team lead, project manager/member, active share, and administrator operation rules in `ContosoDashboard/Services/DocumentAuthorizationService.cs`; cover allowed and denied role/resource combinations, anonymous access, and cross-project access in `ContosoDashboard.Tests/Authorization/DocumentAuthorizationTests.cs`
- [ ] T011 Implement the `{scanJobId, generation}` queue contract, durable pending-enqueue dispatcher, and retryable queue publisher in `ContosoDashboard.DocumentProcessing/Contracts/DocumentScanMessage.cs` and `ContosoDashboard/Services/DocumentScanJobDispatcher.cs`; test persistence-before-publication, duplicate delivery, and retry behavior in `ContosoDashboard.Tests/Queue/DocumentScanJobDispatcherTests.cs`
- [ ] T012 Implement the isolated queue-trigger scan processor and poison-queue handler, with five dequeue attempts, bounded local concurrency, idempotency, stale-generation no-ops, quarantine cleanup, and failed-file unavailability in `ContosoDashboard.DocumentScan.Functions/Functions/ScanDocumentFunction.cs`, `ContosoDashboard.DocumentScan.Functions/Functions/PoisonScanFunction.cs`, and `ContosoDashboard.DocumentScan.Functions/host.json`; test queue outcomes in `ContosoDashboard.Tests/Queue/ScanDocumentFunctionTests.cs`
- [ ] T013 Register the document context, safe schema initialization after the existing `EnsureCreated`, storage, scanner, authorization, dispatcher, and worker-facing services in `ContosoDashboard/Program.cs` without database drop/recreation or EF migrations

**Checkpoint**: Shared database, safe private storage, fail-closed scanning, queue processing, and the resource authorization boundary are ready for story work.

## Phase 3: User Story 1 - Upload and organize documents (Priority: P1) 🎯 MVP

**Goal**: Employees submit one or more files with required metadata and optional description, tags, and project/task context; each result remains pending and unavailable until it has a clean scan, then appears in the uploader's list.

**Independent Test**: Upload a supported clean file with valid metadata and confirm pending status, clean completion, correct metadata, and visibility only to its uploader. Repeat with an unsupported type, 25,000,001-byte file, malicious/inconclusive scan, and mixed valid/invalid batch; verify independent failures and no unavailable file exposure.

### Verification for User Story 1

- [ ] T014 [P] [US1] Add focused tests for supported extensions/signatures, Office package validation, decimal size boundaries, ten-file selection limit, actual streamed-byte counting, invalid metadata, and independent multi-file results in `ContosoDashboard.Tests/Services/DocumentUploadValidatorTests.cs`
- [ ] T015 [P] [US1] Add upload-workflow tests for pending status, queue handoff, per-file terminal results, failed-closed scans, and unavailable bytes before a clean verdict in `ContosoDashboard.Tests/Services/DocumentUploadWorkflowTests.cs`

### Implementation for User Story 1

- [ ] T016 [US1] Implement server-side file-type/signature validation and actual streamed-byte enforcement for the 25,000,000-byte and ten-file limits in `ContosoDashboard.DocumentProcessing/Services/DocumentUploadValidator.cs`
- [ ] T017 [US1] Implement per-file upload orchestration that streams to quarantine, validates metadata/context, persists document and outbox job before dispatch, and records clean/rejected/failed outcomes in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T018 [P] [US1] Add the upload form with required title/category, optional description/tags/project/task, extension guidance, and client-side metadata validation in `ContosoDashboard/Pages/DocumentUploadForm.razor`
- [ ] T019 [US1] Add byte-transfer progress, per-file pending/final status display, and persisted scan-status polling in `ContosoDashboard/Pages/DocumentUploadForm.razor.cs`
- [ ] T020 [US1] Add the uploader's personal document list showing title, category, upload date, actual size, project, and scan status in `ContosoDashboard/Pages/MyDocuments.razor`
- [ ] T021 [US1] Add the focused upload journey integration tests using SQLite, temporary storage, a fake scanner, and a queue test double in `ContosoDashboard.Tests/Services/DocumentUploadJourneyTests.cs`

**Checkpoint**: Story 1 is independently testable with both an automated fake-scanner workflow and the documented local real-scanner procedure.

## Phase 4: User Story 2 - Find and access authorized documents (Priority: P1)

**Goal**: Users browse, filter, sort, search, preview, and download only documents they are currently authorized to access.

**Independent Test**: Seed documents with different owners, categories, projects, tags, shares, and scan states; verify authorized list/search/filter/sort results and permitted preview/download, then verify missing, pending, rejected, revoked-share, lost-membership, unrelated-user, and unauthenticated requests disclose no metadata or bytes.

### Verification for User Story 2

- [ ] T022 [P] [US2] Add list/search/filter/sort tests proving authorization is applied before results and counts, including revoked shares, changed project membership, guessed IDs, and pending/rejected documents in `ContosoDashboard.Tests/Authorization/DocumentQueryAuthorizationTests.cs`
- [ ] T023 [P] [US2] Add route contract tests for authenticated download/preview responses, safe headers, not-found equivalence, unsupported preview types, and audit-before-stream behavior in `ContosoDashboard.Tests/Controllers/DocumentsControllerTests.cs`

### Implementation for User Story 2

- [ ] T024 [US2] Implement authorized document query, title/description/tag/uploader/project search, category/project/date filters, stable sorting, and bounded pagination in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T025 [US2] Implement authenticated `/documents/{documentId:int}/download` and `/documents/{documentId:int}/preview` streaming routes with resource checks, clean-state checks, safe filenames, allowlisted preview types, `nosniff`, `private, no-store`, and no physical-path disclosure in `ContosoDashboard/Controllers/DocumentsController.cs`
- [ ] T026 [US2] Add the searchable, filterable, sortable authorized document browser and empty/invalid-date states in `ContosoDashboard/Pages/Documents.razor`
- [ ] T027 [US2] Add the My Documents, Shared with Me, and role-appropriate team/project navigation and links to guarded preview/download routes in `ContosoDashboard/Pages/Documents.razor`
- [ ] T028 [US2] Add automated query and streaming integration coverage for up to 500 seeded accessible records and ensure inaccessible records are excluded in `ContosoDashboard.Tests/Services/DocumentAccessJourneyTests.cs`
- [ ] T029 [US2] Add repeatable local verification for unauthenticated, guessed-ID, changed-membership, revoked-share, pending-file, static-URL, and response-header cases in `specs/001-document-upload-management/quickstart.md`

**Checkpoint**: Story 2 is independently testable against seeded data and enforces the same service-level access decision for lists, searches, previews, and downloads.

## Phase 5: User Story 3 - Manage document details and lifecycle (Priority: P2)

**Goal**: Owners, department team leads, and project managers can perform their permitted metadata edits, replacements, and confirmed permanent deletions; unauthorized operations and administrator edit/replacement/share attempts are denied.

**Independent Test**: As owner, team lead, project manager, administrator, and unrelated employee, try metadata updates, replacement, and confirmed deletion across own, same-department, managed-project, and unrelated documents. Verify allowed and denied access, clean replacement promotion, preservation of the old clean file on replacement failure, removal of recipients on deletion, and retained delete activity.

### Verification for User Story 3

- [ ] T030 [P] [US3] Add operation-matrix tests for allowed and denied metadata edits, replacement, and deletion across owner/team-lead/project-manager/administrator roles, including anonymous and cross-resource denials in `ContosoDashboard.Tests/Authorization/DocumentLifecycleAuthorizationTests.cs`
- [ ] T031 [P] [US3] Add replacement and deletion tests for clean promotion, stale generation, rejected/failed replacement preserving the old clean version, confirmed permanent cleanup, revoked recipients, and retained audit activity in `ContosoDashboard.Tests/Services/DocumentLifecycleTests.cs`

### Implementation for User Story 3

- [ ] T032 [US3] Implement service-boundary metadata updates for title, description, category, and normalized tags with operation-specific authorization and activity records in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T033 [US3] Implement replacement as a new scan generation that keeps the current clean storage key available until successful promotion and leaves it unchanged on rejection or exhausted retries in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T034 [US3] Implement confirmed permanent deletion of file/quarantine content, document metadata, tags, shares, and jobs while retaining a minimal deletion activity record and reporting cleanup failures in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T035 [US3] Add metadata-edit, replacement, and confirmed-delete controls with role-appropriate visibility and clear failure outcomes in `ContosoDashboard/Pages/DocumentDetails.razor`
- [ ] T036 [US3] Add service integration tests proving denied mutations do not alter metadata or file state and successful actions create the required activity in `ContosoDashboard.Tests/Services/DocumentLifecycleJourneyTests.cs`

**Checkpoint**: Story 3 can be tested independently with a role/resource matrix and temporary-file lifecycle assertions.

## Phase 6: User Story 4 - Share documents and connect them to work (Priority: P2)

**Goal**: Owners share with eligible people or departments and revoke access; task/project attachments follow current project membership, notifications are created, and the dashboard displays the user's five latest documents and count.

**Independent Test**: Share a non-project document with one user and a department; verify recipient list visibility and notifications. Attempt a project-document share with a nonmember, then with a current member; revoke the successful share and verify immediate denial. Attach a document to a project task and verify task/project access follows membership; verify dashboard recent-five/count results are scoped to the current user.

### Verification for User Story 4

- [ ] T037 [P] [US4] Add share/revoke tests for owner-only sharing, exact-one recipient selection, department recipients, project-membership restrictions, immediate access revocation, and denied administrator/manager share operations in `ContosoDashboard.Tests/Services/DocumentSharingTests.cs`
- [ ] T038 [P] [US4] Add project/task attachment, membership change, document notification, and current-user dashboard recent-five/count tests in `ContosoDashboard.Tests/Services/DocumentWorkAssociationTests.cs`

### Implementation for User Story 4

- [ ] T039 [US4] Implement owner-only share creation and revocation, rechecking active share and project membership for each access, and record share/revoke activity in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T040 [US4] Extend in-app notification types and create share and project-document notifications for the eligible recipients in `ContosoDashboard/Models/Notification.cs` and `ContosoDashboard/Services/DocumentService.cs`
- [ ] T041 [US4] Implement task association that derives and validates its project ID and rejects unavailable or inconsistent project/task/recipient context in `ContosoDashboard/Services/DocumentService.cs`
- [ ] T042 [US4] Add share/revoke controls and the Shared with Me view with clear access and notification outcomes in `ContosoDashboard/Pages/DocumentSharing.razor`
- [ ] T043 [US4] Add project/task document sections and the current-user recent-five-document list and document count to `ContosoDashboard/Pages/ProjectDetails.razor`, `ContosoDashboard/Pages/TaskDetails.razor`, and `ContosoDashboard/Pages/Index.razor`

**Checkpoint**: Story 4 is independently testable for direct shares, department recipients, project/task membership constraints, notifications, and dashboard summaries.

## Phase 7: User Story 5 - Review document activity (Priority: P3)

**Goal**: Administrators review append-only document activity and generate document-type, uploader, and access-pattern summaries; every non-administrator is denied.

**Independent Test**: Generate upload, download, share, revoke, replacement, scan-failure, and deletion activity; verify administrator event details and summaries, including deleted-document snapshots, and verify non-administrators cannot view or query reports.

### Verification for User Story 5

- [ ] T044 [P] [US5] Add administrator report authorization tests for every supported role, including anonymous and non-administrator denials in `ContosoDashboard.Tests/Authorization/DocumentReportsAuthorizationTests.cs`
- [ ] T045 [P] [US5] Add report-query tests for action/actor/document/time fields, deleted-document snapshots, document-type totals, active uploader counts, and access-pattern summaries in `ContosoDashboard.Tests/Services/DocumentReportsTests.cs`

### Implementation for User Story 5

- [ ] T046 [US5] Implement administrator-only activity querying and document-type, uploader, and access-pattern aggregation in `ContosoDashboard/Services/DocumentReportService.cs`
- [ ] T047 [US5] Add the authenticated administrator activity and summary report page with filters and empty states in `ContosoDashboard/Pages/DocumentReports.razor`

**Checkpoint**: Story 5 can be independently verified by generating known activity and comparing report data and role denials.

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Verify the integrated offline workflow, preserve local data, and document operational checks without expanding feature scope.

- [ ] T048 Add an end-to-end local verification checklist for fresh and legacy LocalDB initialization, real ClamAV clean/EICAR/fail-closed outcomes, Azurite outage/retry/poison behavior, replacement failure, and multi-file results in `specs/001-document-upload-management/quickstart.md`
- [ ] T049 Add repeatable manual measurements for 25,000,000-byte upload within 30 seconds, list/search of 500 authorized documents within 2 seconds, preview within 3 seconds, and upload usability outcomes in `specs/001-document-upload-management/quickstart.md`
- [ ] T050 Document and exercise the operator-owned backup/restore procedure for LocalDB metadata and private file storage before training rollout in `specs/001-document-upload-management/quickstart.md`
- [ ] T051 Run the full test suite and application/Functions builds using the documented .NET commands, and record any unmet coverage limitations in `specs/001-document-upload-management/quickstart.md`
- [ ] T052 Verify local-only settings, Azurite data, scanner configuration, and document storage are outside source control and outside the app/web root; update ignore rules only if required in `.gitignore`

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No prerequisite; create the shared library, Functions worker, and test harness.
- **Foundational (Phase 2)**: Depends on Setup; blocks all five user stories.
- **User Stories (Phases 3–7)**: Depend on the completed foundation. Priority order is P1 (US1, US2), P2 (US3, US4), then P3 (US5).
- **Polish (Phase 8)**: Depends on the stories selected for delivery and their automated tests.

### User Story Dependencies

- **US1 (P1)**: Starts after the foundation; no dependency on another story. MVP story.
- **US2 (P1)**: Starts after the foundation and depends on US1's document and clean-scan workflow for meaningful file browsing and delivery; query and route authorization can be developed/tested with seeded records.
- **US3 (P2)**: Starts after the foundation and US1's persisted document workflow; edits, replacement, and deletion operate on the shared document and storage lifecycle.
- **US4 (P2)**: Starts after the foundation and US1's persisted document workflow; story-level access checks build on US2's common authorized query behavior.
- **US5 (P3)**: Starts after the foundation; report queries can be tested with seeded activity, while end-to-end event coverage depends on the story actions that emit events.

### Within Each User Story

- Add focused automated tests before implementation where practical; implement models/shared behavior before consumers and UI.
- Foundation-level entities, storage, scanning, queue semantics, and authorization must not be duplicated in story phases.
- Complete the independent test criteria at each story checkpoint before treating that story as delivered.
- LocalDB, real ClamAV, Azurite, performance/usability, and backup/restore checks are manual environmental checks; xUnit tests remain the automated regression gate.

### Parallel Opportunities

- In Setup, T003 can run independently from T001 and T002; T004 follows project creation.
- In Foundation, entity/schema work (T005–T006), storage (T008), scanner (T009), and authorization (T010) can proceed in parallel where files do not overlap; initializer tests (T007) can be authored against the planned schema/initializer contract.
- After Foundation, US1 and US2 test authoring tasks can proceed in parallel; the story phases remain sequenced where the dependency graph requires the persisted document workflow.
- Within each story, parallel `[P]` verification tasks use distinct test files and may be authored together. Avoid concurrent changes to the shared `DocumentService.cs`; merge service changes serially across stories.

### Parallel Example: User Story 1

```text
After T013, author T014 and T015 in parallel:
- T014: upload validation boundary and batch tests in ContosoDashboard.Tests/Services/DocumentUploadValidatorTests.cs
- T015: pending/scan workflow tests in ContosoDashboard.Tests/Services/DocumentUploadWorkflowTests.cs
Then implement T016–T020 in dependency order; run T021 after integration.
```

### Parallel Example: User Story 2

```text
After the shared document workflow is available, author T022 and T023 in parallel:
- T022: authorized query tests in ContosoDashboard.Tests/Authorization/DocumentQueryAuthorizationTests.cs
- T023: protected streaming route tests in ContosoDashboard.Tests/Controllers/DocumentsControllerTests.cs
Implement the query service, controller, and browser in sequence; finish with T028–T029.
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Phase 1 Setup and Phase 2 Foundation.
2. Complete Phase 3 US1: upload validation, independent per-file status, durable scan handoff, and personal list.
3. Run automated upload/storage/queue tests and repeat the real local ClamAV/Azurite upload checks in `quickstart.md`.
4. Stop and validate US1 independently before beginning further stories.

### Incremental Delivery

1. Setup + Foundation → safe persistence, private storage, scan queue, and service authorization.
2. US1 → upload and organize documents; independently validate as the MVP.
3. US2 → authorized discovery and file delivery; validate role/resource boundaries.
4. US3 → lifecycle management; validate replacement preservation and permanent deletion.
5. US4 → sharing and work-item integration; validate notification and membership behavior.
6. US5 → administrator activity reporting; validate audit summaries and denials.
7. Polish → complete full tests, local operational checks, performance measurements, and operator restore exercise.

## Notes

- Every task uses the required checkbox, sequential ID, optional `[P]`, story label only in story phases, and an exact repository-relative file path.
- `[P]` is reserved for tasks that can work on distinct files without waiting on incomplete dependencies.
- No existing automated test harness was found; the plan's new xUnit project supplies the harness. Manual coverage remains necessary for actual Windows LocalDB, ClamAV, Azurite, UI usability/performance, and operator restore behavior.
- Keep the feature offline and training-only. Do not introduce cloud runtime services, production identity, version history, or unrelated refactoring.
