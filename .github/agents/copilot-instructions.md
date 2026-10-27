# ContosoDashboard Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-10-27

## Active Technologies

- C# 12 / .NET 8 + ASP.NET Core Blazor Server, EF Core SQL Server 8, .NET isolated Azure Functions v4 Queue Storage trigger, Azurite for offline queue emulation, local ClamAV `clamd`/`clamdscan`, xUnit and EF Core SQLite (001-document-upload-management)

## Project Structure

```text
ContosoDashboard/
├── Controllers/
├── Data/
├── Models/
├── Pages/
├── Services/
└── Shared/
ContosoDashboard.DocumentProcessing/
ContosoDashboard.DocumentScan.Functions/
ContosoDashboard.Tests/
specs/001-document-upload-management/
```

## Commands

```powershell
dotnet restore ContosoDashboard/ContosoDashboard.csproj
dotnet build ContosoDashboard/ContosoDashboard.csproj
dotnet build ContosoDashboard.DocumentScan.Functions/ContosoDashboard.DocumentScan.Functions.csproj
dotnet test ContosoDashboard.Tests/ContosoDashboard.Tests.csproj
dotnet run --project ContosoDashboard/ContosoDashboard.csproj
# Start Azurite locally and run `func start` from ContosoDashboard.DocumentScan.Functions.
```

## Code Style

C# 12 / .NET 8: Follow standard conventions

## Recent Changes

- 001-document-upload-management: Added C# 12 / .NET 8 + ASP.NET Core Blazor Server, EF Core SQL Server 8, existing cookie authentication; locally installed ClamAV `clamd`/`clamdscan`; xUnit and EF Core SQLite for the new test project

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
