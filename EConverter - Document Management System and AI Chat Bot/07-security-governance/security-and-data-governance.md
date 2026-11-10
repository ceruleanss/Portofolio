# Security and Data Governance

**Purpose:** Provide an evidence-based analysis of the platform's implemented security controls and data governance mechanisms across authentication, RBAC, resource authorization, audit logging, versioning, retention/deletion, file storage encryption, temporary files, backup considerations, and RAG-derived data lifecycle—clearly separating `Code-verified` controls from recommended improvements.  
**Scope:** All security-critical components across `DocumentConverter.Security`, `DocumentConverter.Domain`, `DocumentConverter.Infrastructure`, `DocumentConverter.Web`, `DocumentWorker`, and Docker Compose deployment configurations.  
**Evidence status:** `Code-verified` & `Runtime-verified` (`DocumentSecurityAndIdorTests`, `Milestone8SecurityHardeningTests`, `PdfDigitalSignatureTests`, `DocumentIntelligenceTests`).  
**Last reviewed:** 2026-11-05

> [!IMPORTANT]
> **Compliance Disclaimer:** In accordance with Mandatory Analysis Rule 4.H, this document **does not claim ISO/IEC 27001, SOC 2, HIPAA, or GDPR certification or regulatory compliance**. Achieving formal regulatory compliance requires organizational policies, external audits, key management procedures, and physical/infrastructure controls beyond application source code.

---

## 1. Authentication

### 1.1 Implemented Controls (`Code-verified`)
- **ASP.NET Core Identity Local Store (`Program.cs`, lines 46–61):** Passwords hashed in `identity.Users` (`AppIdentityUser`); requires `>= 8` characters with uppercase, lowercase, digit, and special symbol; enforces account lockout for `15 minutes` after `5` failed login attempts (`lockoutOnFailure: true` in `AccountEndpoints.cs`, line 41).
- **Session Cookie Hardening (`Program.cs`, lines 63–81):** Cookie `.DocumentConverter.Auth` is configured with `HttpOnly = true`, `SameSite = SameSiteMode.Lax`, `ExpireTimeSpan = 8 hours`, and `SlidingExpiration = true`.
- **Optional OpenID Connect Federation (`Program.cs`, lines 84–99):** Supports external OIDC Identity Providers (`Authority`, `ClientId`, `ClientSecret`, `ResponseType = "code"`, `SaveTokens = true`) when `Auth:UseOidc = true`.
- **Brute-Force Rate Limiting (`Program.cs`, lines 110–116):** Fixed-window limiter `auth` restricts `/api/account/login` and `/api/account/external-login` to `10` requests per minute per IP address (`HTTP 429`).

---

## 2. Role-Based Access Control (RBAC) and Authorization Enforcement

### 2.1 Implemented Controls (`Code-verified` & `Runtime-verified`)
- **Decoupled Internal Identity Claim (`AppClaimsTransformation.cs`):** Maps authenticated users by normalized email to `dbo.Users.Id` and attaches the `dc:uid` claim (`HttpContextUserContext.UserIdClaimType`) plus `ClaimTypes.Role` claims from `dbo.UserRoles`.
- **Role-Based Policy (`AdminOnly`):** Defined in `Program.cs` (lines 101–104) requiring `RoleNames.Administrator`. Enforced at both the Razor page route level (`@attribute [Authorize(Policy = "AdminOnly")]` on all `/admin/*` pages) and inside `AdminService.EnsureAdmin()`.
- **Strict Resource-Level Ownership Isolation (Anti-IDOR):**
  - **Documents & Folders (`DocumentService.cs`):** Every query filters by `OwnerUserId == userId`. Non-owners attempting to read, download, rename, move, add a version, rollback, split, merge, soft-delete, restore, or purge another user's document or folder receive `KeyNotFoundException` (`404 Not Found`), preventing resource existence enumeration (`Runtime-verified` in `DocumentSecurityAndIdorTests`).
  - **Processing & Batch Jobs (`ProcessingService.cs`):** `CreateJobAsync`, `CreateBatchAsync`, `GetJobAsync`, `ListJobsAsync`, and `CancelJobAsync` verify `OwnerUserId == userId` (`Runtime-verified` in `BatchProcessingTests.CreateBatchAsync_PreventsIdor_WhenAccessingOtherUsersDocuments`).
  - **RAG & Copilot (`DocumentIntelligenceService.cs`):**
    - Single-document RAG (`AnalyzeDocumentAsync`, `AskQuestionAsync`, `SemanticSearchAsync`, `EnsureDocumentIndexedAsync`, `ReindexDocumentAsync`) invokes `EnsureUserHasDocumentAccessAsync(documentId)`, verifying `dbo.Documents.AnyAsync(d => d.Id == documentId && d.OwnerUserId == currentUserId.Value)` and throwing `UnauthorizedAccessException` (`403 Forbidden`) on mismatch (`Runtime-verified` in `DocumentIntelligenceTests.DocumentQueriesAndIntelligence_ShouldEnforceStrictUserIsolation`).
    - Repository-wide RAG (`SemanticSearchRepositoryAsync`) executes a Stage 1 SQL filter `.Where(d => d.Status == DocumentStatus.Available && d.OwnerUserId == currentUserId.Value)` before querying `HnswVectorIndex` with the allowed `candidateDocumentIds` (`HashSet<Guid>`).
    - Persistent chat sessions (`dbo.CopilotChatSessions`) and long-term memories (`dbo.CopilotMemoryItems`) filter by `UserId == currentUserId` (`Runtime-verified` in `DocumentIntelligenceTests.ChatSessionsAndMemory_WithDatabaseContext_ShouldPersistEntitiesToDbAndSupportUserScoping`).

---

## 3. Audit Logging

- **Implemented (`AuditService.cs`, `dbo.AuditEvents`):** Records 32 distinct event types across document upload, malware rejection, download, version addition/rejection/rollback, split/merge, soft-delete/restore/purge, folder lifecycle, job submission/cancellation, and administrative changes, capturing `UserId` (`dc:uid`), `IpAddress`, `ResourceType`, `ResourceId`, `Metadata` (JSON), and `OccurredAt` (UTC).
- **Verified Gaps:**
  1. `AccountEndpoints.cs` does not record `User.LoginSucceeded` or `User.LoginFailed` events in `dbo.AuditEvents`.
  2. `AdminService.UpdateSettingAsync` (lines 133–138) logs `new { key, value }` in `dbo.AuditEvents.Metadata` without masking sensitive setting keys such as `AI.ApiKey`.

---

## 4. Document Versioning and Cryptographic Integrity

- **Immutable Version History (`dbo.DocumentVersions`):** Modifications, PDF edits (`commit-version`), digital signatures, and rollbacks never overwrite existing `.bin` files; each creates a new `v{N+1}.bin` blob with its own 64-character lowercase `SHA-256` hex digest (`Sha256Hex`).
- **PAdES Digital Signatures (`PdfDigitalSignatureService.cs`):** Implements ISO 32000-1 / PAdES detached PKCS#7 (`/SubFilter /adbe.pkcs7.detached`, SHA-256 digest) signatures via PDF incremental updates, preserving existing signature byte-ranges in multi-signer workflows and detecting any post-signing byte tampering via `SignedCms.CheckSignature(verifySignatureOnly: true)`.

---

## 5. File Storage Security and Temporary Files

### 5.1 File Ingestion & At-Rest Storage Security (`Code-verified`)
1. **Untrusted Filename Isolation (`FileNameSanitizer.cs` & `FileSystemDocumentStorage.cs`):** User-supplied filenames are sanitized for metadata display and **never** used in filesystem paths. Physical blobs are stored at `{RootPath}/{docId[0..2]}/{docId[2..4]}/{docId:N}/v{versionNumber}.bin` with `ResolveSafePath` path-traversal validation.
2. **Magic-Byte & Archive Bomb Defense (`FileSignatureDetector.cs` & `ZipBombGuard.cs`):** Rejects spoofed extensions, ZIP path traversal, VBA macro binaries (`vbaProject.bin`), and decompression bombs (`> 500 MB`, ratio `> 50:1`, `> 5,000` entries).
3. **Fail-Closed Malware Scanning (`EicarSignatureDetector.cs` & `ClamAvScanner.cs`):** Streams uploads over TCP to ClamAV (`zINSTREAM\0`) without writing unvalidated files to storage, rejecting uploads if malware is found or if `clamd` is unreachable when `ClamAV:Enabled = true`.
4. **AES-256-GCM Envelope Encryption (`AesGcmStorageEncryptionService.cs`):** When `Storage:Encryption:Enabled = true`, encrypts every blob on disk with a random 256-bit per-file DEK wrapped by the 256-bit Master KEK (`DCE1` 92-byte header + ciphertext), providing authenticated encryption that throws `CryptographicException` if any byte on disk is altered.

### 5.2 Temporary File Lifecycle (`Code-verified`)
| Component | Temporary Path Pattern | Cleanup Mechanism | Verification Status |
|---|---|---|---|
| `FileSystemDocumentStorage.SaveAsync` | `{fullPath}.{Guid:N}.tmp` | Atomic `File.Move(tempPath, fullPath, overwrite: true)` or `File.Delete(tempPath)` in `catch` block (`FileSystemDocumentStorage.cs` L63–90). | `Code-verified` |
| `JobProcessorWorker` & `DocumentWorker` | `{TempPath}/dc-jobs/{jobId:N}/` | `Directory.Delete(tempRoot, recursive: true)` in `finally` block (`JobProcessorWorker.cs` L278–283; `Worker.cs` L257–262). | `Code-verified` |
| `DocumentIntelligenceService` (`MarkItDown`) | `{TempPath}/DocConverter_MarkItDown/{Guid:N}{ext}` | `File.Delete(tempFilePath)` in `finally` block (`DocumentIntelligenceService.cs` L1762–1769). | `Code-verified` |
| `CopilotDossierExporter.ExportDossierAsync` | `{TempPath}/DocConverter_CopilotExports/{Guid:N}.md` & `.pdf` | `File.Delete(tempMdPath)` and `File.Delete(tempPdfPath)` in `finally` block (`CopilotDossierExporter.cs` L76–80). | `Code-verified` |
| `OfficeDocumentProcessor` (LibreOffice) | `{OutputDirectory}/lo_profile_{Guid:N}/` | `Directory.Delete(userProfileDir, recursive: true)` in `finally` block (`OfficeDocumentProcessor.cs` L238–241). | `Code-verified` |

---

## 6. Data Lifecycle, Retention, Deletion, and RAG-Derived Data Lifecycle

### 6.1 Primary & Derived Data Stores Matrix

| Data Artifact | Storage Location | Encryption / Protection | Behavior on Soft-Delete (`DeleteAsync`) | Behavior on Permanent Purge (`PurgeDocumentAsync`) |
|---|---|---|---|---|
| **Document Binary (`v{N}.bin`)** | Sharded filesystem (`documents_storage` volume) | Optional `DCE1` AES-256-GCM envelope encryption | Retained on disk; download blocked (`Status = Deleted`). | **Physically deleted from disk** (`_storage.DeleteAsync` for all versions). |
| **Document Metadata (`dbo.Documents`)** | SQL Server (`dbo.Documents`) | SQL Server storage | `Status` set to `'Deleted'`, `DeletedAt` populated. | Row retained as tombstone with `Status = 'Purged'`. |
| **Version Metadata (`dbo.DocumentVersions`)** | SQL Server (`dbo.DocumentVersions`) | SQL Server storage | Retained. | Metadata rows retained (since parent `dbo.Documents` row is not physically deleted). |
| **Extracted Insights & Entities (`dbo.DocumentIntelligenceIndices`)** | SQL Server (`dbo.DocumentIntelligenceIndices`) | SQL Server storage | Retained in DB; excluded from repository-wide Copilot search. | **Retained in DB** (`Code-verified` gap: `ON DELETE CASCADE` does not fire because `dbo.Documents` row is marked `Purged` rather than deleted). |
| **Vector Chunks & Text (`dbo.DocumentVectorChunks`)** | SQL Server (`dbo.DocumentVectorChunks`) | Prompt-injection sanitized; SQL Server storage | Retained in DB; excluded from `SemanticSearchRepositoryAsync` via `Status == Available` filter. | **Retained in DB** (`Code-verified` gap: not deleted by `PurgeDocumentAsync`). |
| **In-Memory Vector Graph (`GlobalHnswIndex` & `DocumentIndexCache`)** | Process RAM (`DocumentIntelligenceService`) | Process memory isolation | Retained in RAM; filtered out of `SemanticSearchRepositoryAsync` by Stage 1 SQL `candidateDocumentIds`. | **Retained in RAM** until process restart (`Code-verified` gap: no cache eviction hook in `DeleteAsync` / `PurgeDocumentAsync`). |
| **Audit Records (`dbo.AuditEvents`)** | SQL Server (`dbo.AuditEvents`) | Append-only | Appends `Document.Deleted`. | Appends `Document.Purged`. |

---

## 7. Backup Considerations

- **Consistent Multi-Store Backup Requirement:** Because document metadata, versions, and RAG vectors live in SQL Server (`sqlserver_data` volume) while encrypted document binaries live on the filesystem (`documents_storage` volume) and are encrypted with `STORAGE_ENCRYPTION_KEY`, **backups must cover all three artifacts**:
  1. SQL Server database (`DocumentConverterDb`).
  2. Shared filesystem storage volume (`documents_storage` / `app_data/storage`).
  3. The Master Key (`STORAGE_ENCRYPTION_KEY` in `/opt/document-converter/.env` or enterprise secret vault). **Warning:** Losing `STORAGE_ENCRYPTION_KEY` renders all `DCE1`-encrypted `.bin` files permanently unrecoverable (`AesGcmStorageEncryptionService.cs`).
- No automated backup scripts exist in the repository (`Not implemented in repository`); see [Deployment and Maintenance](../10-deployment/deployment-and-maintenance.md) for recommended backup procedures.

---

## 8. Implemented Controls vs. Recommended Improvements

| Area | Implemented Control (`Code-verified`) | Recommended Improvement (`Recommendation`) |
|---|---|---|
| **RAG Derived-Data Cleanup on Purge/Delete** | `SemanticSearchRepositoryAsync` filters by `Status == DocumentStatus.Available` in Stage 1 SQL. | In `DocumentService.DeleteAsync` and `PurgeDocumentAsync`, explicitly delete `dbo.DocumentVectorChunks` and `dbo.DocumentIntelligenceIndices` for `documentId`, evict `DocumentIndexCache`, and enforce `Status == DocumentStatus.Available` inside `EnsureUserHasDocumentAccessAsync`. |
| **RAG Re-Indexing on New Version** | Users can manually trigger `POST /api/ai/documents/{id}/reindex`. | Inject `IDocumentIndexingChannel` into `DocumentService.AddVersionAsync` and `RollbackVersionAsync` to automatically re-queue updated documents. |
| **Automated Retention Worker** | `Retention.Days` (`90`) and `Document.ExpiresAt` column exist in schema/seed data. | Implement a `RetentionCleanupWorker : BackgroundService` that periodically soft-deletes or purges documents where `ExpiresAt <= DateTime.UtcNow`. |
| **Audit Log Secret Masking** | `AdminService.UpdateSettingAsync` records `Admin.SettingUpdated` in `dbo.AuditEvents`. | Redact `value` to `"***REDACTED***"` in `AdminService.UpdateSettingAsync` whenever `key` contains `"Key"`, `"Secret"`, `"Password"`, or `"Token"`. |
| **Master Key Rotation** | `AesGcmStorageEncryptionService` uses a single 256-bit Master Key (KEK) to wrap per-file DEKs (`DCE1`). | Because envelope encryption already uses per-file DEKs, add a key-version identifier (`DCE2` or key ID byte) to support re-wrapping 32-byte DEKs during KEK rotation without re-encrypting multi-gigabyte file payloads. |
| **ASP.NET Core DataProtection Key Persistence** | Containerized `dc-web` uses default ephemeral in-container DataProtection key ring for `.DocumentConverter.Auth` cookies. | Persist ASP.NET Core DataProtection keys to a mounted volume or SQL Server (`PersistKeysToDbContext`) so active user sessions survive container recreations. |

---

## 9. Related Documentation

- [System Architecture](../03-system-design/system-architecture.md)
- [Module 01: Authentication and Authorization](../04-modules/01-authentication-and-authorization.md)
- [Module 03: Document Upload and Storage](../04-modules/03-document-upload-and-storage.md)
- [Module 07: Audit Logging](../04-modules/07-audit-logging.md)
- [Module 08: Retention and Deletion](../04-modules/08-retention-and-deletion.md)
- [RAG Technical Documentation](../08-rag/rag-technical-documentation.md)

