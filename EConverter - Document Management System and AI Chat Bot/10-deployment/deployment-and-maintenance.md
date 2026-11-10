# Deployment, Operations, and Maintenance Guide

- **Purpose:** Document the verified environment requirements, configuration keys, local setup, build/test/publish commands, database initialization, encrypted file storage setup (`DCE1`), local AI runtime (Ollama / BYOK) setup, Docker Compose (`docker-compose.yml`, `docker-compose.prod.yml`) and GitHub Actions CI/CD (`.github/workflows/deploy.yml`) procedures, backup/restore, monitoring, troubleshooting, maintenance, and developer handover.
- **Scope:** `src/DocumentConverter.Web`, `src/Workers/DocumentWorker`, `src/DocumentConverter.Web/Dockerfile`, `src/Workers/DocumentWorker/Dockerfile`, `docker-compose.yml`, `docker-compose.prod.yml`, `.github/workflows/deploy.yml`, `DEPLOYMENT.md`, and runtime configuration providers.
- **Evidence status:** `Code-verified` (Dockerfiles, Compose manifests, CI/CD workflow, startup code, and `DEPLOYMENT.md`) and `Runtime-verified` (`dotnet --info`, `dotnet sln list`, and `dotnet test`). Live production infrastructure is not inspected (`Requires confirmation`).
- **Last reviewed:** 2026-11-05

---

## 1. Environment Requirements and Prerequisites

### 1.1 Hardware and OS Requirements (`Code-verified`)

| Environment | CPU | RAM | Disk Space | Notes |
|---|---|---|---|---|
| **Local Development** | 4+ cores (`x64` / `arm64`) | 8 GB minimum (16 GB recommended if running Docker + Ollama) | 10 GB free | Windows 10/11, macOS, or Linux |
| **Containerized Staging / Production (Core Stack)** | 4+ vCPU (`dc-document-worker`: 2.0 CPUs limit in `docker-compose.prod.yml`) | 8 GB+ (`dc-document-worker`: 2048 MB limit, `dc-sqlserver`: 2–4 GB) | 50 GB+ SSD (`sqlserver_data`, `documents_storage`, `rabbitmq_data`, `clamav_db` volumes) | Linux VPS/host with Docker Engine 24+ and Docker Compose V2 |
| **Optional Local LLM (Ollama)** | 8+ vCPU or NVIDIA GPU | +8 GB RAM (for 3B–8B parameter GGUF models like `llama3.2`) | +10 GB for model weights | Only required when `AI.CloudEnabled = true` with local Ollama endpoint |

### 1.2 Software Prerequisites (`Code-verified` & `Runtime-verified`)

| Prerequisite | Version | Required By | Verification Evidence |
|---|---|---|---|
| **.NET 10 SDK** | `10.0.114` (pinned in `global.json`, `rollForward: latestFeature`; verified with `10.0.403`) | Building, testing, and running `DocumentConverter.slnx` locally | `global.json`, `Directory.Build.props` (`net10.0`) |
| **SQL Server** | SQL Server 2022 (`mcr.microsoft.com/mssql/server:2022-latest`) or LocalDB / SQL Express | Primary relational store (`AppDbContext` — 25 tables across `dbo` & `identity`) | `docker-compose.yml`, `src/DocumentConverter.Web/Program.cs` |
| **LibreOffice (`soffice`)** | LibreOffice 7.x / 24.x (optional on host) | Optional CLI invocation in `OfficeDocumentProcessor`; falls back to pure-managed OpenXML / PDFsharp converters when absent | `src/DocumentConverter.Infrastructure/Processing/OfficeDocumentProcessor.cs` |
| **Tesseract OCR Data** | `eng.traineddata` in `tessdata/` | `Tesseract` (`5.2.0`) OCR in `TesseractOcrEngine`; bundled in `src/DocumentConverter.Infrastructure/tessdata` and `src/DocumentConverter.Web/tessdata` | `src/DocumentConverter.Infrastructure/Ocr/TesseractOcrEngine.cs` |
| **RabbitMQ (Optional locally)** | RabbitMQ 4 (`rabbitmq:4-management`) | Required when `RabbitMQ:Enabled=true`; falls back to SQL polling (`JobProcessorWorker` / `Worker.cs`) when disabled or unreachable | `docker-compose.yml`, `RabbitMqJobDispatcher.cs` |
| **ClamAV (Optional locally)** | `clamav/clamav:latest` (TCP port `3310`) | Required when `ClamAV:Enabled=true`; `EicarSignatureDetector` always runs in-memory regardless | `docker-compose.yml`, `ClamAvScanner.cs` |
| **Ollama / OpenAI-Compatible API (Optional)** | Ollama `0.5+` (HTTP port `11434`) | Optional neural synthesis and translation via `ExternalAiClient` when `AI.CloudEnabled = true` | `src/DocumentConverter.Infrastructure/AI/ExternalAiClient.cs` |

---

## 2. Configuration Reference (`Code-verified`)

Never commit secrets to `appsettings.json` or `.env` files in source control. Use environment variables or `.NET User Secrets` in development.

### 2.1 Application & Environment Configuration Keys

| Configuration Key (JSON Path) | Environment Variable Equivalent | Default in Code / `appsettings.json` | Purpose and Secret Sensitivity |
|---|---|---|---|
| `ConnectionStrings:DefaultConnection` | `ConnectionStrings__DefaultConnection` | Local SQL Server (`DocumentConverterDb`) | **Sensitive.** EF Core SQL Server connection string (`AppDbContext`). |
| `AutoMigrateDatabase` | `AutoMigrateDatabase` | `true` | Runs `db.Database.MigrateAsync()` and `DbSeeder.SeedAsync(...)` at `DocumentConverter.Web` startup (`Program.cs`, line 197). |
| `Storage:RootPath` | `Storage__RootPath` or `DOCUMENT_STORAGE_ROOT` | `./app_data/storage` (local) / `/data/storage` (Docker) | Root directory for sharded document blobs (`{docId[0..2]}/{docId[2..4]}/{docId:N}/v{N}.bin`). |
| `Storage:Encryption:Enabled` | `Storage__Encryption__Enabled` | `false` (dev) / `true` (`docker-compose.prod.yml`) | Enables `DCE1` AES-256-GCM per-file DEK/KEK envelope encryption in `AesGcmStorageEncryptionService`. |
| `Storage:Encryption:MasterKeyBase64` | `Storage__Encryption__MasterKeyBase64` or `STORAGE_ENCRYPTION_KEY` | Ephemeral fallback in Dev (throws in Production if missing when enabled) | **Secret.** Base64-encoded 32-byte (256-bit) Master Key Encryption Key (KEK) for `DCE1` envelope encryption. |
| `Processing:RunWorkerInWeb` | `Processing__RunWorkerInWeb` | `true` (local) / `false` (Docker) | When `true`, `DocumentConverter.Web` registers `JobProcessorWorker` in-process (`Program.cs`, line 130). Set `false` when offloading jobs to the standalone `dc-document-worker` container. |
| `RabbitMQ:Enabled` | `RabbitMQ__Enabled` | `false` (local Dev) / `true` (Docker) | Enables AMQP 0-9-1 job publishing/consumption via `RabbitMqJobDispatcher`. |
| `RabbitMQ:Host` | `RabbitMQ__Host` | `localhost` (`rabbitmq` in Docker) | RabbitMQ hostname (`RabbitMqOptions.cs`). |
| `RabbitMQ:Port` | `RabbitMQ__Port` | `5672` | RabbitMQ AMQP port. |
| `RabbitMQ:UserName` | `RabbitMQ__UserName` | `guest` (`dc_user` in Prod) | **Sensitive.** RabbitMQ username. |
| `RabbitMQ:Password` | `RabbitMQ__Password` | `guest` | **Secret.** RabbitMQ password. |
| `ClamAV:Enabled` | `ClamAV__Enabled` | `false` (local Dev) / `true` (`docker-compose.prod.yml`) | Enables fail-closed TCP `zINSTREAM\0` malware scanning on upload (`ClamAvScanner.cs`). |
| `ClamAV:Host` | `ClamAV__Host` | `localhost` (`clamav` in Docker) | ClamAV daemon hostname. |
| `ClamAV:Port` | `ClamAV__Port` | `3310` | ClamAV TCP port. |
| `ClamAV:TimeoutSeconds` | `ClamAV__TimeoutSeconds` | `15` | Socket timeout for ClamAV scan stream. |
| `Auth:UseOidc` | `Auth__UseOidc` | `false` | Enables external OpenID Connect authentication alongside ASP.NET Core Identity cookies. |
| `Seed:AdminPassword` | `SEED_ADMIN_PASSWORD` | `Admin123!@#` (Development only) | **Secret.** Initial password for seeded administrator (`admin@documentconverter.local`). |
| `Seed:DemoPassword` | `SEED_DEMO_PASSWORD` | `Demo123!@#` (Development only) | **Secret.** Initial password for seeded standard user (`demo@documentconverter.local`). |

### 2.2 Database-Backed Runtime Settings (`dbo.SystemSettings`)

Seeded by `DbSeeder.cs` (`src/DocumentConverter.Infrastructure/Persistence/DbSeeder.cs`, lines 187–203) and editable at runtime via `/admin` (`AdminService.UpdateSettingAsync`):

| Setting Key | Seeded Default | Description |
|---|---|---|
| `Upload.MaxFileSizeMb` | `100` | Maximum upload file size in megabytes. |
| `Storage.TotalQuotaGb` | `50` | Organizational storage quota indicator in gigabytes. |
| `Retention.Days` | `90` | Configured retention window in days (note: automatic expiration worker is not yet implemented). |
| `Processing.MaxConcurrentJobs` | `4` | Concurrency setting for background conversion jobs. |
| `Security.RequireMalwareScan` | `true` | Policy metadata flag for malware scanning. |
| `AI.CloudEnabled` | `false` | **Zero-Egress Switch.** When `false` (default), all RAG and Copilot operations execute 100% locally without calling external LLM APIs. |
| `AI.Provider` | `OpenAI` | Selected provider (`OpenAI`, `Ollama`, `AzureOpenAI`, `Groq`, `Custom`). |
| `AI.Endpoint` | `https://api.openai.com/v1` | Base URL for OpenAI-compatible `/chat/completions` API (e.g., `http://localhost:11434/v1` for local Ollama). |
| `AI.Model` | `gpt-4o-mini` | Target LLM model identifier (e.g., `llama3.2` for Ollama). |
| `AI.ApiKey` | *(empty)* | **Secret.** API key for system-wide external AI provider (if not using per-session BYOK). |
| `AI.Temperature` | `0.2` | Sampling temperature for neural synthesis (`0.0`–`1.0`). |

---

## 3. Local Development Setup, Build, and Test

### 3.1 Generate a 256-bit Storage Master Key (`STORAGE_ENCRYPTION_KEY`)

```powershell
# PowerShell (Windows / Cross-platform)
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$env:STORAGE_ENCRYPTION_KEY = [Convert]::ToBase64String($bytes)
```

Or on Linux/macOS:
```bash
export STORAGE_ENCRYPTION_KEY="$(openssl rand -base64 32)"
```

### 3.2 Restore, Build, and Test Commands (`Runtime-verified`)

```powershell
# 1. Verify .NET 10 SDK (requires 10.0.114+ per global.json; verified on 10.0.403)
dotnet --info

# 2. List all 11 solution projects
dotnet sln DocumentConverter.slnx list

# 3. Restore NuGet packages (centrally managed via Directory.Packages.props)
dotnet restore DocumentConverter.slnx

# 4. Build solution in Release configuration (TreatWarningsAsErrors=true)
dotnet build DocumentConverter.slnx --configuration Release --no-restore

# 5. Execute all 130 xUnit v3 tests across Domain, Architecture, and Application test projects
dotnet test DocumentConverter.slnx --configuration Release --no-build -p:IsTestingPlatformApplication=false -p:TestingPlatformDotnetTestSupport=false --verbosity minimal
```

### 3.3 Database Initialization and Seeding (`Code-verified`)

When `DocumentConverter.Web` starts (`src/DocumentConverter.Web/Program.cs`, lines 197–217):
1. If `app.Environment.IsDevelopment()` or `AutoMigrateDatabase = true`, it resolves `AppDbContext` and runs `await db.Database.MigrateAsync()` to apply all 5 EF Core migrations (`InitialCreate`, `AddEnterpriseSecurityAndFeatures`, `AddCopilotChatAndMemoryPersistence`, `AddDocumentIntelligenceVectorPersistence`, `AddMultilingualVectorChunkFields`).
2. It invokes `await DbSeeder.SeedAsync(db, userManager, roleManager, logger, isDevelopment)`, which idempotently seeds:
   - **3 Departments:** `ENGINEERING` (`Engineering & IT`), `LEGAL` (`Legal & Compliance`), `OPERATIONS` (`Operations & Finance`).
   - **2 Roles:** `Administrator` and `User` (in both `dbo.Roles` and `identity.Roles`).
   - **2 Users:**
     - `admin@documentconverter.local` (`Administrator`, Engineering & IT) — password from `SEED_ADMIN_PASSWORD` (or `Admin123!@#` in Development).
     - `demo@documentconverter.local` (`User`, Operations & Finance) — password from `SEED_DEMO_PASSWORD` (or `Demo123!@#` in Development).
   - **17 Converter Definitions** across `processor.pdf`, `processor.office`, `processor.image`, `processor.pdf.compress`, `processor.image.compress`, and `processor.ocr`.
   - **11 `SystemSettings` rows** (listed in Section 2.2).

---

## 4. Storage and Local AI Runtime Setup

### 4.1 Sharded Encrypted File Storage (`FileSystemDocumentStorage` & `AesGcmStorageEncryptionService`)

- **Sharded Directory Layout (`src/DocumentConverter.Infrastructure/Storage/FileSystemDocumentStorage.cs`):**
  - Stores each document version as `{RootPath}/{docId[0..2]}/{docId[2..4]}/{docId:N}/v{versionNumber}.bin`.
  - User-supplied filenames are never used in filesystem paths, eliminating path-traversal and OS filename-encoding vulnerabilities.
- **Envelope Encryption (`src/DocumentConverter.Infrastructure/Security/StorageEncryption/AesGcmStorageEncryptionService.cs`):**
  - When `Storage:Encryption:Enabled = true`, every file is encrypted with a fresh 256-bit Data Encryption Key (DEK) using AES-256-GCM, and the DEK is wrapped by the 256-bit Master Key (`STORAGE_ENCRYPTION_KEY`) in a 92-byte `DCE1` header.
  - Unencrypted legacy files (not starting with magic bytes `DCE1`) are read transparently via `PassThroughStream` for backward compatibility.

### 4.2 Local AI Runtime Setup (Zero-Egress Default + Optional Ollama)

By default (`AI.CloudEnabled = false`), `DocumentIntelligenceService` executes **100% in-process** using deterministic 256-D MurmurHash3 + BM25 + concept embeddings, `HnswVectorIndex`, offline bilingual dictionary translation (`MultilingualChunkProcessor`), and extractive sentence synthesis—requiring **zero external AI services**.

To enable optional local LLM synthesis via [Ollama](https://ollama.com/):
1. Start Ollama locally (`http://localhost:11434`) and pull a model:
   ```bash
   ollama pull llama3.2
   ```
2. Log in as `Administrator` (`admin@documentconverter.local`), navigate to `/admin` (or configure via the Copilot BYOK settings dialog), and set:
   - `AI.CloudEnabled` = `true`
   - `AI.Provider` = `Ollama`
   - `AI.Endpoint` = `http://localhost:11434/v1` (or `http://host.docker.internal:11434/v1` from Docker)
   - `AI.Model` = `llama3.2`

---

## 5. Containerized Deployment and CI/CD Pipeline

### 5.1 Multi-Stage Dockerfiles (`Code-verified`)

1. **`src/DocumentConverter.Web/Dockerfile`:**
   - Builds and publishes `DocumentConverter.Web.csproj` on `mcr.microsoft.com/dotnet/sdk:10.0`.
   - Runs on `mcr.microsoft.com/dotnet/aspnet:10.0` exposing ports `8080` and `8081` under non-root `USER $APP_UID`.
2. **`src/Workers/DocumentWorker/Dockerfile`:**
   - Builds and publishes `DocumentWorker.csproj` on `mcr.microsoft.com/dotnet/sdk:10.0`.
   - Runs on `mcr.microsoft.com/dotnet/runtime:10.0` under non-root `USER $APP_UID`.

### 5.2 Docker Compose Topology (`docker-compose.yml` & `docker-compose.prod.yml`)

The container stack orchestrates 5 services across two bridge networks (`dc-public` and `dc-internal` where `dc-internal` sets `internal: true` to enforce **zero external internet egress** for backend containers):
1. **`web` (`dc-web`):** Joined to `dc-public` and `dc-internal`; maps host port `${WEB_PORT:-5000}:8080`; mounts `documents_storage:/data/storage`; healthcheck polls `curl -f http://localhost:8080/health`.
2. **`document-worker` (`dc-document-worker`):** Joined **only** to `dc-internal` (`no-new-privileges:true`, limited to `2.0` CPUs and `2048M` RAM); mounts `documents_storage:/data/storage`.
3. **`sqlserver` (`dc-sqlserver`):** `mcr.microsoft.com/mssql/server:2022-latest` on `dc-internal`; mounts `sqlserver_data:/var/opt/mssql`.
4. **`rabbitmq` (`dc-rabbitmq`):** `rabbitmq:4-management` on `dc-internal`; mounts `rabbitmq_data:/var/lib/rabbitmq`.
5. **`clamav` (`dc-clamav`):** `clamav/clamav:latest` on `dc-internal`; mounts `clamav_db:/var/lib/clamav`.

**Production Deployment Commands (on VPS in `/opt/document-converter`):**
```bash
# Pull pre-built images from GHCR and start the production stack
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d --remove-orphans
docker compose -f docker-compose.prod.yml ps
```

### 5.3 GitHub Actions CI/CD Pipeline (`.github/workflows/deploy.yml`)

Triggered on `push` to `main` or `workflow_dispatch`:
1. **Job 1 (`test` — Quality Gate):** Checks out code on `ubuntu-latest`, installs .NET `10.0.x`, and runs `dotnet restore`, `dotnet build --configuration Release`, and `dotnet test --configuration Release`.
2. **Job 2 (`build-and-push` — GHCR Image Publishing):** Logs in to `ghcr.io` using `GITHUB_TOKEN` and builds/pushes `ghcr.io/<owner>/dc-web:{latest,sha}` and `ghcr.io/<owner>/dc-document-worker:{latest,sha}` with GitHub Actions cache (`type=gha,mode=max`).
3. **Job 3 (`deploy` — Automated SSH Rolling Deployment):** If `VPS_HOST` is configured, copies `docker-compose.prod.yml` to `${DEPLOY_PATH:-/opt/document-converter}` via `appleboy/scp-action@v0.1.7`, then SSHs in via `appleboy/ssh-action@v1.2.0` to validate `.env`, pull fresh images, run `docker compose -f docker-compose.prod.yml up -d --remove-orphans`, and prune unused images.

---

## 6. Backup, Restore, Monitoring, and Troubleshooting

### 6.1 Backup and Restore Procedures (`Recommendation — requires environment-owner approval`)

A complete backup must capture **three synchronized artifacts**:
1. **SQL Server Database (`DocumentConverterDb` on `sqlserver_data`):** Contains all metadata, versions, audit logs, and RAG vector chunks (`dbo.DocumentVectorChunks`).
2. **Document Storage Volume (`documents_storage` at `/data/storage`):** Contains all sharded `v{N}.bin` document binaries.
3. **Master Encryption Key (`STORAGE_ENCRYPTION_KEY` in `/opt/document-converter/.env`):** **Critical:** Without this 32-byte Base64 key, `DCE1`-encrypted blobs on `documents_storage` cannot be decrypted (`AesGcmStorageEncryptionService.cs`).

**Recommended Backup Commands:**
```bash
# 1. Back up SQL Server database inside dc-sqlserver container
docker exec dc-sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "$SQLSERVER_SA_PASSWORD" -C \
  -Q "BACKUP DATABASE [DocumentConverterDb] TO DISK = N'/var/opt/mssql/backup/DocumentConverterDb.bak' WITH INIT, COMPRESSION"

# 2. Archive the documents_storage volume
docker run --rm -v documents_storage:/data/storage:ro -v $(pwd)/backups:/backup \
  alpine tar -czf /backup/documents_storage_$(date +%F).tar.gz -C /data/storage .
```

### 6.2 Monitoring and Observability (`Code-verified`)

1. **Health Check Endpoint (`GET /health`):**
   - Registered in `src/DocumentConverter.Web/Program.cs` (lines 178–179, 262) via `.AddHealthChecks().AddDbContextCheck<AppDbContext>("database")`.
   - Polled every 15 seconds by Docker Compose (`docker-compose.prod.yml`, lines 34–39).
2. **OpenTelemetry Tracing and Metrics:**
   - Registered in `src/DocumentConverter.Web/Program.cs` (lines 182–192) with `AddAspNetCoreInstrumentation()`, `AddHttpClientInstrumentation()`, and `AddRuntimeInstrumentation()`.

### 6.3 Troubleshooting Guide

| Symptom | Probable Cause | Resolution |
|---|---|---|
| `InvalidOperationException` on startup when `Storage:Encryption:Enabled=true` in Production | `STORAGE_ENCRYPTION_KEY` is missing or not a valid 32-byte Base64 string | Generate a 32-byte Base64 key (`openssl rand -base64 32`) and set `STORAGE_ENCRYPTION_KEY` in `.env`. |
| `CryptographicException` when downloading or indexing an existing document | `STORAGE_ENCRYPTION_KEY` was changed after the file was encrypted with `DCE1` | Restore the original `STORAGE_ENCRYPTION_KEY` that wrapped the file's DEK. |
| Uploads fail with malware scanner error when `ClamAV:Enabled=true` | `ClamAvScanner` is fail-closed when enabled (`ClamAvScanner.cs`), and `dc-clamav` is still loading virus definitions or unreachable | Check `docker compose logs clamav` and wait for `clamd` port `3310` to accept connections, or set `ClamAV__Enabled=false` in local development. |
| `dotnet test` fails on .NET 10 SDK with Microsoft.Testing.Platform runner mismatch | `xunit.v3` + `Microsoft.NET.Test.Sdk` requires VSTest properties on .NET 10 CLI | Pass `-p:IsTestingPlatformApplication=false -p:TestingPlatformDotnetTestSupport=false` to `dotnet test`. |

---

## 7. Developer Handover: Known Issues and Technical Debt (`Code-verified`)

| ID | Category | Finding / Limitation | Source Location | Recommended Remediation |
|---|---|---|---|---|
| `TD-01` | **Data Governance / Retention (Medium)** | **No Automated Retention Expiration Worker:** `Retention.Days = 90` is seeded in `dbo.SystemSettings` and `dbo.Documents` has an `ExpiresAt` column, but no `BackgroundService` queries or purges expired documents automatically. | `src/DocumentConverter.Infrastructure/Persistence/DbSeeder.cs` (line 195), `src/DocumentConverter.Domain/Entities/Document.cs` | Implement a `RetentionCleanupWorker : BackgroundService` that periodically soft-deletes/purges documents past `ExpiresAt` or `DeletedAt + Retention.Days`. |
| `TD-02` | **RAG Lifecycle / Governance (Medium)** | **RAG Vectors Retained After Soft-Delete/Purge & Single-Doc Status Check:** `DeleteAsync` and `PurgeDocumentAsync` update `dbo.Documents.Status` (`Deleted` / `Purged`) without deleting rows in `dbo.DocumentVectorChunks` / `dbo.DocumentIntelligenceIndices` or evicting `DocumentIndexCache`; single-doc RAG methods check ownership (`EnsureUserHasDocumentAccessAsync`) without filtering by `Status == Available`. | `src/DocumentConverter.Infrastructure/Documents/DocumentService.cs` (lines 430–525), `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 1036–1055) | Delete `DocumentVectorChunks` and `DocumentIntelligenceIndices` rows on purge, evict `DocumentIndexCache`, and require `d.Status == DocumentStatus.Available` inside `EnsureUserHasDocumentAccessAsync`. |
| `TD-03` | **Concurrency / Testing (Medium)** | **Static In-Memory RAG Cache & HNSW Index:** `DocumentIndexCache` and `GlobalHnswIndex` are `private static readonly` fields on `DocumentIntelligenceService`, causing cross-test state pollution in `DocumentIntelligenceTests` (`TC-RAG-02` failure when `_dbContext == null`). | `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 37–42) | Refactor `DocumentIndexCache` and `GlobalHnswIndex` into an injectable singleton service with a `Clear()` reset hook for test isolation. |
| `TD-04` | **Audit Security (Low)** | **Unmasked Setting Values in Audit Metadata & Missing Login Audit Events:** `AdminService.UpdateSettingAsync` logs `{ key, value }` in `dbo.AuditEvents.Metadata` without redacting `AI.ApiKey`, and `AccountEndpoints.cs` does not write login success/failure events to `dbo.AuditEvents`. | `src/DocumentConverter.Infrastructure/Admin/AdminService.cs` (lines 133–138), `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs` | Mask sensitive setting keys (`AI.ApiKey`) before calling `_audit.RecordAsync` and add `User.LoginSucceeded` / `User.LoginFailed` audit entries in `AccountEndpoints.cs`. |

---

## 8. Evidence Traceability

**Evidence**
- `src/DocumentConverter.Web/Dockerfile`, `src/Workers/DocumentWorker/Dockerfile`, `docker-compose.yml`, `docker-compose.prod.yml`, `.github/workflows/deploy.yml`, `DEPLOYMENT.md` — container build, network isolation (`dc-internal`), Compose stack, and CI/CD workflow.
- `src/DocumentConverter.Web/Program.cs` and `src/Workers/DocumentWorker/Program.cs` — startup migration/seeding, authentication, rate limiting, OpenTelemetry, `/health`, and DI bindings.

**Status:** `Code-verified` and `Runtime-verified` (build and test).
