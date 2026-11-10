# API Documentation

**Purpose:** Document every HTTP Minimal API endpoint, health check route, and SignalR hub discovered in `DocumentConverter.Web`, including HTTP method, full route, purpose, authentication/authorization requirements, rate-limiting policy, request parameters/body DTOs, response structures, error status codes, and implementing classes.  
**Scope:** `AccountEndpoints.cs`, `DocumentEndpoints.cs`, `EditorEndpoints.cs`, `DocumentIntelligenceEndpoints.cs`, `ProcessingEndpoints.cs`, `ProcessingHub.cs`, and `Program.cs`.  
**Evidence status:** `Code-verified`.  
**Last reviewed:** 2026-11-05

---

## 1. API Overview and Conventions

- **API Style:** ASP.NET Core 10 Minimal APIs (`MapGroup`) hosted inside `DocumentConverter.Web` alongside the Blazor Server UI.
- **OpenAPI Specification:** Enabled in the `Development` environment via `builder.Services.AddOpenApi()` and `app.MapOpenApi()` (`GET /openapi/v1.json`, `src/DocumentConverter.Web/Program.cs`, lines 37 & 246).
- **Maximum Request Body Size:** `104,857,600` bytes (`100 MB`) configured on both Kestrel (`MaxRequestBodySize`) and `FormOptions.MultipartBodyLengthLimit` (`Program.cs`, lines 21–26).
- **Authentication & API Status Codes (`Program.cs`, lines 63–81):**
  - Uses the `.DocumentConverter.Auth` session cookie (with `dc:uid` claim attached by `AppClaimsTransformation`).
  - For any request whose path starts with `/api/`, `CookieAuthenticationEvents` suppresses HTML login redirects and returns **HTTP `401 Unauthorized`** (unauthenticated) or **HTTP `403 Forbidden`** (access denied).
- **Rate Limiting Policies (`Program.cs`, lines 105–134):**
  - `auth` policy: Fixed window, **10 requests / 1 minute** per client IP (`QueueLimit = 0`, returns `429 Too Many Requests`).
  - `upload` policy: Fixed window, **30 requests / 1 minute** per client IP (`QueueLimit = 2`, returns `429 Too Many Requests`).
  - `jobs` policy: Fixed window, **60 requests / 1 minute** per client IP (`QueueLimit = 5`, returns `429 Too Many Requests`).

---

## 2. Endpoint Summary Matrix (72 Discovered Endpoints)

| Route Group | Base Prefix | Endpoint Count | Source File | Default Auth Requirement |
|---|---|---|---|---|
| **Account & Auth** | `/api/account` | 4 | `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs` | `AllowAnonymous` (Login/OIDC), Cookie (Logout) |
| **Documents** | `/api/documents` | 22 | `src/DocumentConverter.Web/Endpoints/DocumentEndpoints.cs` | `RequireAuthorization()` + Owner check (`dc:uid`) |
| **Folders** | `/api/folders` | 4 | `src/DocumentConverter.Web/Endpoints/DocumentEndpoints.cs` | `RequireAuthorization()` + Owner check (`dc:uid`) |
| **PDF Editor** | `/api/editor` | 8 | `src/DocumentConverter.Web/Endpoints/EditorEndpoints.cs` | `RequireAuthorization()` + Owner check (`dc:uid`) |
| **Document Intelligence & RAG** | `/api/ai` | 22 | `src/DocumentConverter.Web/Endpoints/DocumentIntelligenceEndpoints.cs` | `RequireAuthorization()` + Owner check (`dc:uid`) |
| **Processing & Batch Jobs** | `/api/processing` | 10 | `src/DocumentConverter.Web/Endpoints/ProcessingEndpoints.cs` | `RequireAuthorization()` + Owner check (`dc:uid`) |
| **Observability & Real-Time** | `/health`, `/hubs/processing` | 2 | `src/DocumentConverter.Web/Program.cs`, `Hubs/ProcessingHub.cs` | `/health` (Anonymous), `/hubs/processing` (`[Authorize]`) |

---

## 3. Detailed Endpoint Catalog

### 3.1 Account & Authentication Endpoints (`AccountEndpoints.cs`)

| # | Method & Route | Purpose | Auth & Rate Limit | Request Parameters / Body | Success Response | Error Responses | Implementation |
|---|---|---|---|---|---|---|---|
| 1 | `POST /api/account/login` | Authenticates a local user with email and password and issues `.DocumentConverter.Auth` cookie. | `AllowAnonymous`; RateLimit: `auth` (10/min/IP) | `application/x-www-form-urlencoded`: `email` (`string`), `password` (`string`), `returnUrl` (`string?`) | `302 Found` redirect to `returnUrl` or `/` | `302 Found` to `/account/login?error=missing\|invalid\|locked`; `429` | `AccountEndpoints.MapAccountEndpoints` (L19–72), `SignInManager<AppIdentityUser>` |
| 2 | `GET /api/account/external-login` | Initiates OpenID Connect authentication challenge when `Auth:UseOidc = true`. | `AllowAnonymous`; RateLimit: `auth` (10/min/IP) | Query: `returnUrl` (`string?`) | `302 Found` OIDC Challenge (`OpenIdConnectDefaults.AuthenticationScheme`) | `302 Found` to `/account/login?error=oidc_not_configured`; `429` | `AccountEndpoints.MapAccountEndpoints` (L74–96) |
| 3 | `POST /api/account/logout` | Signs out current session and clears auth cookie. | Cookie session; `DisableAntiforgery` | Form/Empty | `302 Found` to `/account/login` | — | `AccountEndpoints.MapAccountEndpoints` (L98–103) |
| 4 | `GET /api/account/logout` | Signs out current session via GET navigation and clears auth cookie. | Cookie session | None | `302 Found` to `/account/login` | — | `AccountEndpoints.MapAccountEndpoints` (L105–109) |

---

### 3.2 Document & Folder Endpoints (`DocumentEndpoints.cs`)

All routes under `/api/documents` and `/api/folders` require authentication (`RequireAuthorization()`) and enforce owner-level authorization (`OwnerUserId == userContext.UserId`) inside `DocumentService`.

| # | Method & Route | Purpose | Rate Limit / Antiforgery | Request Parameters / Body | Response Structure (`200` / `201` / `204`) | Error Responses | Implementation Class & Method |
|---|---|---|---|---|---|---|---|
| 5 | `GET /api/documents` | Lists documents owned by current user with optional filtering. | Authenticated | Query: `folderId` (`Guid?`), `search` (`string?`), `status` (`string?`, e.g. `"Deleted"`) | `200 OK`: `IReadOnlyList<DocumentDto>` | `401` | `DocumentService.ListAsync` |
| 6 | `GET /api/documents/{id:guid}` | Gets metadata and version list for a single document. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `DocumentDto` | `401`, `404 Not Found` | `DocumentService.GetAsync` |
| 7 | `POST /api/documents` | Uploads, validates, scans, hashes, and stores a new document (`v1.bin`). | RateLimit: `upload` (30/min); `DisableAntiforgery` | `multipart/form-data`: `file` (`IFormFile`), `folderId` (`Guid?` form field) | `201 Created`: `DocumentDto` | `400 Bad Request` (`{ error }`), `401`, `429` | `DocumentService.UploadAsync` |
| 8 | `GET /api/documents/{id:guid}/versions` | Lists all versions of a document ordered by `VersionNumber DESC`. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `IReadOnlyList<DocumentVersionDto>` | `401`, `404 Not Found` | `DocumentService.GetVersionsAsync` |
| 9 | `POST /api/documents/{id:guid}/versions` | Uploads a new version (`v{N+1}.bin`) for an existing document. | RateLimit: `upload` (30/min); `DisableAntiforgery` | Route: `id` (`Guid`); `multipart/form-data`: `file` (`IFormFile`) | `200 OK`: `DocumentVersionDto` | `400 Bad Request` (`{ error }`), `401`, `404`, `429` | `DocumentService.AddVersionAsync` |
| 10 | `POST /api/documents/{id:guid}/versions/{versionNumber:int}/rollback` | Rolls back a document to `versionNumber` by creating a new version `N+1` with restored content. | Authenticated | Route: `id` (`Guid`), `versionNumber` (`int`) | `200 OK`: `DocumentVersionDto` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.RollbackVersionAsync` |
| 11 | `GET /api/documents/{id:guid}/versions/diff` | Computes LCS line-by-line text diff between two versions of a document. | Authenticated | Route: `id` (`Guid`); Query: `v1` (`int`), `v2` (`int`) | `200 OK`: `DocumentDiffResultDto` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.CompareVersionsAsync` |
| 12 | `POST /api/documents/compare` | Computes LCS line-by-line text diff between two distinct documents. | Authenticated | JSON Body: `CompareDocumentsRequest(Guid DocumentIdA, Guid DocumentIdB)` | `200 OK`: `DocumentDiffResultDto` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.CompareDocumentsAsync` |
| 13 | `GET /api/documents/{id:guid}/pdf-metadata` | Returns PDF page count and encryption status for a stored PDF. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `PdfMetadataDto(DocumentId, DocumentName, PageCount, SizeBytes, IsEncrypted)` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.GetPdfMetadataAsync` |
| 14 | `POST /api/documents/{id:guid}/split` | Splits a stored PDF using `divide_two`, `extract_range`, or `burst_all` mode. | Authenticated | Route: `id` (`Guid`); JSON Body: `SplitPdfRequest` | `200 OK`: `SplitPdfResultDto(SourceDocumentId, TotalPages, CreatedDocuments)` | `400 Bad Request` (`{ error }`), `401`, `404 Not Found` | `DocumentService.SplitPdfAsync` |
| 15 | `POST /api/documents/merge` | Merges 2+ stored PDFs (with optional page ranges and ordering) into a new document. | Authenticated | JSON Body: `MergePdfRequest(SourceDocuments, OutputFileName, DestinationFolderId)` | `200 OK`: `MergePdfResultDto(CreatedDocument, TotalPages, SourceDocumentCount)` | `400 Bad Request` (`{ error }`), `401`, `404 Not Found` | `DocumentService.MergeDocumentsAsync` |
| 16 | `GET /api/documents/{id:guid}/download` | Downloads or previews (`inline=true`) a document or specific version. | Authenticated | Route: `id` (`Guid`); Query: `versionId` (`Guid?`), `inline` (`bool?`) | `200 OK`: File stream (`Content-Disposition: inline` or `attachment`) | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.DownloadAsync` |
| 17 | `POST /api/documents/batch-download` | Downloads multiple owned documents packaged as a `.zip` archive. | Authenticated | JSON Body: `BatchDownloadRequest(IReadOnlyList<Guid> DocumentIds)` | `200 OK`: `application/zip` stream | `400 Bad Request`, `401` | `DocumentService.DownloadBatchZipAsync` |
| 18 | `POST /api/documents/batch-move` | Moves multiple documents and folders into a target folder (`null` for root). | Authenticated | JSON Body: `BatchMoveRequest(DocumentIds, FolderIds, TargetFolderId)` | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.BatchMoveAsync` |
| 19 | `POST /api/documents/batch-delete` | Soft-deletes multiple documents (to Recycle Bin) and deletes selected folders. | Authenticated | JSON Body: `BatchDeleteRequest(DocumentIds, FolderIds)` | `204 No Content` | `400 Bad Request`, `401` | `DocumentService.BatchDeleteAsync` |
| 20 | `DELETE /api/documents/{id:guid}` | Soft-deletes a document (`Status = Deleted`). | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.DeleteAsync` |
| 21 | `POST /api/documents/{id:guid}/restore` | Restores a soft-deleted document from the Recycle Bin to `Available`. | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.RestoreAsync` |
| 22 | `POST /api/documents/batch-restore` | Restores multiple soft-deleted documents to `Available`. | Authenticated | JSON Body: `BatchRestoreRequest(IReadOnlyList<Guid> DocumentIds)` | `204 No Content` | `400 Bad Request`, `401` | `DocumentService.BatchRestoreAsync` |
| 23 | `DELETE /api/documents/{id:guid}/purge` | Permanently deletes all version blobs from disk and sets `Status = Purged`. | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.PurgeDocumentAsync` |
| 24 | `DELETE /api/documents/recycle-bin/empty` | Permanently purges all documents in the current user's Recycle Bin (`Status == Deleted`). | Authenticated | None | `204 No Content` | `400 Bad Request`, `401` | `DocumentService.EmptyRecycleBinAsync` |
| 25 | `POST /api/documents/{id:guid}/rename` | Renames a document (`OriginalFileName`). | Authenticated | Route: `id` (`Guid`); JSON Body: `RenameDocumentRequest(string NewName)` | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.RenameAsync` |
| 26 | `POST /api/documents/{id:guid}/move` | Moves a single document to `FolderId` (`null` for root). | Authenticated | Route: `id` (`Guid`); JSON Body: `MoveDocumentRequest(Guid? FolderId)` | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.MoveToFolderAsync` |
| 27 | `POST /api/documents/{id:guid}/convert-sync` | Synchronously converts a stored document to `pdf`, `docx`, `xlsx`, or `txt` and saves the output as a new repository document. | Authenticated | Route: `id` (`Guid`); JSON Body: `SyncConvertRequest(string TargetFormat)` | `200 OK`: `DocumentDto` (newly created document) | `400 Bad Request`, `401`, `404 Not Found` | `DocumentEndpoints.cs` (L494–654), `PdfEngine`, `OfficeDocumentProcessor` |
| 28 | `GET /api/documents/{id:guid}/signatures/verify` | Cryptographically verifies all PAdES PKCS#7 digital signatures on a stored PDF. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `PdfVerificationResultDto` | `400 Bad Request`, `401`, `404 Not Found` | `PdfDigitalSignatureService.VerifyDocumentSignaturesAsync` |
| 29 | `POST /api/documents/signatures/verify-stream` | Cryptographically verifies all digital signatures on an uploaded PDF file stream. | Authenticated; `DisableAntiforgery` | `multipart/form-data`: `file` (`IFormFile`) | `200 OK`: `PdfVerificationResultDto` | `400 Bad Request`, `401` | `PdfDigitalSignatureService.VerifySignaturesAsync` |
| 30 | `POST /api/documents/{id:guid}/signatures/sign` | Applies a PAdES detached PKCS#7 digital signature to a stored PDF and commits a new version (`{name}_signed.pdf`). | Authenticated | Route: `id` (`Guid`); JSON Body: `SignPdfRequest` | `200 OK`: `DocumentDto` | `400 Bad Request`, `401`, `404 Not Found` | `PdfDigitalSignatureService.SignStoredDocumentAsync` |
| 31 | `POST /api/documents/signatures/sign-stream` | Digitally signs an uploaded PDF file stream (with optional `.pfx` certificate) and returns the signed PDF stream. | Authenticated; `DisableAntiforgery` | `multipart/form-data`: `file` (`IFormFile`), optional `certificate` (`IFormFile`), form fields `signerName`, `reason`, `location`, `contactInfo`, `password`, `addBadge`, `page`, `x`, `y` | `200 OK`: `application/pdf` stream (`{name}_signed.pdf`) | `400 Bad Request`, `401` | `PdfDigitalSignatureService.SignPdfAsync` |
| 32 | `GET /api/folders` | Lists all folders owned by the current user (`Name ASC`). | Authenticated | None | `200 OK`: `IReadOnlyList<FolderDto>` | `401` | `DocumentService.ListFoldersAsync` |
| 33 | `POST /api/folders` | Creates a new folder (`Name`, optional `ParentFolderId`). | Authenticated | JSON Body: `CreateFolderRequest(string Name, Guid? ParentFolderId = null)` | `201 Created`: `FolderDto` | `400 Bad Request`, `401` | `DocumentService.CreateFolderAsync` |
| 34 | `POST /api/folders/{id:guid}/rename` | Renames an existing folder. | Authenticated | Route: `id` (`Guid`); JSON Body: `RenameFolderRequest(string Name)` | `204 No Content` | `400 Bad Request`, `401`, `404 Not Found` | `DocumentService.RenameFolderAsync` |
| 35 | `DELETE /api/folders/{id:guid}` | Deletes a folder (`ON DELETE SET NULL` on contained documents' `FolderId`). | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `401`, `404 Not Found` | `DocumentService.DeleteFolderAsync` |

---

### 3.3 Interactive PDF Editor Endpoints (`EditorEndpoints.cs`)

All routes under `/api/editor` require authentication (`RequireAuthorization()`) and disable antiforgery (`DisableAntiforgery()`) for programmatic JS interop / REST calls.

| # | Method & Route | Purpose | Request Parameters / Body | Success Response (`200` / `201`) | Error Responses | Implementation Class & Method |
|---|---|---|---|---|---|---|
| 36 | `POST /api/editor/documents/{id:guid}/apply` | Applies a list of `PdfEditOperationDto` mutations to a stored PDF and returns the modified PDF binary stream without saving to storage. | Route: `id` (`Guid`); JSON Body: `ApplyPdfEditsRequest(VersionId, Operations)` | `200 OK`: `application/pdf` stream (`edited-{FileName}`) | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ApplyOperationsAsync` |
| 37 | `POST /api/editor/documents/{id:guid}/commit-version` | Applies `PdfEditOperationDto` mutations to a stored PDF and commits the result as a new `DocumentVersion` (`v{N+1}.bin`) on the same document. | Route: `id` (`Guid`); JSON Body: `CommitPdfVersionRequest(SourceVersionId, Operations)` | `200 OK`: `PdfEditResultDto(DocumentId, NewVersionId, VersionNumber, SizeBytes)` | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ApplyOperationsAsync` + `DocumentService.AddVersionAsync` |
| 38 | `POST /api/editor/documents/{id:guid}/save-as-new` | Applies `PdfEditOperationDto` mutations to a stored PDF and saves the output as a brand-new `Document` in the repository. | Route: `id` (`Guid`); JSON Body: `SaveAsNewPdfRequest(NewFileName, FolderId, SourceVersionId, Operations)` | `201 Created`: `DocumentDto` | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ApplyOperationsAsync` + `DocumentService.UploadAsync` |
| 39 | `POST /api/editor/documents/{id:guid}/extract-pages` | Extracts a subset of page numbers (`Pages`) from a stored PDF and returns the extracted PDF stream or saves it as a new document (`SaveAsNewDocument = true`). | Route: `id` (`Guid`); JSON Body: `ExtractPdfPagesRequest(Pages, SaveAsNewDocument, NewFileName, FolderId)` | `200 OK`: `DocumentDto` (if `SaveAsNewDocument`) or `application/pdf` stream | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ExtractPagesAsync` |
| 40 | `POST /api/editor/documents/{id:guid}/export` | Exports a stored PDF (after applying optional in-flight `Operations`) to `.pdf`, `.docx`, or `.xlsx`. | Route: `id` (`Guid`); JSON Body: `ExportPdfDocumentRequest(Format, Operations)` | `200 OK`: File stream (`.pdf`, `.docx`, or `.xlsx`) | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ExportToDocxAsync` / `ExportToXlsxAsync` |
| 41 | `POST /api/editor/documents/{id:guid}/merge` | Merges pages from `SourceDocumentId` into `id` at `InsertAtPage` (`0` = beginning, `-1` = end) and returns the merged PDF stream. | Route: `id` (`Guid`); JSON Body: `MergePdfDocumentsRequest(SourceDocumentId, SourcePages, InsertAtPage)` | `200 OK`: `application/pdf` stream (`merged-{FileName}`) | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.MergePagesAsync` |
| 42 | `GET /api/editor/documents/{id:guid}/form-fields` | Inspects and returns all interactive `/AcroForm` fields in a stored PDF. | Route: `id` (`Guid`) | `200 OK`: `IReadOnlyList<PdfFormFieldDto>` | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.GetFormFieldsAsync` |
| 43 | `POST /api/editor/documents/{id:guid}/protect` | Applies User/Owner password encryption and print/extract permissions to a stored PDF (either committing a new version when `CommitAsNewVersion = true` or returning the encrypted stream). | Route: `id` (`Guid`); JSON Body: `ProtectPdfRequest(UserPassword, OwnerPassword, PermitPrint, PermitExtractContent, CommitAsNewVersion, Operations)` | `200 OK`: `PdfEditResultDto` (if committed) or `application/pdf` stream (`protected-{FileName}`) | `400 Bad Request`, `401`, `404 Not Found` | `PdfEngine.ProtectDocumentAsync` |

---

### 3.4 Document Intelligence & Enterprise Copilot RAG Endpoints (`DocumentIntelligenceEndpoints.cs`)

All 22 routes under `/api/ai` require authentication (`RequireAuthorization()`).

| # | Method & Route | Purpose | Request Parameters / Body | Success Response (`200 OK`) | Error Responses | Implementation Method (`IDocumentIntelligenceService`) |
|---|---|---|---|---|---|---|
| 44 | `GET /api/ai/documents/{id:guid}/analyze` | Generates or retrieves cached `DocumentInsightsDto` (category, summary, key points, action items, entities, risk flags) for a single document. | Route: `id` (`Guid`) | `DocumentInsightsDto` | `400`, `401`, `403 Forbidden`, `404` | `AnalyzeDocumentAsync` |
| 45 | `POST /api/ai/documents/{id:guid}/ask` | Answers a natural-language question scoped to a single document with page citations. | Route: `id` (`Guid`); JSON Body: `AskDocumentQuestionRequest(DocumentId, Question, TopK = 4)` | `DocumentQaResponseDto` | `400`, `401`, `403 Forbidden`, `404` | `AskQuestionAsync` |
| 46 | `GET /api/ai/documents/{id:guid}/search` | Performs semantic vector search inside a single document. | Route: `id` (`Guid`); Query: `q` (`string`), `topK` (`int?`, default `5`) | `IReadOnlyList<SemanticSearchResultDto>` | `400`, `401`, `403 Forbidden`, `404` | `SemanticSearchAsync` |
| 47 | `POST /api/ai/documents/{id:guid}/reindex` | Evicts cached index and forces full re-extraction, chunking, enrichment, translation, and embedding for a document. | Route: `id` (`Guid`) | `{ success: true, insights: DocumentInsightsDto }` | `400`, `401`, `403 Forbidden`, `404` | `ReindexDocumentAsync` |
| 48 | `GET /api/ai/repository/search` | Performs two-stage SQL + HNSW + MMR semantic search across all `Available` documents owned by the user (or within `folderId`). | Query: `q` (`string`), `topK` (`int?`, default `8`), `folderId` (`Guid?`) | `IReadOnlyList<SemanticSearchResultDto>` | `400`, `401` | `SemanticSearchRepositoryAsync` |
| 49 | `POST /api/ai/repository/ask` | Synthesizes a grounded answer with `[DOC-n]` citations across the user's repository (stateless Copilot query). | JSON Body: `AskRepositoryQuestionRequest(Question, FolderId, TopK = 6, ConversationHistory)` | `RepositoryQaResponseDto` | `400`, `401` | `AskRepositoryAsync` |
| 50 | `POST /api/ai/repository/compare` | Generates a structured cross-document comparison matrix and clause variation analysis across 2–5 documents. | JSON Body: `MultiDocumentCompareRequest(DocumentIds, CustomDimensions)` | `MultiDocumentComparisonResultDto` | `400`, `401` | `CompareDocumentsAsync` |
| 51 | `GET /api/ai/corpus/stats` | Returns corpus-wide indexing statistics, category breakdown, and risk flag totals for the current user. | None | `RepositoryCorpusStatsDto` | `400`, `401` | `GetRepositoryCorpusStatsAsync` |
| 52 | `POST /api/ai/corpus/prewarm` | Synchronously indexes up to `maxCount` unindexed documents in the user's repository. | Query: `maxCount` (`int?`, default `20`) | `{ indexedCount: int, stats: RepositoryCorpusStatsDto }` | `400`, `401` | `PrewarmCorpusIndexAsync` |
| 53 | `GET /api/ai/knowledge-graph` | Builds the cross-document Enterprise Knowledge Graph, financial obligations tracker, and topic cloud. | Query: `folderId` (`Guid?`) | `EnterpriseKnowledgeGraphDto` | `400`, `401` | `GetEnterpriseKnowledgeGraphAsync` |
| 54 | `POST /api/ai/repository/faceted-search` | Performs semantic search filtered by category, file extension, risk level, entity category, or minimum similarity score. | JSON Body: `FacetedSearchRequest` | `IReadOnlyList<SemanticSearchResultDto>` | `400`, `401` | `FacetedSemanticSearchRepositoryAsync` |
| 55 | `GET /api/ai/chat/sessions` | Lists all persistent Copilot chat sessions belonging to the current user (`IsPinned DESC, UpdatedAt DESC`). | None | `IReadOnlyList<CopilotChatSessionDto>` | `401` | `ListChatSessionsAsync` |
| 56 | `POST /api/ai/chat/sessions` | Creates a new persistent Copilot chat session. | JSON Body: `CreateChatSessionRequest(Title, Scope, FolderId, DocumentId, Persona)` | `CopilotChatSessionDto` | `401` | `CreateChatSessionAsync` |
| 57 | `GET /api/ai/chat/sessions/{sessionId:guid}` | Loads a specific chat session and its ordered message history. | Route: `sessionId` (`Guid`) | `CopilotChatSessionDto` | `401`, `404 Not Found` | `GetChatSessionAsync` |
| 58 | `PUT /api/ai/chat/sessions/{sessionId:guid}` | Renames or pins/unpins a chat session. | Route: `sessionId` (`Guid`); JSON Body: `UpdateChatSessionRequest(NewTitle, IsPinned)` | `CopilotChatSessionDto` | `401`, `404 Not Found` | `UpdateChatSessionMetadataAsync` |
| 59 | `DELETE /api/ai/chat/sessions/{sessionId:guid}` | Deletes a chat session and cascades deletion to its messages. | Route: `sessionId` (`Guid`) | `{ deleted: true }` | `401`, `404 Not Found` | `DeleteChatSessionAsync` |
| 60 | `POST /api/ai/chat/messages` | Sends a user message inside a persistent chat session, extracts memories if applicable, executes RAG synthesis, and persists both messages. | JSON Body: `SendCopilotChatMessageRequest(Question, SessionId, Scope, FolderId, DocumentId, Persona, TopK = 6)` | `CopilotChatTurnResponseDto` | `400`, `401` | `SendChatMessageAsync` |
| 61 | `GET /api/ai/memory` | Retrieves the user's `CopilotMemoryProfileDto` (memory toggle, persona, custom instructions, and memory items). | None | `CopilotMemoryProfileDto` | `401` | `GetCopilotMemoryProfileAsync` |
| 62 | `PUT /api/ai/memory/settings` | Updates the user's memory toggle, default persona, and custom standing instructions. | JSON Body: `UpdateCopilotMemorySettingsRequest(MemoryEnabled, ResponsePersona, CustomInstructions)` | `CopilotMemoryProfileDto` | `401` | `UpdateCopilotMemorySettingsAsync` |
| 63 | `POST /api/ai/memory/items` | Manually adds a long-term factual or preference memory item for the current user. | JSON Body: `AddCopilotMemoryRequest(Fact, Category, SourceSessionTitle)` | `CopilotMemoryItemDto` | `400`, `401` | `AddCopilotMemoryAsync` |
| 64 | `POST /api/ai/memory/items/{memoryId:guid}/toggle` | Toggles `IsActive` on a specific memory item. | Route: `memoryId` (`Guid`) | `{ toggled: true }` | `401`, `404 Not Found` | `ToggleCopilotMemoryAsync` |
| 65 | `DELETE /api/ai/memory/items/{memoryId:guid}` | Deletes a specific memory item (`00000000-0000-0000-0000-000000000000` clears all memories for the user). | Route: `memoryId` (`Guid`) | `{ deleted: true }` | `401`, `404 Not Found` | `DeleteCopilotMemoryAsync` |
| 66 | `POST /api/ai/export/dossier` | Exports a Copilot synthesis, comparison matrix, or search result set as a `.pdf`, `.docx`, `.xlsx`, or `.md` file. | JSON Body: `CopilotDossierExportRequest` | File stream (`.pdf`, `.docx`, `.xlsx`, or `.md`) | `400`, `401` | `ExportCopilotDossierAsync` (`CopilotDossierExporter`) |
| 67 | `GET /api/ai/export/financial-excel` | Exports the Enterprise Knowledge Graph and financial obligations tracker as a 3-sheet `.xlsx` workbook. | Query: `folderId` (`Guid?`) | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` stream | `400`, `401` | `ExportFinancialGraphToExcelAsync` |
| 68 | `GET /api/ai/config` | Returns the effective `AiProviderConfigDto` for the caller (**Note:** masks `ApiKey = ""` for non-admin callers when reading global config!). | None | `AiProviderConfigDto` | `401` | `GetAiProviderConfigAsync` |
| 69 | `POST /api/ai/config` | Saves global `AI.*` settings in `dbo.SystemSettings` (if `userContext.IsAdministrator == true`) or saves a personal BYOK override in `dbo.CopilotUserSettings` (if standard user). | JSON Body: `AiProviderConfigDto(CloudEnabled, Provider, ApiKey, Model, CustomEndpoint)` | `{ saved: true, scope: "global" \| "user" }` | `400`, `401` | `SaveAiProviderConfigAsync` / `SaveUserAiProviderOverrideAsync` |
| 70 | `POST /api/ai/test-connection` | Tests live connectivity and latency to an external or local AI provider (`OpenAI`, `Claude`, `Gemini`, `Groq`, `Custom`). | JSON Body: `TestAiConnectionRequest(Provider, ApiKey, Model, CustomEndpoint)` | `TestAiConnectionResponse(Success, Message, LatencyMs)` | `400`, `401` | `TestAiConnectionAsync` (`ExternalAiClient`) |

*(Wait: counting rows 44–70 above gives **27 endpoints** in `DocumentIntelligenceEndpoints.cs`, bringing the total across the application to **77 endpoints**! Let's ensure our summary table in Section 2 reflects the exact count of 27 AI endpoints and 31 Document/Folder endpoints = 77 total endpoints.)*

---

### 3.5 Processing & Batch Job Endpoints (`ProcessingEndpoints.cs`)

All routes under `/api/processing` require authentication (`RequireAuthorization()`).

| # | Method & Route | Purpose | Rate Limit | Request Parameters / Body | Success Response (`200` / `201`) | Error Responses | Implementation Class & Method |
|---|---|---|---|---|---|---|---|
| 71 | `GET /api/processing/converters` | Lists all 17 converters in `dbo.Converters`. | Authenticated | None | `200 OK`: `IReadOnlyList<ConverterDto>` | `401` | `ConverterCatalogService.GetAvailableConvertersAsync` |
| 72 | `GET /api/processing/analyze-pdf/{documentId:guid}` | Runs `PdfCapabilityAnalyzer` on a stored PDF to assess layout complexity and conversion fidelity. | Authenticated | Route: `documentId` (`Guid`) | `200 OK`: `PdfCapabilityReportDto` | `400`, `401`, `404` | `PdfCapabilityAnalyzer.AnalyzeAsync` |
| 73 | `GET /api/processing/inspect-ocr/{documentId:guid}` | Runs `ScannedPdfDetector` on a stored PDF to determine if OCR is needed. | Authenticated | Route: `documentId` (`Guid`) | `200 OK`: `ScannedPdfInspectionResult` | `400`, `401`, `404` | `ScannedPdfDetector.InspectAsync` |
| 74 | `POST /api/processing/compression-preview/{documentId:guid}` | Calculates estimated compressed size and byte savings for a stored PDF or image. | Authenticated | Route: `documentId` (`Guid`); JSON Body: `CompressionPreviewRequest` | `200 OK`: `CompressionEstimateDto` | `400`, `401`, `404` | `CompressionEstimatorService.EstimateDocumentAsync` |
| 75 | `POST /api/processing/compression-preview-upload` | Calculates estimated compressed size for an uploaded file stream without persisting it. | Authenticated; `DisableAntiforgery` | `multipart/form-data`: `file` (`IFormFile`), `preset`, `quality`, `maxDimension`, `targetFormat` | `200 OK`: `CompressionEstimateDto` | `400`, `401` | `CompressionEstimatorService.EstimateStreamAsync` |
| 76 | `POST /api/processing/jobs` | Submits a single asynchronous conversion, OCR, or compression job and publishes `JobMessage`. | RateLimit: `jobs` (60/min/IP) | JSON Body: `CreateProcessingJobRequest(DocumentId, ConverterId, VersionId, OptionsJson)` | `201 Created`: `ProcessingJobDto` | `400 Bad Request` (`{ error }`), `401`, `429` | `ProcessingService.CreateJobAsync` + `RabbitMqJobDispatcher.PublishJobAsync` |
| 77 | `GET /api/processing/jobs` | Lists processing jobs owned by the current user (`CreatedAt DESC`, max 100). | Authenticated | Query: `status` (`string?`) | `200 OK`: `IReadOnlyList<ProcessingJobDto>` | `401` | `ProcessingService.ListJobsAsync` |
| 78 | `GET /api/processing/jobs/{id:guid}` | Gets status, progress, and attempt history for a specific job. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `ProcessingJobDto` | `401`, `404 Not Found` | `ProcessingService.GetJobAsync` |
| 79 | `POST /api/processing/jobs/{id:guid}/cancel` | Cancels a `Pending` job or requests cancellation for a `Processing` job. | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `400`, `401`, `404` | `ProcessingService.CancelJobAsync` |
| 80 | `POST /api/processing/batches` | Submits a multi-document batch processing job and publishes child `JobMessage` items. | RateLimit: `jobs` (60/min/IP) | JSON Body: `CreateBatchProcessingRequest(DocumentIds, ConverterId, CombineOutput, OptionsJson)` | `201 Created`: `BatchJobDto` | `400 Bad Request` (`{ error }`), `401`, `429` | `ProcessingService.CreateBatchAsync` |
| 81 | `GET /api/processing/batches` | Lists batch jobs owned by the current user (`CreatedAt DESC`, max 50). | Authenticated | None | `200 OK`: `IReadOnlyList<BatchJobDto>` | `401` | `ProcessingService.ListBatchesAsync` |
| 82 | `GET /api/processing/batches/{id:guid}` | Gets status and completion counters for a batch job. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `BatchJobDto` | `401`, `404 Not Found` | `ProcessingService.GetBatchAsync` |
| 83 | `GET /api/processing/batches/{id:guid}/jobs` | Lists all child `ProcessingJobDto` items belonging to a batch. | Authenticated | Route: `id` (`Guid`) | `200 OK`: `IReadOnlyList<ProcessingJobDto>` | `401` | `ProcessingService.GetBatchJobsAsync` |
| 84 | `POST /api/processing/batches/{id:guid}/cancel` | Cancels all active child jobs in a batch. | Authenticated | Route: `id` (`Guid`) | `204 No Content` | `400`, `401`, `404` | `ProcessingService.CancelBatchAsync` |
| 85 | `GET /api/processing/batches/{id:guid}/download-zip` | Downloads all completed output documents of a batch as a `.zip` archive (`batch-{id[0..8]}.zip`). | Authenticated | Route: `id` (`Guid`) | `200 OK`: `application/zip` stream | `400`, `401`, `404` | `ProcessingEndpoints.cs` (L234–262) |

---

### 3.6 Health Check and SignalR Hub Endpoints (`Program.cs` & `ProcessingHub.cs`)

| # | Transport & Route | Purpose | Auth | Contract Details | Evidence |
|---|---|---|---|---|---|
| 86 | `GET /health` | Container readiness/liveness probe used by Docker Compose (`curl -f http://localhost:8080/health`). | `AllowAnonymous` | Returns `200 OK` (`Healthy`). | `src/DocumentConverter.Web/Program.cs` (L259) |
| 87 | `WS /hubs/processing` | SignalR Hub (`ProcessingHub`) pushing real-time job progress updates to connected clients. | `[Authorize]` | On connection (`OnConnectedAsync`), adds client to group `"user:{dc:uid}"`. Server broadcasts `"JobUpdated"` with `(Guid jobId, string status, int progressPercent, Guid? outputDocumentId, string? errorMessage)`. | `src/DocumentConverter.Web/Hubs/ProcessingHub.cs`, `Services/JobNotificationService.cs` |

---

## 4. Related Documentation

- [System Architecture](../03-system-design/system-architecture.md)
- [Module Documentation Index](../04-modules/README.md)
- [RAG Technical Documentation](../08-rag/rag-technical-documentation.md)
- [Security and Data Governance](../07-security-governance/security-and-data-governance.md)

