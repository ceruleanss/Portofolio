# Evidence Register, Open Questions, and Documentation Completeness Report

- **Purpose:** Consolidate the repository-wide Scope and Evidence Register (`EV-*`), the Open Questions and Owner Confirmation Register (`OQ-*`), and the final Documentation Completeness Report required by Phase 6 of the documentation protocol.
- **Scope:** All 11 projects in `DocumentConverter.slnx`, all 24 documentation deliverables (`README.md`, `docs/README.md`, and `docs/01-overview/` through `docs/12-evidence/`), automated verification commands, unverified items, missing implementation evidence, recommended manual validation, and outstanding academic research activities.
- **Evidence status:** `Code-verified` and `Runtime-verified` (for source files, build, and 130 automated xUnit tests); external environment and empirical research items are explicitly marked `Not verified` / `Requires confirmation`.
- **Last reviewed:** 2026-11-05

---

## 1. Scope and Evidence Register (`EV-*`)

| ID | Finding / Document | Primary Source Evidence | Status | Remaining Validation |
|---|---|---|---|---|
| `EV-01` | 11-project .NET 10 (`net10.0`) solution structure and NuGet dependency inventory (`docs/01-overview/project-overview.md`) | `DocumentConverter.slnx`, `global.json`, `Directory.Build.props`, `Directory.Packages.props`, `dotnet --info` (`10.0.403`), `dotnet sln list` | `Code-verified` & `Runtime-verified` | None |
| `EV-02` | Functional requirements (`FR-*`), non-functional requirements (`NFR-*`), and 12 Use Cases (`UC-01`..`UC-12`) (`docs/02-requirements/requirements-analysis.md`) | `src/DocumentConverter.Application/**`, `src/DocumentConverter.Infrastructure/**`, `src/DocumentConverter.Web/**` | `Code-verified` | Confirm organizational stakeholder titles and SLA targets with project owner (`OQ-01`) |
| `EV-03` | Clean Architecture dependency rules and startup/middleware pipeline (`docs/03-system-design/system-architecture.md`) | `src/DocumentConverter.Web/Program.cs`, `src/Workers/DocumentWorker/Program.cs`, `tests/DocumentConverter.Architecture.Tests/ArchitectureTests.cs` (`3/3` passed) | `Code-verified` & `Runtime-verified` | None |
| `EV-04` | Authentication (`.DocConv.Auth` cookie + optional OIDC), `AppClaimsTransformation` (`dc:uid`), and RBAC (`Administrator`, `User`) (`docs/04-modules/01-authentication-and-authorization.md`) | `src/DocumentConverter.Web/Program.cs` (L46–91), `src/DocumentConverter.Web/Authentication/AppClaimsTransformation.cs`, `src/DocumentConverter.Web/Endpoints/AccountEndpoints.cs` | `Code-verified` | Verify external OIDC provider integration if enabled in staging/production (`OQ-02`) |
| `EV-05` | User, role, department, converter, and system settings administration (`docs/04-modules/02-user-and-role-management.md`, `10-administrative-functionality.md`) | `src/DocumentConverter.Infrastructure/Admin/AdminService.cs`, `src/DocumentConverter.Infrastructure/Persistence/DbSeeder.cs`, `src/DocumentConverter.Web/Components/Pages/Admin/*.razor` | `Code-verified` | None |
| `EV-06` | Multi-stage upload validation, `DCE1` AES-256-GCM envelope encryption, and sharded filesystem storage (`docs/04-modules/03-document-upload-and-storage.md`) | `FileSignatureDetector.cs`, `ZipBombGuard.cs`, `EicarSignatureDetector.cs`, `ClamAvScanner.cs`, `AesGcmStorageEncryptionService.cs`, `FileSystemDocumentStorage.cs`, `Milestone8SecurityHardeningTests.cs` (`7/7` passed) | `Code-verified` & `Runtime-verified` | Verify live ClamAV daemon (`dc-clamav:3310`) scan latency under container load |
| `EV-07` | 17 seeded converters across 7 `IDocumentProcessor` engines, RabbitMQ dispatch, and dual-host worker execution (`docs/04-modules/04-document-conversion.md`) | `src/DocumentConverter.Infrastructure/Processing/*.cs`, `RabbitMqJobDispatcher.cs`, `JobProcessorWorker.cs`, `src/Workers/DocumentWorker/Worker.cs`, `FormatExpansionConversionTests.cs` (`9/9` passed) | `Code-verified` & `Runtime-verified` | Verify LibreOffice CLI fallback vs. native execution on production Linux container |
| `EV-08` | Interactive PDF Editor (`PdfEngine`), PAdES PKCS#7 digital signatures (`PdfDigitalSignatureService`), merge/split, and versioning (`docs/04-modules/05-document-management-and-editor.md`, `06-document-versioning.md`) | `PdfEngine.cs`, `PdfDigitalSignatureService.cs`, `DocumentService.cs`, `EditorEndpoints.cs`, `PdfEngineTests.cs` (`17/17`), `PdfDigitalSignatureTests.cs` (`5/5`), `DocumentVersioningDiffTests.cs` (`5/5`) | `Code-verified` & `Runtime-verified` | Confirm whether production signing will use enterprise HSM/CA `.pfx` certificates (`OQ-03`) |
| `EV-09` | Append-only audit logging (`dbo.AuditEvents`) (`docs/04-modules/07-audit-logging.md`) | `src/DocumentConverter.Infrastructure/Security/AuditService.cs`, `src/DocumentConverter.Domain/Entities/AuditEvent.cs` | `Code-verified` | Address missing login audit events and `AI.ApiKey` metadata masking (`TD-04`) |
| `EV-10` | Two-stage deletion lifecycle (Recycle Bin soft-delete & permanent purge) and absence of automated retention worker (`docs/04-modules/08-retention-and-deletion.md`) | `DocumentService.cs` (`DeleteAsync`, `RestoreDocumentAsync`, `PurgeDocumentAsync`), `Document.cs` (`ExpiresAt`), `DbSeeder.cs` (`Retention.Days = 90`) | `Code-verified` & `Runtime-verified` | Confirm whether `Retention.Days` should soft-delete or permanently purge once a worker is implemented (`OQ-04`) |
| `EV-11` | Local & hybrid RAG pipeline (`MarkItDown`/OCR extraction, 180w/30w chunking, 10-rule enrichment, prompt-injection sanitization, 256-D MurmurHash3/BM25 embeddings, `HnswVectorIndex`, ID/EN cross-lingual chunks, MMR, owner-isolated search) (`docs/08-rag/rag-technical-documentation.md`, `docs/04-modules/09-rag-knowledge-management.md`) | `DocumentIntelligenceService.cs`, `HnswVectorIndex.cs`, `MultilingualChunkProcessor.cs`, `DocumentLanguageDetector.cs`, `ExternalAiClient.cs`, `DocumentIntelligenceTests.cs` (`28/29` passed; 1 static-state failure) | `Code-verified` & `Runtime-verified` | Refactor static `GlobalHnswIndex` for test isolation (`TD-03`) and execute empirical IR benchmark ($P@K, R@K, MRR$) |
| `EV-12` | SQL Server database schema (`25` tables across `dbo` [18] and `identity` [7]) and 5 EF Core migrations (`docs/05-database/database-documentation.md`) | `src/DocumentConverter.Infrastructure/Persistence/AppDbContext.cs`, `src/DocumentConverter.Infrastructure/Persistence/Migrations/*.cs` | `Code-verified` | Live SQL Server catalog not inspected (`Database-verified: No`, per non-production safety rules) |
| `EV-13` | Complete HTTP API catalog (`51` Minimal API endpoints + `/health` + `/hubs/processing`) (`docs/06-api/api-documentation.md`) | `AccountEndpoints.cs`, `DocumentEndpoints.cs`, `DocumentIntelligenceEndpoints.cs`, `EditorEndpoints.cs`, `ProcessingEndpoints.cs`, `Program.cs` | `Code-verified` | None |
| `EV-14` | Automated test execution (`130` tests: `129` passed, `1` failed) and evaluation plans (`docs/09-testing/testing-and-evaluation.md`) | `tests/DocumentConverter.Domain.Tests`, `tests/DocumentConverter.Architecture.Tests`, `tests/DocumentConverter.Application.Tests` | `Runtime-verified` (xUnit) / `Not executed` (manual/IR/SUS) | Execute manual UI, performance, RAG IR benchmark, and SUS usability study |

---

## 2. Open Questions and Confirmation Register (`OQ-*`)

| ID | Topic | Observation from Repository | Question Requiring Owner / Supervisor Confirmation | Impacted Documentation |
|---|---|---|---|---|
| `OQ-01` | **Target Case-Study Organization & Stakeholders** | Codebase models generic enterprise departments (`ENGINEERING`, `LEGAL`, `OPERATIONS`) and roles (`Administrator`, `User`). | Which specific organization or university unit is the target case study for the *Tugas Akhir*, and what are the exact stakeholder titles and volume SLAs? | `docs/01-overview/project-overview.md`, `docs/02-requirements/requirements-analysis.md`, `docs/11-research-mapping/academic-research-mapping.md` |
| `OQ-02` | **Identity Provider (Local Cookie vs. OIDC)** | `Program.cs` supports both local ASP.NET Core Identity cookies and optional OpenID Connect (`Auth:UseOidc`). | Will the target deployment use local Identity accounts only, or integrate with an institutional SSO/OIDC provider (e.g., Microsoft Entra ID / Keycloak)? | `docs/04-modules/01-authentication-and-authorization.md`, `docs/10-deployment/deployment-and-maintenance.md` |
| `OQ-03` | **X.509 Certificate Provisioning for PDF Signing** | `PdfDigitalSignatureService` supports user-uploaded `.pfx`/`.p12` certificates or auto-generated ephemeral self-signed RSA-2048 certificates. | Does the organization require integration with an internal PKI / Certificate Authority (CA) for legally binding signatures, or are user-supplied `.pfx` / self-signed certificates sufficient for the project scope? | `docs/04-modules/05-document-management-and-editor.md`, `docs/07-security-governance/security-and-data-governance.md` |
| `OQ-04` | **Automated Retention Policy Semantics** | `Retention.Days = 90` and `Document.ExpiresAt` exist in the database, but no background worker enforces them automatically (`TD-01`). | Should `Retention.Days = 90` govern automatic soft-deletion when `ExpiresAt` is reached, or automatic permanent purging (`PurgeDocumentAsync`) after a document has been in the Recycle Bin (`DeletedAt`) for 90 days? | `docs/04-modules/08-retention-and-deletion.md`, `docs/07-security-governance/security-and-data-governance.md` |
| `OQ-05` | **Department-Level vs. User-Level Document Sharing** | `dbo.Users` links to `dbo.Departments`, but `DocumentService` and `DocumentIntelligenceService` enforce strict single-owner access (`OwnerUserId == currentUserId`). | Is strict per-user isolation intentional for the final thesis evaluation, or should members of the same `Department` be able to view and query shared departmental documents? | `docs/04-modules/03-document-upload-and-storage.md`, `docs/08-rag/rag-technical-documentation.md` |
| `OQ-06` | **Local LLM Hardware Target for Evaluation** | Baseline RAG runs 100% on CPU (`AI.CloudEnabled = false`), with optional `ExternalAiClient` support for local Ollama or cloud OpenAI-compatible APIs. | For the thesis evaluation (`RO-3`), will the researcher evaluate only the deterministic zero-egress extractive synthesizer, or also benchmark a local Ollama model (e.g., `llama3.2` / `qwen2.5`) on local GPU/CPU hardware? | `docs/08-rag/rag-technical-documentation.md`, `docs/09-testing/testing-and-evaluation.md`, `docs/11-research-mapping/academic-research-mapping.md` |

---

## 3. Documentation Completeness Report

### 3.1 Documents Created (24 of 24 Target Files)

| # | Document Path | Deliverable / Section Mapping | Verification Status |
|---:|---|---|---|
| 1 | `README.md` | Section 6: Root repository overview, capabilities, stack, architecture, setup, and documentation index | `Code-verified` & `Runtime-verified` |
| 2 | `docs/README.md` | Documentation portal navigation index, evidence legend, and module/deliverable matrix | `Code-verified` |
| 3 | `docs/01-overview/project-overview.md` | Deliverable A: Project purpose, business context, problem statement, objectives, scope & limitations, technology stack, system overview, and 11-project inventory | `Code-verified` & `Runtime-verified` |
| 4 | `docs/02-requirements/requirements-analysis.md` | Deliverable B: Stakeholder analysis, actors, functional requirements (`FR-*`), non-functional requirements (`NFR-*`), use cases (`UC-01`..`UC-12`), and business processes | `Code-verified` |
| 5 | `docs/03-system-design/system-architecture.md` | Deliverable C: Overall & application architecture, project dependencies, components, data flow, deployment, infrastructure, and security architecture | `Code-verified` & `Runtime-verified` |
| 6 | `docs/04-modules/README.md` | Deliverable E Index: Cross-module navigation and ownership matrix | `Code-verified` |
| 7 | `docs/04-modules/01-authentication-and-authorization.md` | Deliverable E.1: Authentication and authorization module | `Code-verified` |
| 8 | `docs/04-modules/02-user-and-role-management.md` | Deliverable E.2: User, role, and department management module | `Code-verified` |
| 9 | `docs/04-modules/03-document-upload-and-storage.md` | Deliverable E.3: Document upload, malware scanning, `DCE1` encryption, and sharded storage module | `Code-verified` & `Runtime-verified` |
| 10 | `docs/04-modules/04-document-conversion.md` | Deliverable E.4: Asynchronous & synchronous document conversion, compression, and OCR module | `Code-verified` & `Runtime-verified` |
| 11 | `docs/04-modules/05-document-management-and-editor.md` | Deliverable E.5: Document management, interactive PDF editor (`PdfEngine`), and PAdES digital signatures | `Code-verified` & `Runtime-verified` |
| 12 | `docs/04-modules/06-document-versioning.md` | Deliverable E.6: Immutable document versioning, LCS diffing, and non-destructive rollback | `Code-verified` & `Runtime-verified` |
| 13 | `docs/04-modules/07-audit-logging.md` | Deliverable E.7: Append-only security and operational audit logging (`dbo.AuditEvents`) | `Code-verified` |
| 14 | `docs/04-modules/08-retention-and-deletion.md` | Deliverable E.8: Retention policy configuration, Recycle Bin soft-delete, and permanent purge | `Code-verified` & `Runtime-verified` |
| 15 | `docs/04-modules/09-rag-knowledge-management.md` | Deliverable E.9: Enterprise Copilot, hybrid RAG, multi-document comparison, knowledge graph, and dossier export | `Code-verified` & `Runtime-verified` |
| 16 | `docs/04-modules/10-administrative-functionality.md` | Deliverable E.10: System administration, converter catalog governance, and runtime AI settings | `Code-verified` |
| 17 | `docs/05-database/database-documentation.md` | Deliverable D: PlantUML ERD, `dbo` (18 tables) & `identity` (7 tables) schema, full column data dictionary, PK/FK constraints, indexes, migrations, and seeding | `Code-verified` |
| 18 | `docs/06-api/api-documentation.md` | Deliverable G: Complete specification of all 51 Minimal API endpoints + `/health` + `/hubs/processing` | `Code-verified` |
| 19 | `docs/07-security-governance/security-and-data-governance.md` | Deliverable H: Defense-in-depth security analysis, RBAC/IDOR enforcement, `DCE1` encryption, temp-file hygiene, data lifecycle, and RAG-derived data lifecycle | `Code-verified` & `Runtime-verified` |
| 20 | `docs/08-rag/rag-technical-documentation.md` | Deliverable F: Exhaustive 17-section RAG technical specification + separate PlantUML ingestion and query sequence diagrams | `Code-verified` & `Runtime-verified` |
| 21 | `docs/09-testing/testing-and-evaluation.md` | Deliverable I: xUnit v3 test inventory (`129/130` passed), structured test cases (`TC-*`), quantitative RAG IR metrics ($P@K, R@K, MRR$), and 10-item SUS questionnaire | `Code-verified` & `Runtime-verified` (xUnit) |
| 22 | `docs/10-deployment/deployment-and-maintenance.md` | Deliverable J: Environment requirements, configuration reference, local setup, Docker Compose (`dc-internal` zero-egress), GitHub Actions CI/CD, backup/restore, troubleshooting, and technical debt | `Code-verified` & `Runtime-verified` |
| 23 | `docs/11-research-mapping/academic-research-mapping.md` | Section 5: Design Science Research Methodology (DSRM) mapping, research problem, `RO-1`..`RO-4`, `RQ-1`..`RQ-4`, system-integration contribution, and metric traceability | `Code-verified` & `Proposed` |
| 24 | `docs/12-evidence/evidence-and-completeness-report.md` | Section 8 & `AGENTS.md` §7/§10: Scope & evidence register, open questions, and Documentation Completeness Report | `Code-verified` & `Runtime-verified` |

### 3.2 Modules Analyzed (10 of 10 Required Modules)

1. **Authentication and Authorization** (`docs/04-modules/01-authentication-and-authorization.md`)
2. **User and Role Management** (`docs/04-modules/02-user-and-role-management.md`)
3. **Document Upload and Storage** (`docs/04-modules/03-document-upload-and-storage.md`)
4. **Document Conversion** (`docs/04-modules/04-document-conversion.md`)
5. **Document Management and Interactive PDF Editor** (`docs/04-modules/05-document-management-and-editor.md`)
6. **Document Versioning** (`docs/04-modules/06-document-versioning.md`)
7. **Audit Logging** (`docs/04-modules/07-audit-logging.md`)
8. **Retention and Deletion** (`docs/04-modules/08-retention-and-deletion.md`)
9. **RAG Knowledge Management & Enterprise Copilot** (`docs/04-modules/09-rag-knowledge-management.md` & `docs/08-rag/rag-technical-documentation.md`)
10. **Administrative Functionality** (`docs/04-modules/10-administrative-functionality.md`)

### 3.3 PlantUML Diagrams Generated (21 Diagrams Across Deliverables)

- **Root & Overview (`README.md`, `docs/01-overview/project-overview.md`):** Solution Architecture Overview, High-Level System Context Boundary Diagram.
- **Requirements (`docs/02-requirements/requirements-analysis.md`):** System Use Case Diagram (`UC-01`..`UC-12`), Business Process Activity Workflow.
- **System Design (`docs/03-system-design/system-architecture.md`):** C4-Style System Context Diagram, Layered Clean Architecture & Project Dependency Graph, End-to-End Request/Processing/RAG Data Flow Diagram, Docker Compose Network Isolation (`dc-public` vs. `dc-internal`) Deployment Diagram, Defense-in-Depth Security Architecture Diagram.
- **Modules (`docs/04-modules/01..10`):** 10 module-specific sequence, state, and activity diagrams (Authentication & `AppClaimsTransformation` flow, User/Role management flow, Upload validation + ClamAV + `DCE1` encryption sequence, Asynchronous RabbitMQ/Worker conversion sequence, Interactive PDF editor & PAdES signing flow, Document version lifecycle & rollback flow, Audit event recording flow, Two-stage deletion state machine, Copilot workspace interaction flow, Admin governance flow).
- **Database (`docs/05-database/database-documentation.md`):** Complete Crow's Foot Entity Relationship Diagram (ERD) covering all 25 tables in `dbo` and `identity`.
- **RAG Technical (`docs/08-rag/rag-technical-documentation.md`):** Dedicated RAG Ingestion & Cross-Lingual Enrichment Sequence Diagram, Dedicated RAG Hybrid Query, HNSW + MMR & Synthesis Sequence Diagram.
- **Research Mapping (`docs/11-research-mapping/academic-research-mapping.md`):** 6-Phase Design Science Research Methodology (DSRM) Process Diagram.

### 3.4 Unverified Information

In strict compliance with Rule 2 ("Do not invent") and Rule 7 ("Do not access production systems"):
- **Live SQL Server Instance Schema & Production Data (`Not verified`):** All database documentation is derived from `AppDbContext.cs`, entity classes, and the 5 EF Core migration files. No live SQL Server database was queried.
- **Live Container Stack Runtime & External Integrations (`Not verified`):** Docker Compose (`dc-sqlserver`, `dc-rabbitmq`, `dc-clamav`, `dc-web`, `dc-document-worker`), external OIDC identity providers, and live Ollama/OpenAI endpoints were not executed during this analysis session.
- **Empirical RAG IR Metrics & Human Usability Scores (`Not executed`):** Quantitative $P@K$, $R@K$, $MRR$, Faithfulness, Latency benchmarks, and System Usability Scale (SUS) participant questionnaires are designed and documented in `docs/09-testing/testing-and-evaluation.md`, awaiting execution by the researcher.
- **Organizational Context (`Requires confirmation`):** Specific case-study company details, operational SLAs, and literature review references require confirmation by the researcher and academic supervisor.

### 3.5 Missing Implementation Evidence (Code-Verified Gaps in Repository)

1. **Automated Retention Expiration Background Worker (`Not implemented in repository`):** `Retention.Days = 90` (`DbSeeder.cs`) and `Document.ExpiresAt` (`Document.cs`) exist as schema/configuration fields, but no `BackgroundService` automatically soft-deletes or purges expired documents (`TD-01`).
2. **RAG Vector Cleanup on Soft-Delete/Purge (`Partially implemented`):** `SemanticSearchRepositoryAsync` excludes `Deleted` and `Purged` documents in Stage 1 SQL (`Status == DocumentStatus.Available`), but `PurgeDocumentAsync` leaves tombstone rows in `dbo.Documents` (`Status = Purged`), meaning `dbo.DocumentVectorChunks` and `dbo.DocumentIntelligenceIndices` are not deleted from SQL Server nor evicted from `DocumentIndexCache` / `GlobalHnswIndex`, and single-document RAG methods only verify `OwnerUserId` (`TD-02`).
3. **Automatic RAG Re-Indexing on New Document Version (`Not implemented in repository`):** `DocumentService.AddVersionAsync` and `RollbackVersionAsync` do not re-queue the document on `IDocumentIndexingChannel`; manual re-indexing (`POST /api/ai/documents/{id}/reindex`) is required.
4. **Automated Backup Scripts (`Not implemented in repository`):** No cron jobs or backup shell scripts are committed in the repository; recommended backup commands are provided in `docs/10-deployment/deployment-and-maintenance.md` §6.1.

### 3.6 Recommended Manual Validation

1. **Full Docker Compose Stack Smoke Test:** Start `docker compose up -d --build`, log in as `admin@documentconverter.local`, upload a sample PDF and DOCX with `ClamAV:Enabled = true` and `Storage:Encryption:Enabled = true`, verify `DCE1` header bytes on disk, run a conversion job through `dc-document-worker`, and verify real-time SignalR progress notifications on `/hubs/processing`.
2. **PAdES Digital Signature Verification in Adobe Acrobat Reader:** Sign a PDF via `/editor/{id}` (`POST /api/editor/{id:guid}/sign`), download the signed PDF, open it in Adobe Acrobat Reader, and inspect the signature panel to confirm PKCS#7 (`/adbe.pkcs7.detached`) structural validity.
3. **Static HNSW Cache Isolation Fix Verification (`TC-RAG-02`):** Run `dotnet test --filter FullyQualifiedName~CrossLingualRetrieval_EnglishQueryAgainstIndonesianDocument_ShouldMatchIndonesianDocumentPassage` in isolation to verify that the test passes when `GlobalHnswIndex` is not polluted by prior tests, and consider refactoring `GlobalHnswIndex` (`TD-03`).

### 3.7 Outstanding Research Activities (For the *Tugas Akhir* Researcher)

1. **Literature Review & Gap Positioning:** Conduct and cite peer-reviewed academic literature on Enterprise Document Management Systems (EDMS), Retrieval-Augmented Generation (RAG), and Information Governance to substantiate Chapter 2 (*Tinjauan Pustaka*) of the thesis.
2. **Confirm DSRM Scope & Case-Study Organization (`OQ-01`..`OQ-06`):** Review `docs/11-research-mapping/academic-research-mapping.md` and the open questions in Section 2 above with the academic supervisor (*Dosen Pembimbing*).
3. **Execute the Quantitative RAG Benchmark (`docs/09-testing/testing-and-evaluation.md` §3):** Construct the 50-query bilingual ground-truth dataset and compute empirical $P@3$, $P@5$, $R@5$, $MRR$, Faithfulness, and Latency tables.
4. **Conduct the System Usability Scale (SUS) Evaluation (`docs/09-testing/testing-and-evaluation.md` §4):** Administer the 10-item SUS questionnaire to $N \ge 10$ representative users after guided task completion and calculate the final composite SUS score.

