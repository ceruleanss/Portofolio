# Testing and Evaluation Documentation

- **Purpose:** Document the existing automated test suites (`xUnit v3`), runtime test execution outcomes, structured test plans and test cases (`TC-*`) across all 10 required verification domains, measurable RAG evaluation protocols, and the System Usability Scale (SUS) evaluation framework.
- **Scope:** Solution test projects (`tests/DocumentConverter.Domain.Tests`, `tests/DocumentConverter.Application.Tests`, `tests/DocumentConverter.Architecture.Tests`), manual and integration verification plans, Information Retrieval (IR) evaluation metrics, and usability evaluation instruments.
- **Evidence status:** `Code-verified` (test suite implementation) and `Runtime-verified` (automated test execution via `dotnet test`); manual UI, end-to-end, performance, and user evaluation cases are marked `Not executed` pending live environment and participant testing.
- **Last reviewed:** 2026-11-05

---

## 1. Test Strategy and Existing Automated Test Inventory

### 1.1 Automated Testing Stack (`Code-verified`)

All test projects in `DocumentConverter.slnx` target `net10.0` (`Directory.Build.props`) and use the centrally managed test packages defined in `Directory.Packages.props`:

| Package | Version | Role |
|---|---|---|
| `Microsoft.NET.Test.Sdk` | `18.3.0` | MSBuild test host and runner infrastructure |
| `xunit.v3` | `3.2.2` | Unit, integration, and architectural test framework (`[Fact]` attributes) |
| `xunit.runner.visualstudio` | `3.1.5` | Visual Studio / `dotnet test` VSTest adapter |
| `Shouldly` | `4.3.0` | Fluent assertion library |
| `NetArchTest.Rules` | `1.3.2` | Architectural layer dependency rule verification |
| `coverlet.collector` | `8.0.0` | Code coverage data collector |
| `Microsoft.EntityFrameworkCore.InMemory` | `10.0.5` | In-memory EF Core provider used in `DocumentConverter.Application.Tests` |

### 1.2 Test Project Inventory and Runtime Execution Summary (`Runtime-verified`)

The solution contains **3 test projects** with a combined total of **130 `[Fact]` test methods**.

**Command Executed:**
```powershell
dotnet test DocumentConverter.slnx --configuration Release -p:IsTestingPlatformApplication=false -p:TestingPlatformDotnetTestSupport=false --verbosity minimal
```

| Test Project | Project Path | Test Classes | Total Tests | Passed | Failed | Skipped | Duration | Status |
|---|---|---|---:|---:|---:|---:|---:|---|
| `DocumentConverter.Domain.Tests` | `tests/DocumentConverter.Domain.Tests/DocumentConverter.Domain.Tests.csproj` | 2 | 13 | 13 | 0 | 0 | 300 ms | `Runtime-verified` (100% pass) |
| `DocumentConverter.Architecture.Tests` | `tests/DocumentConverter.Architecture.Tests/DocumentConverter.Architecture.Tests.csproj` | 1 | 3 | 3 | 0 | 0 | 634 ms | `Runtime-verified` (100% pass) |
| `DocumentConverter.Application.Tests` | `tests/DocumentConverter.Application.Tests/DocumentConverter.Application.Tests.csproj` | 12 | 114 | 113 | 1 | 0 | 14.0 s | `Runtime-verified` (99.1% pass; 1 static-state failure) |
| **Total** | **`DocumentConverter.slnx`** | **15** | **130** | **129** | **1** | **0** | **~15.0 s** | **`Runtime-verified`** |

### 1.3 Automated Test Class Breakdown (`Code-verified`)

| Test Project | Test Class | Relative Path | `[Fact]` Count | Verification Scope |
|---|---|---|---:|---|
| `Domain.Tests` | `DocumentLifecycleTests` | `tests/DocumentConverter.Domain.Tests/DocumentLifecycleTests.cs` | 7 | `Document` creation validation, versioning (`AddVersion`), soft-delete (`MarkDeleted`), restore (`Restore`), and permanent purge (`MarkPurged`) state transitions |
| `Domain.Tests` | `ProcessingJobLifecycleTests` | `tests/DocumentConverter.Domain.Tests/ProcessingJobLifecycleTests.cs` | 6 | `ProcessingJob` state transitions (`Pending` -> `Queued` -> `Running` -> `Succeeded` / `Failed` / `Cancelled` / `DeadLettered`), progress clamping, and attempt tracking |
| `Architecture.Tests` | `ArchitectureTests` | `tests/DocumentConverter.Architecture.Tests/ArchitectureTests.cs` | 3 | `NetArchTest.Rules` assertions verifying `Domain` does not reference `Application`/`Infrastructure`/`Web`, `Application` does not reference `Infrastructure`/`Web`, and `Infrastructure` does not reference `Web` |
| `Application.Tests` | `DocumentIntelligenceTests` | `tests/DocumentConverter.Application.Tests/DocumentIntelligenceTests.cs` | 29 | 10-rule RAG chunk enrichment (`EnrichChunkForRag`), prompt-injection sanitization, extractive summarization, TF-IDF keywords, sentiment, entity extraction, risk & obligation detection, single-doc & repository Q&A, citation anchors, hybrid lexical-semantic retrieval, cross-lingual ID-EN retrieval, multi-document comparison, dossier export, strict user isolation, and persistent Copilot chat/memory |
| `Application.Tests` | `PdfEngineTests` | `tests/DocumentConverter.Application.Tests/PdfEngineTests.cs` | 17 | PDF metadata reading/updating, rotation, page extraction, deletion, insertion, reordering, watermarking, text/image annotations, form field reading/filling, encryption, decryption, and page rendering |
| `Application.Tests` | `FormatExpansionConversionTests` | `tests/DocumentConverter.Application.Tests/FormatExpansionConversionTests.cs` | 9 | Format conversions across `CSV`, `XLSX`, `PPTX`, `HTML`, `MD`, `TXT`, `EPUB`, `DOCX`, and `PDF` |
| `Application.Tests` | `Milestone8SecurityHardeningTests` | `tests/DocumentConverter.Application.Tests/Milestone8SecurityHardeningTests.cs` | 7 | `DCE1` AES-256-GCM envelope storage encryption (`AesGcmStorageEncryptionService`), magic-byte detection (`FileSignatureDetector`), ZIP-bomb/macro guard (`ZipBombGuard`), and `EICAR` detection (`EicarSignatureDetector`) |
| `Application.Tests` | `BatchProcessingTests` | `tests/DocumentConverter.Application.Tests/BatchProcessingTests.cs` | 6 | `BatchJob` creation, multi-item processing, partial failure handling (`PartiallyFailed`), IDOR prevention, and ZIP packaging |
| `Application.Tests` | `CompressionTests` | `tests/DocumentConverter.Application.Tests/CompressionTests.cs` | 6 | PDF compression (`PdfCompressProcessor`), image compression (`ImageCompressProcessor`), and `CompressionEstimatorService` |
| `Application.Tests` | `DocumentSecurityAndIdorTests` | `tests/DocumentConverter.Application.Tests/DocumentSecurityAndIdorTests.cs` | 6 | Owner-isolated IDOR prevention across `DocumentService` (`GetAsync`, `DownloadAsync`, `DeleteAsync`, `PurgeDocumentAsync`, `MoveToFolderAsync`, `AddVersionAsync`) |
| `Application.Tests` | `PdfMergeTests` | `tests/DocumentConverter.Application.Tests/PdfMergeTests.cs` | 6 | Multi-PDF merge ordering, version creation, and corrupted/invalid input rejection |
| `Application.Tests` | `PdfSplitTests` | `tests/DocumentConverter.Application.Tests/PdfSplitTests.cs` | 6 | PDF split by page range and individual pages via `DocumentService.SplitPdfAsync` |
| `Application.Tests` | `DocumentVersioningDiffTests` | `tests/DocumentConverter.Application.Tests/DocumentVersioningDiffTests.cs` | 5 | Line-by-line version diff (`DiffVersionsAsync`), rollback (`RollbackVersionAsync`), and version metadata |
| `Application.Tests` | `PdfDigitalSignatureTests` | `tests/DocumentConverter.Application.Tests/PdfDigitalSignatureTests.cs` | 5 | Ephemeral RSA-2048 X.509 PFX generation, PAdES (`adbe.pkcs7.detached`) signing, signature verification, and byte-tamper detection |
| `Application.Tests` | `OcrProcessingTests` | `tests/DocumentConverter.Application.Tests/OcrProcessingTests.cs` | 4 | `ScannedPdfDetector`, native PDF text extraction vs. scanned image/PDF Tesseract OCR (`TesseractOcrEngine` & `OcrDocumentProcessor`) |
| `Application.Tests` | `PdfCapabilityAndWordConversionTests` | `tests/DocumentConverter.Application.Tests/PdfCapabilityAndWordConversionTests.cs` | 4 | `PdfCapabilityAnalyzer` inspection and PDF-to-Word (`.docx`) conversion |

### 1.4 Root-Cause Analysis of the Single Automated Test Failure (`Code-verified` & `Runtime-verified`)

- **Failing Test:** `DocumentConverter.Application.Tests.DocumentIntelligenceTests.CrossLingualRetrieval_EnglishQueryAgainstIndonesianDocument_ShouldMatchIndonesianDocumentPassage` (`tests/DocumentConverter.Application.Tests/DocumentIntelligenceTests.cs:line 1039`).
- **Assertion Failure:** `Assert.Contains(results.Results, r => r.DocumentId == indonesianDoc.Id)` failed because the top-3 returned results did not include `indonesianDoc.Id`.
- **Root Cause (`Code-verified`):**
  1. `DocumentIntelligenceService` declares `DocumentIndexCache` (`ConcurrentDictionary<Guid, CachedDocumentIndex>`) and `GlobalHnswIndex` (`HnswVectorIndex`) as `private static readonly` fields (`src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs`, lines 37–42).
  2. In `DocumentIntelligenceTests`, the test helper constructs `new DocumentIntelligenceService(documentService: null!, storage, ...)` without an `AppDbContext` (`_dbContext == null`).
  3. Because `_dbContext` is `null`, `SemanticSearchRepositoryAsync(query, topK: 3)` runs with `candidateDocumentIds = null` and searches across **all** vector chunks accumulated in the static `GlobalHnswIndex` by all previous test methods executed in the same xUnit test process.
  4. English contractual documents indexed by earlier tests in `DocumentIntelligenceTests` achieve higher lexical + 256-D vector similarity scores against the English query `"What is the maximum penalty for late delivery?"` than the offline dictionary-translated Indonesian chunk, pushing `indonesianDoc.Id` outside `topK = 3`.

---

## 2. Structured Test Plan and Test Cases (`TC-*`)

All test cases below follow the mandatory 8-column structure (`Test ID`, `Objective`, `Preconditions`, `Steps`, `Expected Result`, `Actual Result`, `Status`, `Evidence Reference`). Manual, UI, performance, and usability test cases that have not been executed against a live environment or human participants are explicitly marked `Not executed`.

### 2.1 Category 1: Functional Suitability (`TC-FUNC-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-FUNC-01` | Verify architectural layer isolation across `Domain`, `Application`, `Infrastructure`, and `Web` assemblies | Solution compiled in `Release` mode | Execute `ArchitectureTests` via `dotnet test` | `Domain` has zero dependencies on other layers; `Application` depends only on `Domain`/`Contracts`/`Abstractions`; `Infrastructure` does not reference `Web` | Passed (`3/3` assertions succeeded in 634 ms) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Architecture.Tests/ArchitectureTests.cs` |
| `TC-FUNC-02` | Verify `ProcessingJob` lifecycle state transitions and progress clamping | None (in-memory domain unit test) | Execute `ProcessingJobLifecycleTests` via `dotnet test` | Job transitions `Pending` -> `Queued` -> `Running` -> `Succeeded`/`Failed`/`Cancelled`/`DeadLettered`; progress clamped to `[0, 100]` | Passed (`6/6` assertions succeeded in 300 ms) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Domain.Tests/ProcessingJobLifecycleTests.cs` |
| `TC-FUNC-03` | Verify multi-document `BatchJob` orchestration and ZIP archive generation | In-memory EF Core context and temporary storage | Execute `BatchProcessingTests` via `dotnet test` | Batch job processes all child `ProcessingJobs`, handles partial failures (`PartiallyFailed`), enforces owner isolation, and packages outputs into a `.zip` archive | Passed (`6/6` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/BatchProcessingTests.cs` |
| `TC-FUNC-04` | Verify end-to-end Blazor Interactive Server dashboard KPI rendering | Running SQL Server + `DocumentConverter.Web` | 1. Log in via `/account/login`<br>2. Navigate to `/`<br>3. Inspect KPI cards and recent jobs | Dashboard displays accurate counts of documents, storage bytes, and processing jobs for the authenticated user scope | Not executed | `Not executed` | `src/DocumentConverter.Web/Components/Pages/Home.razor` |

### 2.2 Category 2: Authentication and Authorization (`TC-AUTH-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-AUTH-01` | Verify valid user login issues `.DocConv.Auth` cookie and attaches `dc:uid` internal user claim | Seeded database with `admin@documentconverter.local` | 1. Submit `POST /api/account/login` with valid email and password<br>2. Inspect HTTP response and cookie | Returns `302 Found` redirect to `/` (or local `returnUrl`) with `.DocConv.Auth` cookie (`HttpOnly`, `SameSite=Lax`, 8h expiry); `AppClaimsTransformation` maps `dc:uid` | Not executed | `Not executed` | `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs`, `src/DocumentConverter.Web/Authentication/AppClaimsTransformation.cs` |
| `TC-AUTH-02` | Verify account lockout after 5 consecutive failed login attempts | Active user account in `identity.Users` | Submit `POST /api/account/login` 5 times with an invalid password | On the 5th attempt (`lockoutOnFailure: true`), user is locked out and redirected with lockout error message | Not executed | `Not executed` | `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs` (line 41) |
| `TC-AUTH-03` | Verify deactivated user (`dbo.Users.IsActive == false`) is blocked from logging in | User exists with `IsActive = false` in `dbo.Users` | Submit `POST /api/account/login` with valid credentials for the deactivated user | Sign-in is rejected before cookie issuance and redirects to `/account/login?errorMessage=...` | Not executed | `Not executed` | `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs` |
| `TC-AUTH-04` | Verify `RequireAdministrator` policy and `AdminService.EnsureAdmin()` block non-Administrator users from `/admin/*` operations | Logged in as standard `User` (`demo@documentconverter.local`) | Navigate to `/admin` or invoke `IAdminService` methods | Access is denied (`UnauthorizedAccessException` / redirect to `/account/login?errorMessage=Access+Denied`) | Not executed | `Not executed` | `src/DocumentConverter.Web/Program.cs` (line 90), `src/DocumentConverter.Infrastructure/Admin/AdminService.cs` |

### 2.3 Category 3: Document Conversion (`TC-CONV-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-CONV-01` | Verify multi-format conversions across `CSV`, `XLSX`, `PPTX`, `HTML`, `MD`, `TXT`, `EPUB`, `DOCX`, and `PDF` | Temporary storage provider and sample files | Execute `FormatExpansionConversionTests` via `dotnet test` | All 9 conversion pathways produce valid target files with non-zero byte size and expected content | Passed (`9/9` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/FormatExpansionConversionTests.cs` |
| `TC-CONV-02` | Verify PDF capability inspection (`PdfCapabilityAnalyzer`) and PDF-to-Word (`DOCX`) conversion | Sample PDF documents | Execute `PdfCapabilityAndWordConversionTests` via `dotnet test` | `PdfCapabilityAnalyzer` accurately reports encryption/signature/text status; PDF-to-Word produces a valid `.docx` file | Passed (`4/4` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/PdfCapabilityAndWordConversionTests.cs` |
| `TC-CONV-03` | Verify OCR pipeline (`TesseractOcrEngine` & `OcrDocumentProcessor`) extracts text from scanned documents and detects native vs. scanned PDFs | Tesseract `eng.traineddata` in `tessdata/` | Execute `OcrProcessingTests` via `dotnet test` | `ScannedPdfDetector` distinguishes native vs. scanned PDFs; `OcrDocumentProcessor` generates searchable PDF and extracted text | Passed (`4/4` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/OcrProcessingTests.cs` |
| `TC-CONV-04` | Verify PDF stream and image compression (`PdfCompressProcessor`, `ImageCompressProcessor`, `CompressionEstimatorService`) | Sample PDF and image files | Execute `CompressionTests` via `dotnet test` | Compressed outputs are valid and pre-execution size estimates are calculated accurately | Passed (`6/6` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/CompressionTests.cs` |

### 2.4 Category 4: Document Management and Editor (`TC-DM-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-DM-01` | Verify 17 PDF manipulation operations in `PdfEngine` (rotate, extract, delete, insert, reorder, watermark, annotations, form fields, encrypt/decrypt, metadata, render) | Sample PDF files in test fixture | Execute `PdfEngineTests` via `dotnet test` | All 17 PDF operations produce structurally valid PDFs matching expected page counts, rotation angles, annotations, and encryption states | Passed (`17/17` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/PdfEngineTests.cs` |
| `TC-DM-02` | Verify multi-PDF merge and PDF split operations | Sample PDF files in test fixture | Execute `PdfMergeTests` and `PdfSplitTests` via `dotnet test` | Merged PDFs contain all source pages in order; split operations create new owned documents with expected page subsets | Passed (`12/12` tests passed across both classes) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/PdfMergeTests.cs`, `PdfSplitTests.cs` |
| `TC-DM-03` | Verify PAdES (`adbe.pkcs7.detached`) PDF digital signing and tamper detection | Ephemeral or provided X.509 `.pfx` certificate | Execute `PdfDigitalSignatureTests` via `dotnet test` | Signed PDF verifies as authentic (`IsSigned = true`, `IsValid = true`); modified byte stream triggers integrity failure | Passed (`5/5` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/PdfDigitalSignatureTests.cs` |
| `TC-DM-04` | Verify `DCE1` AES-256-GCM envelope encryption, magic-byte validation, ZIP-bomb/macro guard, and EICAR rejection | `AesGcmStorageEncryptionService`, `FileSignatureDetector`, `ZipBombGuard`, `EicarSignatureDetector` | Execute `Milestone8SecurityHardeningTests` via `dotnet test` | Encrypted blobs begin with `DCE1` 92-byte header and decrypt transparently; spoofed extensions, macro archives, and `EICAR` streams are rejected | Passed (`7/7` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/Milestone8SecurityHardeningTests.cs` |

### 2.5 Category 5: Document Versioning (`TC-VER-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-VER-01` | Verify `Document` domain entity version increment and invariant validation | None (domain unit test) | Execute `DocumentLifecycleTests` via `dotnet test` | `AddVersion` increments `CurrentVersion` (`1 -> 2`), updates `SizeBytes` and `Sha256Hex`, and appends immutable `DocumentVersion` (`v2.bin`) | Passed (`7/7` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Domain.Tests/DocumentLifecycleTests.cs` |
| `TC-VER-02` | Verify line-level LCS version diff (`DiffVersionsAsync`) and non-destructive version rollback (`RollbackVersionAsync`) | Document with $\ge 2$ versions in `DocumentService` | Execute `DocumentVersioningDiffTests` via `dotnet test` | Diff returns accurate `Added`, `Removed`, and `Unchanged` line segments; rolling back to version 1 creates version 3 (`v3.bin`) copying version 1's binary | Passed (`5/5` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/DocumentVersioningDiffTests.cs` |

### 2.6 Category 6: Retention and Deletion (`TC-RET-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-RET-01` | Verify soft delete moves document to Recycle Bin (`Status = Deleted`, `DeletedAt = UtcNow`) and restore returns it to `Status = Available` | Existing `Available` `Document` | Execute soft-delete and restore tests in `DocumentLifecycleTests` and `DocumentSecurityAndIdorTests` | `MarkDeleted()` sets `Status = Deleted` and `DeletedAt`; `Restore()` sets `Status = Available` and `DeletedAt = null` | Passed (`Runtime-verified`) | `Passed (Runtime-verified)` | `src/DocumentConverter.Domain/Entities/Document.cs`, `tests/DocumentConverter.Domain.Tests/DocumentLifecycleTests.cs` |
| `TC-RET-02` | Verify permanent purge (`PurgeDocumentAsync`) deletes all physical version blobs (`v{N}.bin`) and sets tombstone `Status = Purged` | Soft-deleted document in `DocumentService` | Execute purge test in `DocumentSecurityAndIdorTests` | Physical `.bin` files for all versions are removed from disk via `_storage.DeleteAsync`; `Document.Status` becomes `Purged`; `Document.Purged` audit event is logged | Passed (`Runtime-verified`) | `Passed (Runtime-verified)` | `src/DocumentConverter.Infrastructure/Documents/DocumentService.cs` (lines 480–525) |
| `TC-RET-03` | Verify RAG exclusion of soft-deleted and purged documents during repository search vs. single-document query behavior | Document indexed in RAG, then soft-deleted or purged | 1. Soft-delete or purge an indexed document<br>2. Run `SemanticSearchRepositoryAsync` (repository search)<br>3. Run `AskQuestionAsync(documentId)` (single-doc query) | **Code-Verified Behavior:** Repository search (`SemanticSearchRepositoryAsync`) excludes the document via Stage 1 SQL (`Status == DocumentStatus.Available`), but `dbo.DocumentVectorChunks` rows remain in SQL and single-document `AskQuestionAsync(documentId)` only checks ownership (`OwnerUserId == currentUserId`) without checking `Status == Available` | Verified by static code analysis (`Code-verified`); end-to-end HTTP test not executed | `Not executed` (Code-verified gap documented in `docs/07-security-governance/security-and-data-governance.md` §6) | `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 601–615, 1036–1055) |

### 2.7 Category 7: RAG Retrieval and Response Quality (`TC-RAG-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-RAG-01` | Verify 10-rule RAG chunk enrichment, prompt-injection sanitization, single-doc & repository Q&A, citation anchors, document comparison, dossier export, and persistent Copilot chat | In-memory EF Core context + `DocumentIntelligenceService` | Execute `DocumentIntelligenceTests` (28 of 29 `[Fact]` tests) via `dotnet test` | All 10 enrichment rules serialize valid JSON, prompt injections are redacted to `[UNTRUSTED_DIRECTIVE_REDACTED]`, and extractive/hybrid Q&A, comparison matrices, and dossier exports succeed with page-level citations | Passed (`28/29` tests in `DocumentIntelligenceTests` passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/DocumentIntelligenceTests.cs` |
| `TC-RAG-02` | Verify cross-lingual English query retrieval against an Indonesian document (`CrossLingualRetrieval_EnglishQueryAgainstIndonesianDocument_ShouldMatchIndonesianDocumentPassage`) | `DocumentIntelligenceService` with Indonesian document indexed | Execute `CrossLingualRetrieval_EnglishQueryAgainstIndonesianDocument_ShouldMatchIndonesianDocumentPassage` in full suite run | `SemanticSearchRepositoryAsync(topK: 3)` returns the Indonesian document in top 3 | **Failed** during full-assembly test execution because `static GlobalHnswIndex` retained English contract chunks from earlier tests when `_dbContext` was `null` | `Failed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/DocumentIntelligenceTests.cs:line 1039`, `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 37–42) |
| `TC-RAG-03` | Verify optional local LLM synthesis via Ollama (`POST /v1/chat/completions`) when `AI.CloudEnabled = true` (`Provider = "Ollama"`) | Local Ollama running at `http://localhost:11434/v1` with `llama3.2` pulled | 1. Enable `AI.CloudEnabled = true` via Admin AI Settings or BYOK session<br>2. Submit `POST /api/ai/repository/ask` | `ExternalAiClient` sends grounded prompt to `http://localhost:11434/v1/chat/completions` and returns synthesized answer with page citations | Not executed (no live Ollama instance in test run) | `Not executed` | `src/DocumentConverter.Infrastructure/AI/ExternalAiClient.cs` |

### 2.8 Category 8: Unauthorized Document Retrieval / IDOR Prevention (`TC-IDOR-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-IDOR-01` | Verify strict per-user ownership (`OwnerUserId`) and IDOR prevention on `DocumentService` and `BatchProcessing` | User A and User B in in-memory database | Execute `DocumentSecurityAndIdorTests` (`6/6`) and `BatchProcessingTests.CreateBatchAsync_PreventsIdor_WhenAccessingOtherUsersDocuments` (`1/1`) | User B attempting to read, download, move, version, soft-delete, purge, or batch-process User A's document receives `KeyNotFoundException` (`404 Not Found`) | Passed (`7/7` tests passed) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/DocumentSecurityAndIdorTests.cs`, `BatchProcessingTests.cs` |
| `TC-IDOR-02` | Verify strict per-user ownership isolation in RAG single-document and repository-wide semantic search and Q&A (`DocumentQueriesAndIntelligence_ShouldEnforceStrictUserIsolation`) | User A's confidential document indexed in `DocumentIntelligenceService`; current user context switched to User B | Execute `DocumentIntelligenceTests.DocumentQueriesAndIntelligence_ShouldEnforceStrictUserIsolation` via `dotnet test` | User B is blocked with `UnauthorizedAccessException` on single-document RAG calls (`AnalyzeDocumentAsync`, `AskQuestionAsync`, `SemanticSearchAsync`) and receives zero chunks from User A's documents in `SemanticSearchRepositoryAsync` and `AskRepositoryQuestionAsync` | Passed (`Runtime-verified`) | `Passed (Runtime-verified)` | `tests/DocumentConverter.Application.Tests/DocumentIntelligenceTests.cs` (lines 1105–1195), `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 601–615, 1036–1055) |

### 2.9 Category 9: Performance (`TC-PERF-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-PERF-01` | Measure single-document indexing latency (`EnsureDocumentIndexedAsync`) for a 50-page searchable PDF | Release build running on target hardware | Upload a 50-page PDF and measure time for `DocumentIndexingWorker` to complete `EnsureDocumentIndexedAsync` | Indexing completes within acceptable background threshold (e.g., $< 3,000\text{ ms}$) | Not executed | `Not executed` | `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` |
| `TC-PERF-02` | Measure hybrid HNSW + BM25 + MMR repository retrieval latency (`SemanticSearchRepositoryAsync`) across 1,000 indexed chunks | 1,000 chunks indexed in `GlobalHnswIndex` | Execute 50 warm queries via `POST /api/ai/repository/search` and record $P_{50}$, $P_{95}$, and $P_{99}$ (`SearchDurationMs`) | Extractive retrieval latency $P_{95} < 250\text{ ms}$ | Not executed | `Not executed` | `src/DocumentConverter.Infrastructure/AI/HnswVectorIndex.cs` |
| `TC-PERF-03` | Verify rate limiters enforce `10 requests/minute` on `login-policy` and `20 concurrent requests` on `processing-policy` | Running `DocumentConverter.Web` | Send 11 consecutive requests to `/api/account/login` from the same IP within 1 minute | Requests 1–10 are processed; request 11 returns `429 Too Many Requests` | Not executed | `Not executed` | `src/DocumentConverter.Web/Program.cs` (lines 141–157) |

### 2.10 Category 10: Usability (`TC-USAB-*`)

| Test ID | Objective | Preconditions | Steps | Expected Result | Actual Result | Status | Evidence Reference |
|---|---|---|---|---|---|---|---|
| `TC-USAB-01` | Evaluate end-user perceived usability across document upload, conversion, PDF editor, versioning, and RAG Copilot workflows using the System Usability Scale (SUS) | Deployed staging environment; $N \ge 10$ representative organizational participants (`Administrator` and `User` roles) | 1. Have each participant complete standardized task scenarios (`UC-01` through `UC-12`)<br>2. Administer the 10-item SUS questionnaire (Section 4)<br>3. Compute composite SUS score | Mean composite SUS score $\ge 68.0$ (industry benchmark for acceptable usability) | Not executed | `Not executed` | Section 4 of this document |

---

## 3. Measurable RAG Evaluation Framework

To support rigorous academic evaluation for the Information Systems Final Project (*Tugas Akhir*), the RAG subsystem (`DocumentIntelligenceService`) should be evaluated across both **Information Retrieval (IR) accuracy** and **Response Generation quality** using the metrics defined below.

### 3.1 Information Retrieval (IR) Metrics

Let $Q = \{q_1, q_2, \dots, q_{|Q|}\}$ be a benchmark set of test queries (including monolingual Indonesian, monolingual English, and cross-lingual ID$\leftrightarrow$EN queries), $R_q$ be the ground-truth set of relevant document chunks for query $q$, and $TopK(q) = \langle c_1, c_2, \dots, c_K \rangle$ be the ordered list of $K$ chunks returned by `SemanticSearchRepositoryAsync(q, topK: K)`.

1. **Precision@K ($P@K$):** Measures the fraction of retrieved top-$K$ chunks that are genuinely relevant to query $q$:
   $$P@K(q) = \frac{|\{c_1, c_2, \dots, c_K\} \cap R_q|}{K}, \quad \text{Mean } P@K = \frac{1}{|Q|} \sum_{q \in Q} P@K(q)$$

2. **Recall@K ($R@K$):** Measures the fraction of all ground-truth relevant chunks for query $q$ that are successfully retrieved within the top-$K$ results:
   $$R@K(q) = \frac{|\{c_1, c_2, \dots, c_K\} \cap R_q|}{|R_q|}, \quad \text{Mean } R@K = \frac{1}{|Q|} \sum_{q \in Q} R@K(q)$$

3. **Mean Reciprocal Rank ($MRR$):** Evaluates how high the **first** relevant chunk appears in the ranked list $TopK(q)$. If $\text{rank}_q$ is the 1-based index of the first chunk $c_i \in R_q$ (or $\infty$ if none is found):
   $$MRR = \frac{1}{|Q|} \sum_{q \in Q} \frac{1}{\text{rank}_q}$$

### 3.2 Generation and Groundedness Metrics

1. **Faithfulness / Groundedness ($F$):** Measures whether every factual claim in the generated answer $A_q$ is directly supported by the retrieved context passages $TopK(q)$ without hallucination. Let $S(A_q)$ be the set of atomic statements in $A_q$ and $S_{\text{supported}}(A_q, TopK(q))$ be the subset verifiable from the cited `[Document, p.X]` passages (`Citations` collection in `DocumentQuestionAnswerDto` / `RepositoryQuestionAnswerDto`):
   $$\text{Faithfulness}(A_q) = \frac{|S_{\text{supported}}(A_q, TopK(q))|}{|S(A_q)|}$$
   *(Note: In default zero-egress extractive synthesis mode, `DocumentIntelligenceService` constructs answers directly from top-scored sentences in retrieved chunks, yielding high structural faithfulness by design, whereas neural LLM mode via `ExternalAiClient` requires empirical verification.)*

2. **Answer Relevance ($AR$):** Evaluates how directly and completely the generated answer $A_q$ addresses the user's question $q$ on a 5-point expert rubric ($1 = \text{Irrelevant}$, $5 = \text{Complete and concise}$) or via cosine similarity between the query embedding and answer embedding.

3. **Retrieval and Synthesis Latency ($L$):**
   - **Search Latency ($L_{\text{search}}$):** Reported directly by `RepositorySemanticSearchResultDto.SearchDurationMs` (milliseconds).
   - **Extractive Query Latency ($L_{\text{extractive}}$):** End-to-end wall-clock duration of `AskRepositoryQuestionAsync` with `AI.CloudEnabled = false` (measuring query expansion + 256-D MurmurHash3 embedding + HNSW search + BM25/concept scoring + MMR + sentence synthesis).
   - **Neural LLM Query Latency ($L_{\text{llm}}$):** End-to-end duration with `AI.CloudEnabled = true` (measuring retrieval + HTTP round-trip via `ExternalAiClient`).

### 3.3 Recommended Experimental Protocol for the Thesis

1. **Corpus Construction:** Assemble a controlled test corpus of 30–50 organizational documents (e.g., SOPs, IT policies, procurement contracts, financial spreadsheets, presentations, and scanned memos) across Indonesian and English.
2. **Ground-Truth Annotation:** Define 50 question-chunk pairs ($|Q| = 50$): 20 monolingual English, 20 monolingual Indonesian, and 10 cross-lingual (English query over Indonesian document and vice versa).
3. **Ablation / Comparative Conditions:** Evaluate $P@3$, $P@5$, $R@5$, and $MRR$ across:
   - Baseline lexical-only search vs. Hybrid 256-D HNSW + BM25 + 10-rule RAG Enrichment + Cross-lingual chunk projection.

---

## 4. Usability Evaluation: System Usability Scale (SUS)

### 4.1 Standardized 10-Item SUS Questionnaire

Participants rate each statement on a 5-point Likert scale from **1 (Strongly Disagree)** to **5 (Strongly Agree)** immediately after completing the guided task scenarios:

| Item # | Polarity | Statement | Rating Scale |
|---:|---|---|---|
| 1 | Positive (Odd) | I think that I would like to use this Document Converter & RAG Knowledge Management system frequently. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 2 | Negative (Even) | I found the system unnecessarily complex. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 3 | Positive (Odd) | I thought the system was easy to use. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 4 | Negative (Even) | I think that I would need the support of a technical person to be able to use this system. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 5 | Positive (Odd) | I found the various functions in this system (document conversion, PDF editor, versioning, digital signing, and AI Copilot) were well integrated. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 6 | Negative (Even) | I thought there was too much inconsistency in this system. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 7 | Positive (Odd) | I would imagine that most people would learn to use this system very quickly. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 8 | Negative (Even) | I found the system very cumbersome to use. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 9 | Positive (Odd) | I felt very confident using the system. | 1 (Strongly Disagree) – 5 (Strongly Agree) |
| 10 | Negative (Even) | I needed to learn a lot of things before I could get going with this system. | 1 (Strongly Disagree) – 5 (Strongly Agree) |

### 4.2 SUS Scoring Instructions

Let $x_i \in \{1, 2, 3, 4, 5\}$ denote a participant's raw response to item $i \in \{1, \dots, 10\}$:

1. **For odd-numbered items ($i \in \{1, 3, 5, 7, 9\}$ — positive statements):** Subtract 1 from the user's rating:
   $$s_i = x_i - 1 \in \{0, 1, 2, 3, 4\}$$
2. **For even-numbered items ($i \in \{2, 4, 6, 8, 10\}$ — negative statements):** Subtract the user's rating from 5:
   $$s_i = 5 - x_i \in \{0, 1, 2, 3, 4\}$$
3. **Compute the composite SUS score (range $0\text{–}100$):** Sum the 10 adjusted scores (maximum raw sum $= 40$) and multiply by $2.5$:
   $$\text{SUS} = 2.5 \times \left( \sum_{i \in \{1,3,5,7,9\}} (x_i - 1) + \sum_{i \in \{2,4,6,8,10\}} (5 - x_i) \right)$$
4. **Interpretation Scale (Bangor, Kortum, & Miller / Sauro-Lewis Grade Scale):**
   - $\text{SUS} \ge 80.3$: **Grade A (Excellent)**
   - $68.0 \le \text{SUS} < 80.3$: **Grade B/C (Good / Above Average)** — $68.0$ is the standard literature threshold for acceptable usability.
   - $51.0 \le \text{SUS} < 68.0$: **Grade D (Marginal — usability improvements needed)**
   - $\text{SUS} < 51.0$: **Grade F (Poor / Unacceptable)**

---

## 5. Evidence Traceability

**Evidence**
- `tests/DocumentConverter.Domain.Tests/DocumentLifecycleTests.cs` and `ProcessingJobLifecycleTests.cs` — 13 domain unit tests (`13/13` passed).
- `tests/DocumentConverter.Architecture.Tests/ArchitectureTests.cs` — 3 layer-dependency reflection tests (`3/3` passed).
- `tests/DocumentConverter.Application.Tests/*.cs` — 12 test classes containing 114 integration/application tests (`113/114` passed; 1 static HNSW state failure in `DocumentIntelligenceTests.cs:line 1039`).
- `src/DocumentConverter.Infrastructure/AI/DocumentIntelligenceService.cs` (lines 37–42) — `static` `DocumentIndexCache` and `GlobalHnswIndex` fields explaining `TC-RAG-02`.

**Status:** `Code-verified` and `Runtime-verified` for all 130 xUnit tests; manual/UI/performance/SUS test cases are `Not executed`.
