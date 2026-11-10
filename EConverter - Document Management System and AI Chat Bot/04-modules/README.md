# Module Documentation Index

**Purpose:** Provide a structured index and cross-module matrix of all 10 functional modules implemented in `DocumentConverter.slnx`.  
**Scope:** End-to-end vertical-slice documentation covering purpose, features, business rules, inputs/outputs, processing flows, classes/methods, database dependencies, API endpoints, error handling, security considerations, and PlantUML diagrams for each module.  
**Evidence status:** `Code-verified` & `Runtime-verified`.  
**Last reviewed:** 2026-11-05

---

## 1. Module Catalog

| # | Module Document | Primary Services & Entry Points | Owned / Accessed Database Tables | Verification Status |
|---|---|---|---|---|
| 01 | [Authentication and Authorization](01-authentication-and-authorization.md) | `AccountEndpoints`, `AppClaimsTransformation`, `HttpContextUserContext`, `Login.razor` | `identity.*`, `dbo.Users`, `dbo.Roles`, `dbo.UserRoles` | `Code-verified` |
| 02 | [User and Role Management](02-user-and-role-management.md) | `AdminService`, `UsersAndDepartments.razor`, `CreateUserDialog.razor` | `dbo.Users`, `dbo.Roles`, `dbo.UserRoles`, `dbo.Departments`, `identity.Users`, `identity.UserRoles`, `dbo.AuditEvents` | `Code-verified` |
| 03 | [Document Upload and Storage](03-document-upload-and-storage.md) | `DocumentService.UploadAsync`, `FileSystemDocumentStorage`, `AesGcmStorageEncryptionService`, `ClamAvScanner`, `FileSignatureDetector`, `ZipBombGuard`, `EicarSignatureDetector`, `CdpDownloadWatcher` | `dbo.Documents`, `dbo.DocumentVersions`, `dbo.AuditEvents` | `Code-verified` & `Runtime-verified` |
| 04 | [Document Conversion, OCR, and Compression](04-document-conversion.md) | `ProcessingService`, `JobProcessorWorker`, `DocumentWorker/Worker`, `RabbitMqJobDispatcher`, 6 `IDocumentProcessor` engines, `PdfCapabilityAnalyzer`, `CompressionEstimatorService` | `dbo.Converters`, `dbo.ProcessingJobs`, `dbo.BatchJobs`, `dbo.ProcessingAttempts`, `dbo.ProcessingJobInputs`, `dbo.ProcessingJobOutputs` | `Code-verified` & `Runtime-verified` |
| 05 | [Document Management, PDF Editor, and Digital Signatures](05-document-management-and-editor.md) | `DocumentService`, `PdfEngine`, `PdfDigitalSignatureService`, `DocumentEndpoints`, `EditorEndpoints`, `Documents.razor`, `PdfEditor.razor`, `Tools.razor` | `dbo.Documents`, `dbo.Folders`, `dbo.DocumentVersions`, `dbo.AuditEvents` | `Code-verified` & `Runtime-verified` |
| 06 | [Document Versioning, Diffing, and Rollback](06-document-versioning.md) | `DocumentService` (`AddVersionAsync`, `RollbackVersionAsync`, `CompareVersionsAsync`, `CompareDocumentsAsync`), `DocumentDiffDialog.razor`, `VersionHistoryDialog.razor` | `dbo.Documents`, `dbo.DocumentVersions`, `dbo.AuditEvents` | `Code-verified` & `Runtime-verified` |
| 07 | [Audit Logging](07-audit-logging.md) | `AuditService`, `AdminService.GetAuditLogsAsync`, `SecurityAuditLog.razor` | `dbo.AuditEvents`, `dbo.Users` | `Code-verified` & `Runtime-verified` |
| 08 | [Retention and Deletion (Recycle Bin & Purge)](08-retention-and-deletion.md) | `DocumentService` (`DeleteAsync`, `BatchDeleteAsync`, `RestoreAsync`, `BatchRestoreAsync`, `PurgeDocumentAsync`, `EmptyRecycleBinAsync`) | `dbo.Documents`, `dbo.DocumentVersions`, `dbo.SystemSettings`, `dbo.AuditEvents` | `Code-verified` & `Runtime-verified` (Note: Automated retention worker not implemented) |
| 09 | [RAG Knowledge Management and Enterprise Copilot](09-rag-knowledge-management.md) | `DocumentIntelligenceService`, `HnswVectorIndex`, `MultilingualChunkProcessor`, `DocumentLanguageDetector`, `ExternalAiClient`, `DocumentIndexingWorker`, `CopilotDossierExporter`, `EnterpriseAiChatService` | `dbo.DocumentIntelligenceIndices`, `dbo.DocumentVectorChunks`, `dbo.CopilotChatSessions`, `dbo.CopilotChatMessages`, `dbo.CopilotMemoryItems`, `dbo.CopilotUserSettings`, `dbo.Documents` | `Code-verified` & `Runtime-verified` |
| 10 | [Administrative Functionality](10-administrative-functionality.md) | `AdminService`, `ConvertersCatalog.razor`, `SystemSettings.razor`, `SecurityAuditLog.razor`, `UsersAndDepartments.razor` | `dbo.SystemSettings`, `dbo.Converters`, `dbo.Users`, `dbo.Departments`, `dbo.AuditEvents` | `Code-verified` & `Runtime-verified` |

---

## 2. Related Documentation

- [Project Overview](../01-overview/project-overview.md)
- [Requirements Analysis](../02-requirements/requirements-analysis.md)
- [System Architecture](../03-system-design/system-architecture.md)
- [Database Documentation](../05-database/database-documentation.md)
- [API Documentation](../06-api/api-documentation.md)
- [RAG Technical Documentation](../08-rag/rag-technical-documentation.md)

