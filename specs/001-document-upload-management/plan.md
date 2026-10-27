# Implementation Plan: Document Upload and Management

**Branch**: `001-document-upload-management` | **Date**: 2026-10-27 | **Spec**: [spec.md](spec.md)
**Input**: Stakeholder document and clarified feature specification

This plan follows the workflow in `.github/agents/speckit.plan.agent.md`.

## Summary

Deliver offline document upload, search, sharing, project/task association, authorized preview/download, and audit reporting. Use local filesystem storage outside the app/web root, EF Core metadata in SQL Server LocalDB, a .NET 8 isolated Azure Functions Queue Storage trigger running locally against Azurite, and a loopback-only ClamAV scanning service. The Blazor upload view reports byte-transfer progress, then polls persisted job status until a final verdict. New files remain quarantined and unavailable until clean; a failed replacement leaves the previous clean version available. Existing service-level authorization patterns govern every document operation and file response.

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: C# 12 / .NET 8  
**Primary Dependencies**: ASP.NET Core Blazor Server, EF Core SQL Server 8, existing cookie authentication; .NET 8 isolated Azure Functions worker v4 with Queue Storage extension; Azurite; locally installed ClamAV `clamd`/`clamdscan`; xUnit and EF Core SQLite for tests  
**Storage**: SQL Server LocalDB for metadata and durable scan jobs; configured local filesystem directory outside the application tree and `wwwroot` for quarantine/approved file bytes; Azurite Queue Storage for local scan messages  
**Testing**: New .NET 8 xUnit project; SQLite in-memory service/authorization tests, temporary-directory storage tests, fake scanner tests, Azurite queue/Functions integration tests, and documented Windows LocalDB/ClamAV schema-initializer checks against fresh and legacy databases  
**Target Platform**: Windows training workstation/server running Blazor app, local Functions host, Azurite, and ClamAV; usable offline after prerequisites are installed  
**Project Type**: Existing ASP.NET Core web application with Blazor Server UI and authenticated file streaming endpoints  
**Performance Goals**: Meet SC-001 through SC-004: process one 25,000,000-byte file within 30 seconds on a typical network; load/search up to 500 authorized documents within 2 seconds; preview supported files within 3 seconds  
**Constraints**: No Azure cloud account or cloud service at training runtime; local Azure Functions Core Tools v4 and Azurite are required; real local malware scanner required and fail-closed; 25,000,000-byte per-file limit; maximum 10 files per selection; local backups/recovery remain the training operator's responsibility; existing mock authentication is not production identity  
**Scale/Scope**: Five user journeys; six categories; lists/search for up to 500 accessible documents; project/task relationships use existing integer IDs; one configured local scanner and storage root

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Evaluation |
|---|---|
| I. Training Focus | PASS. The feature remains fictional, local, offline, and explicitly training-only; ClamAV is a local host prerequisite, not a cloud service. |
| II. User Isolation and Authorization | PASS. Document services and file-streaming endpoints perform resource-level access checks; verification covers each role and denied access. |
| III. Layered Design | PASS WITH JUSTIFIED FEATURE REQUIREMENT. Blazor pages call services; a local Functions isolated worker consumes Azurite queue messages and invokes shared document-processing services. Functions Core Tools and Azurite are required locally because the user explicitly selected the Queue-trigger architecture; no cloud service/account is used at runtime. |
| IV. Verifiable Changes | PASS. Add focused xUnit tests for service, storage, queue retries/idempotency, and access rules; exercise real ClamAV and LocalDB integration through the documented local verification procedure. |
| V. Deliberate Scope | PASS. Work is traced to the five approved stories; no cloud migration, version history, collaboration, or unrelated refactor is included. |

**Gate result**: PASS WITH JUSTIFICATION. The local Functions/Azurite prerequisite is expressly selected by the user and preserves offline operation. The existing `ApplicationDbContext` continues to use `EnsureCreated`; a shared document-only `DocumentDbContext` runs an idempotent schema initializer after it on both fresh and legacy LocalDBs. Startup MUST NOT drop/recreate an existing database or mix EF migrations into the EnsureCreated-managed database.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/document-access.md
└── tasks.md              # Generated by /speckit.tasks
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
ContosoDashboard/
├── Controllers/DocumentsController.cs      # Authenticated preview/download routes
├── Data/ApplicationDbContext.cs            # Existing users/projects/tasks database context
├── Pages/
│   ├── Documents.razor                      # My Documents / Shared with Me / team views
│   ├── DocumentReports.razor                # Administrator reports
│   ├── ProjectDetails.razor                  # Project document section
│   ├── TaskDetails.razor                    # Task document section and upload entry point
│   └── Index.razor                           # Recent documents and count
├── Services/
│   ├── DocumentService.cs                    # Metadata, workflow, search, permissions, audit
│   ├── DocumentScanQueuePublisher.cs          # Publishes durable pending scan jobs
│   └── DocumentScanJobDispatcher.cs           # Retries unpublished outbox jobs
└── Program.cs                                # DI, existing EnsureCreated, document schema initialization, routes
ContosoDashboard.DocumentProcessing/
├── Contracts/DocumentScanMessage.cs          # Queue payload (job ID + generation only)
├── Models/                                   # Document, tag, share, activity, and scan-job entities
├── Data/                                     # DocumentDbContext and idempotent schema initializer
├── Data/Schema/DocumentSchema.sql            # Tables/indexes for existing LocalDBs
└── Services/                                 # Storage, ClamAV adapter, and scan processor
ContosoDashboard.DocumentScan.Functions/
├── Functions/ScanDocumentFunction.cs         # Queue-triggered scanner
├── Functions/PoisonScanFunction.cs           # Failed-job handler
├── host.json
└── local.settings.json                       # Local only; ignored by Git
ContosoDashboard.Tests/
├── Authorization/                            # Role/resource access matrix
├── Services/                                 # Document workflow and notification tests
├── Storage/                                  # Temp-directory storage and cleanup tests
└── Queue/                                    # Dispatch, duplicate, retry, and poison tests
```

**Structure Decision**: Extend the existing Blazor Server application and its `Models`, `Data`, `Services`, `Pages`, and `Controllers` layers. Add a small shared document-processing library so the web app and .NET isolated Functions worker use the same job, storage, scan, and persistence contracts. Add a local Functions project with a Queue Storage trigger and poison handler, plus Azurite for offline queue emulation. Add one test project because there is no existing test harness and document access/scanning are security-critical. The browser-facing preview/download routes stream bytes but do not expose the storage root or define a public third-party API.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| New xUnit test project | The repository has no test project; document access, file lifecycle, and fail-closed scanning need automated regression coverage. | Keeping tests out of the application project preserves production dependencies and makes the test boundary explicit. |
| Local Functions and Azurite | The user explicitly selected an Azure Functions Queue Storage trigger while preserving offline training. | A synchronous local scan does not satisfy the requested asynchronous Queue-trigger workflow; local Functions Core Tools and Azurite preserve the Azure programming model without cloud dependency. |
