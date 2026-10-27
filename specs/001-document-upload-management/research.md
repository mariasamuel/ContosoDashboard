# Research: Document Upload and Management

## Decisions

### Local malware scanning

- **Decision**: Use a locally installed ClamAV engine on the Windows training host, exposed through a `clamd` daemon bound only to loopback. A `clamdscan` adapter will submit quarantined files and accept only an explicit clean verdict. Detection, timeout, unavailable daemon, malformed response, or any error leaves the file unavailable and returns a per-file failure.
- **Rationale**: This meets the clarified real-scanner and offline requirements. A resident daemon loads signatures once and avoids starting a new scan engine for every file. The scanner is a host prerequisite, not a cloud dependency or an application-bundled antivirus engine.
- **Alternatives considered**: `clamscan.exe` is a real one-shot scanner but reloads the signature database on each invocation; Microsoft Defender command-line results can include successful remediation in a zero exit status, making a zero-only clean decision ambiguous; cloud scanning violates offline operation.
- **Operational constraints**: The operator installs ClamAV, configures the daemon and local signature database, binds its TCP listener to `127.0.0.1` only, and keeps signatures updated through an offline-capable operator process. The application never starts or reconfigures the daemon. If it is not healthy, upload fails closed.

### File storage and upload lifecycle

- **Decision**: Store file bytes on the local filesystem outside both `wwwroot` and the application tree; keep metadata and relative storage keys in SQL Server LocalDB. Stream uploads to a private quarantine directory using server-generated paths, validate size and allowed type/signature, scan, then move clean content into approved storage before marking metadata available.
- **Rationale**: Microsoft recommends private upload locations outside the app tree, safe generated filenames, server-side size and extension checks, malware scanning, and quarantine until scanning succeeds. Blazor Server streams file data over SignalR; copying the full upload to a `MemoryStream` would add memory pressure.
- **Alternatives considered**: Storing 25 MB work files in SQL Server increases database size and read/write costs; `wwwroot` or direct static-file hosting bypasses authorization; a cloud object store conflicts with the offline requirement.
- **Data consistency**: Generate a unique relative storage key before writing. If file promotion or database persistence fails, compensate by removing the staged/promoted file and leave no available record. Deletion and replacement update the database and filesystem as a coordinated workflow; failures must not silently report success.

### Asynchronous scanning with Azure Functions and Queue Storage

- **Decision**: Run a separate .NET 8 isolated Azure Functions v4 worker locally with a Queue Storage trigger. Use Azurite's Queue Storage emulator and `AzureWebJobsStorage=UseDevelopmentStorage=true`; the training path requires no Azure account, internet connection, or cloud storage. The Blazor app stages files in the shared local quarantine root, persists a pending `DocumentScanJob`, and enqueues only its job ID and generation. The function loads the job and file from the same LocalDB/filesystem, invokes the local ClamAV scanner, and updates scan state.
- **Rationale**: This preserves the offline training boundary while meeting the requested Azure Functions + Queue Storage asynchronous architecture. The isolated .NET worker supports the app's .NET 8 target and avoids the in-process model. Azurite is Microsoft's local emulator for Queue Storage and is supported for local Functions triggers.
- **Alternatives considered**: In-process synchronous scanning does not meet the requested background queue design; Azure-hosted Queue Storage/Functions would violate offline training; the Functions in-process model is not selected.
- **Queue contract**: A message contains only `scanJobId` and `generation`; never file bytes, client filenames, or physical paths. The database job row is authoritative; the worker derives the quarantine key from it. The function no-ops when a job is complete or a message generation is stale.
- **Delivery and failure handling**: Storage Queue triggers provide at-least-once delivery. Processing must be idempotent. Configure `maxDequeueCount` explicitly to five, a nonzero visibility timeout, and bounded concurrency appropriate for a single local scanner. Transient scanner/database errors are thrown for retry. After five failed deliveries the runtime moves the message to `document-scans-poison`; a poison-queue handler marks the job failed for operator review. Pending/failed files remain unavailable. Infected files are marked rejected and the trigger completes normally after quarantine cleanup.
- **Durable enqueue**: Persist the document/job before dispatch and retry publishing from an outbox/dispatcher until queue acceptance is recorded. This prevents a process crash between database commit and queue send from leaving an upload permanently unscanned. If the function host is stopped, jobs remain queued and files remain pending/unavailable.
- **Local prerequisites**: Install Azure Functions Core Tools v4, the .NET 8 SDK, and Azurite. Keep `local.settings.json` and emulator data outside source control. The local worker must share the configured quarantine/approved storage directory and LocalDB with the Blazor app.

### Upload limits and validation

- **Decision**: Interpret the 25 MB limit as 25,000,000 bytes (decimal MB) and enforce it on the actual streamed bytes and `IBrowserFile.OpenReadStream`; do not trust browser-reported size or MIME type. Apply an explicit maximum of 10 files per selection to bound Blazor metadata and local resource use; process each file independently and sequentially.
- **Rationale**: Exact byte limits make boundary behavior testable. Microsoft warns that browser-provided file metadata is untrusted and that Blazor Server buffers can raise memory/denial-of-service risk. A finite file-count limit bounds the initial SignalR file metadata message and the total work for a single upload action.
- **Alternatives considered**: Interpreting MB as 25 MiB accepts 1,214,400 bytes more than the stated decimal limit; trusting extension or Content-Type alone is bypassable; unbounded batch count is unsafe even when each file is size-limited.
- **Validation**: Allow only the listed extensions; also compare file signatures and, for Office Open XML files, verify the expected document package structure. Use the validated type, not the client MIME type, for previews and responses.

### Authorization and file delivery

- **Decision**: Keep metadata rules in document services and stream file bytes only through authenticated download/preview routes that re-check authorization on every request. Never expose physical paths or storage keys in URLs. Query search/list results through the same access rules before returning records.
- **Rationale**: ASP.NET Core resource authorization is imperative because `[Authorize]` runs before resource loading and cannot alone authorize an individual document. This matches the app's existing service-level authorization style and covers guessed IDs, changed membership, and revoked shares.
- **Alternatives considered**: Static files and direct filesystem URLs bypass per-document checks; page-only authorization does not protect a document by ID.

### Persistence and testing

- **Decision**: Keep the existing training-only `EnsureCreated` startup for existing application tables. Add a shared document-processing library with a document-only `DocumentDbContext` and an idempotent SQL schema initializer that runs after `EnsureCreated` on every startup. On both a fresh database and an existing `EnsureCreated` database, the initializer creates only missing document tables/indexes. Never automatically drop or recreate an existing LocalDB. Add a .NET 8 xUnit test project with SQLite in-memory for service and authorization tests, a fake scanner, and temporary local storage.
- **Rationale**: EF Core documents that `EnsureCreated` does not initialize schema when any tables already exist and does not work well with migrations. Because document data is owned by a separate context and its schema initializer is always applied after the current app context creates its tables, both new and existing LocalDBs get the document schema without a migration baseline or data reset.
- **Alternatives considered**: Leaving `EnsureCreated` unchanged would omit the feature tables from existing training databases; replacing it with migrations would require a deliberate baseline or reset of existing databases; automatic database reset would destroy local training data; a test-only mock without authorization and storage tests would leave high-risk behavior unverified.
- **Validation constraint**: Test initialization against both an empty database after the existing app `EnsureCreated` runs and a database created from the current pre-feature schema. Re-running the initializer must be safe and must not alter existing rows. Fail with actionable guidance rather than delete data if the schema cannot be updated safely.

## Sources

- [ASP.NET Core Blazor file uploads (.NET 8)](https://learn.microsoft.com/en-us/aspnet/core/blazor/file-uploads?view=aspnetcore-8.0): streaming, server-side upload progress, explicit stream size, untrusted browser metadata, SignalR considerations.
- [ASP.NET Core file upload security (.NET 8)](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/file-uploads?view=aspnetcore-8.0): storage outside the app tree, generated filenames, extension/signature validation, size limits, scanning and quarantine.
- [Resource-based authorization in ASP.NET Core (.NET 8)](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?view=aspnetcore-8.0): resource authorization requires an imperative authorization check after loading the resource.
- [EF Core migrations overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) and [EnsureCreated limitations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/ensure-created): migrations preserve existing data, while `EnsureCreated` does not evolve existing schemas and is not compatible with migrations.
- [Azure Queue Storage trigger for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue-trigger): isolated C# worker triggers, at-least-once retries, poison queues, visibility timeout, and concurrency settings.
- [Develop and run Azure Functions locally](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local): Core Tools v4, local settings, and Azurite for local storage triggers.
- [Azurite local storage emulator](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite): local Queue Storage emulation for development and testing.
- [Azure Functions best practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices): Functions runtime, isolated worker, concurrency, and operational guidance.
- [ClamAV Windows installation](https://docs.clamav.net/manual/Installing.html): official Windows installer and configuration prerequisites.
- [ClamAV scanning](https://docs.clamav.net/manual/Usage/Scanning.html): `clamscan` loads signatures on each run; `clamdscan` uses a running daemon; TCP scanner access is unauthenticated.
- [ClamD protocol](https://docs.clamav.net/manual/Usage/ClamdProtocol.html): loopback-only TCP guidance, explicit `OK`/`FOUND`/`ERROR` replies, and stream support.
- [ClamAV configuration](https://docs.clamav.net/manual/Usage/Configuration.html): Windows config files, signature database paths, and daemon configuration requirements.
