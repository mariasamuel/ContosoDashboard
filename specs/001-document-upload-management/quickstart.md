# Quickstart: Document Upload and Management

## Prerequisites

- Windows with the .NET 8 SDK and SQL Server LocalDB, as required by the existing application.
- Azure Functions Core Tools v4 and the .NET 8 isolated worker prerequisites.
- Azurite installed locally to emulate Queue Storage; Node.js/npm is required when installing Azurite with npm.
- The local ClamAV Windows build installed and configured with a real signature database. Configure `clamd` to listen on loopback only (`127.0.0.1`); never expose its unauthenticated TCP port to another host or network.
- A private writable document root outside `ContosoDashboard`, the repository, and `wwwroot`. The app identity needs read/write access; the directory must not grant execute permission or static-file access.
- A documented operator backup/restore procedure for both LocalDB metadata and local document files.
- The Blazor app, local Functions host, Azurite, and ClamAV daemon run on the same machine and can access the same LocalDB and private filesystem paths.

## Configure the Local Scanner and Storage

1. Install ClamAV from the official Windows downloads and verify the downloaded package according to its published verification guidance.
2. Create and configure `freshclam.conf` and `clamd.conf` following the official Windows setup guide. Provision current signatures before disconnecting from the internet; the app itself does not require network access.
3. Set the daemon's TCP listener to `127.0.0.1` and a locally agreed port. Configure its stream limit to at least 25,000,000 bytes. Keep logs and signature databases outside the repository.
4. Start `clamd` and verify it is healthy with the configured ClamAV client. Verify a known clean file returns an explicit clean verdict. If the daemon, signatures, or verdict are unavailable/inconclusive, the application must reject the upload.
5. Set local configuration values (for example through user-scoped environment variables) for the private storage root, `clamdscan.exe` path, `clamd.conf` path, scan timeout, 25,000,000-byte per-file limit, and 10-file selection limit. Do not commit machine-specific paths or signature data.
6. Start Azurite bound to loopback with persistent local data, for example `azurite --queueHost 127.0.0.1 --location <local-data-dir>`. No Azure account or cloud storage resource is used.
7. Configure the Functions project's local-only `local.settings.json` with `FUNCTIONS_WORKER_RUNTIME=dotnet-isolated` and `AzureWebJobsStorage=UseDevelopmentStorage=true`, plus the LocalDB, quarantine root, and scanner settings. Ensure `local.settings.json` and Azurite data are ignored by Git.

## Initialize and Run

1. From the repository root, confirm the active branch is `001-document-upload-management`.
2. Before updating a database created by the existing `EnsureCreated` startup, follow the operator's backup procedure. The app's existing context creates its current tables; the shared document context then applies its idempotent document-only schema initializer on both fresh and legacy databases. It must not drop or recreate an existing LocalDB, and must not apply EF migrations to this `EnsureCreated`-managed database.
3. Restore and run the test suite:

   ```powershell
   dotnet test ContosoDashboard.Tests/ContosoDashboard.Tests.csproj
   ```

4. Start the app:

   ```powershell
   dotnet run --project ContosoDashboard/ContosoDashboard.csproj
   ```

5. In another terminal, start the queue-trigger worker from its project directory:

   ```powershell
   Set-Location ContosoDashboard.DocumentScan.Functions
   func start
   ```

6. Sign in with a seeded training user and open Documents. Verify My Documents, Shared with Me, and role-appropriate team/project views. New uploads should remain pending until the local queue-trigger worker records the scan verdict.

## Focused Verification

- Test files at 25,000,000 bytes and 25,000,001 bytes; verify the latter is rejected even if the browser reports a smaller size. Test each allowed extension and mismatched extension/signature pairs.
- With a working local scanner, upload a clean supported test file and confirm it is available only after a clean verdict. Stop or misconfigure the scanner and verify uploads fail closed. In an isolated test environment, use the official EICAR test file to verify detection; do not use live malware.
- Stop the Functions host after an upload and confirm the file remains pending/unavailable. Restart the host and verify the queued job completes.
- Deliver a duplicate scan message and a stale generation; verify both are idempotent no-ops. Simulate transient failures through the configured retry count and confirm the fifth failed delivery reaches `document-scans-poison`, marks the job failed, and leaves the file unavailable.
- Stop Azurite during queue publication and verify the durable job remains eligible for publishing after Azurite returns. The app must not silently mark the document clean or lose its scan request.
- Upload a multi-file selection containing valid and invalid files; verify independent per-file results and the 10-file selection ceiling.
- Exercise the access matrix for employee, team lead, project manager, administrator, project member, shared recipient, and unrelated user. Try guessed IDs and revoke a share or project membership before retrying preview/download.
- Verify files cannot be fetched through static URLs or by changing a storage key. Confirm authorized download/preview routes enforce `no-store` and `nosniff` headers and log activity.
- Exercise metadata editing, file replacement, confirmed deletion, share/revoke notifications, project/task attachments, dashboard recent documents/count, and administrator reports.
- With up to 500 authorized documents, measure list/search against their 2-second targets; measure upload/scanning and PDF/image preview against the specified 30-second/3-second targets.
- Before training rollout, execute and record the operator's restore procedure for the document directory and LocalDB metadata.

## References

- [ClamAV Windows installation and configuration](https://docs.clamav.net/manual/Installing.html)
- [ClamAV scanning and `clamdscan`](https://docs.clamav.net/manual/Usage/Scanning.html)
- [ClamAV daemon protocol](https://docs.clamav.net/manual/Usage/ClamdProtocol.html)
- [EICAR anti-malware test file](https://www.eicar.org/download-anti-malware-testfile/)
- [Azure Queue Storage trigger for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue-trigger)
- [Develop and run Azure Functions locally](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local)
- [Azurite local storage emulator](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite)
