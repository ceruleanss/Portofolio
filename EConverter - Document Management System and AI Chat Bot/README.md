# Enterprise Document Converter & RAG Knowledge Platform — Documentation Index

- **Purpose:** Central navigation portal, deliverable mapping matrix, and evidence status legend for the technical handover and undergraduate Information Systems Final Project (*Tugas Akhir*) documentation suite.
- **Scope:** All 23 Markdown documents under `docs/01-overview/` through `docs/12-evidence/` plus the root `README.md`.
- **Evidence status:** `Code-verified` and `Runtime-verified` (`dotnet build`, `dotnet test` — 129/130 tests passing).
- **Last reviewed:** 2026-11-05

---

## 1. Evidence Status Legend

Throughout this documentation suite, every technical claim is labeled using the following verification taxonomy:

| Label | Meaning |
|---|---|
| `Code-verified` | Verified directly from C# source files, Razor components, `.csproj` / `.props` files, EF Core migrations, Dockerfiles, or CI/CD workflows in this repository. |
| `Runtime-verified` | Verified by executing commands in the workspace (`dotnet --info`, `dotnet sln list`, `dotnet build`, `dotnet test` across 130 xUnit v3 tests). |
| `Inferred` / `Proposed` | Logical architectural or academic DSRM interpretation derived from implementation evidence; subject to researcher/owner confirmation. |
| `Not executed` | Test cases, benchmarks, or usability surveys designed in the documentation that require a live multi-container environment, external runtime (e.g., Ollama), or human participants to execute. |
| `Not verified` / `Requires confirmation` | Organizational context, live production infrastructure, or business policies not present in the source repository. |

---

## 2. Complete Documentation Navigation Matrix

| Directory | Document | Deliverable Mapping | Summary of Contents |
|---|---|---|---|
| **Root** | [Root README](../README.md) | Section 6 (Root README) | Executive overview, key capabilities, verified technology stack, architecture diagram, quick-start commands, and documentation map. |
| **`01-overview`** | [Project Overview & Inventory](01-overview/project-overview.md) | Deliverable A | Project purpose, business context, problem statement, objectives, scope & verified limitations, technology stack, 11-project catalog, and NuGet inventory. |
| **`02-requirements`** | [Requirements Analysis](02-requirements/requirements-analysis.md) | Deliverable B | Stakeholder analysis, actor descriptions, functional requirements (`FR-*` mapped to modules), non-functional requirements (`NFR-*`), 12 use cases (`UC-01`..`UC-12`), and business processes. |
| **`03-system-design`** | [System Architecture](03-system-design/system-architecture.md) | Deliverable C | Overall & Clean Architecture design, project dependency graph, component catalog, startup/middleware pipeline, data flows, Docker network isolation (`dc-internal`), and security architecture. |
| **`04-modules`** | [Module Documentation Index](04-modules/README.md) | Deliverable E (Index) | Overview and cross-reference index for all 10 functional modules. |
| **`04-modules`** | [01 — Authentication & Authorization](04-modules/01-authentication-and-authorization.md) | Deliverable E.1 | ASP.NET Core Identity cookies (`.DocConv.Auth`), OIDC support, `AppClaimsTransformation` (`dc:uid`), RBAC policies (`RequireUser`, `RequireAdministrator`), and login rate limiting. |
| **`04-modules`** | [02 — User & Role Management](04-modules/02-user-and-role-management.md) | Deliverable E.2 | Dual-schema user provisioning (`identity.Users` + `dbo.Users`), role assignment, activation/deactivation, and department mapping (`AdminService`). |
| **`04-modules`** | [03 — Document Upload & Storage](04-modules/03-document-upload-and-storage.md) | Deliverable E.3 | Magic-byte validation, ZIP-bomb & macro guard, EICAR & ClamAV `INSTREAM` scanning, `DCE1` AES-256-GCM envelope encryption, and sharded filesystem storage. |
| **`04-modules`** | [04 — Document Conversion](04-modules/04-document-conversion.md) | Deliverable E.4 | 17 seeded converters across 7 `IDocumentProcessor` engines (PDF, Office, Image, PDF/Image Compress, Tesseract OCR, Echo), RabbitMQ dispatch, and batch ZIP orchestration. |
| **`04-modules`** | [05 — Document Management & PDF Editor](04-modules/05-document-management-and-editor.md) | Deliverable E.5 | Folder hierarchy, owner isolation, interactive browser PDF Editor (`PdfEngine`), and PAdES (`adbe.pkcs7.detached`) cryptographic digital signatures (`PdfDigitalSignatureService`). |
| **`04-modules`** | [06 — Document Versioning](04-modules/06-document-versioning.md) | Deliverable E.6 | Immutable `v{N}.bin` version storage, `SHA-256` integrity tracking, LCS line-by-line text diffing (`DiffVersionsAsync`), and non-destructive rollback (`RollbackVersionAsync`). |
| **`04-modules`** | [07 — Audit Logging](04-modules/07-audit-logging.md) | Deliverable E.7 | Append-only `dbo.AuditEvents` recording across 32 action types (`AuditService`), JSON metadata serialization, and administrative audit viewer. |
| **`04-modules`** | [08 — Retention & Deletion](04-modules/08-retention-and-deletion.md) | Deliverable E.8 | Two-stage deletion lifecycle (Recycle Bin soft-delete `Status = Deleted` and permanent binary purge `Status = Purged`), `Retention.Days` analysis, and derived-data behavior. |
| **`04-modules`** | [09 — RAG Knowledge Management](04-modules/09-rag-knowledge-management.md) | Deliverable E.9 | Enterprise Copilot workspace, single-doc & repository Q&A, multi-document comparison matrices, knowledge graph extraction, persistent chat/memory, and dossier export. |
| **`04-modules`** | [10 — Administrative Functionality](04-modules/10-administrative-functionality.md) | Deliverable E.10 | Platform KPIs, converter catalog management, runtime `dbo.SystemSettings` governance, and AI provider / Zero-Egress (`AI.CloudEnabled`) controls. |
| **`05-database`** | [Database Documentation](05-database/database-documentation.md) | Deliverable D | PlantUML Crow's Foot ERD, 25-table catalog (`18` in `dbo`, `7` in `identity`), complete column data dictionary, PK/FK relationships, indexes, 5 EF Core migrations, and `DbSeeder`. |
| **`06-api`** | [API Documentation](06-api/api-documentation.md) | Deliverable G | Complete specification of all 51 Minimal API routes (`Account`, `Document`, `DocumentIntelligence`, `Editor`, `Processing`) + `/health` + `/hubs/processing`. |
| **`07-security-governance`** | [Security & Data Governance](07-security-governance/security-and-data-governance.md) | Deliverable H | Authentication, RBAC/IDOR enforcement, audit logging, versioning, retention, `DCE1` storage encryption, temporary file cleanup, backups, and RAG-derived data lifecycle. |
| **`08-rag`** | [RAG Technical Documentation](08-rag/rag-technical-documentation.md) | Deliverable F | Exhaustive 17-point RAG deep dive (`MarkItDown`/OCR, 180w/30w chunking, 10-rule enrichment, prompt-injection sanitization, 256-D embeddings, `HnswVectorIndex`, ID/EN cross-lingual chunks, MMR, citations) + 2 sequence diagrams. |
| **`09-testing`** | [Testing & Evaluation](09-testing/testing-and-evaluation.md) | Deliverable I | Automated xUnit v3 inventory (`129/130` passed), 35 structured test cases (`TC-*`), quantitative RAG IR evaluation equations ($P@K, R@K, MRR$, Faithfulness), and 10-item SUS questionnaire. |
| **`10-deployment`** | [Deployment & Maintenance](10-deployment/deployment-and-maintenance.md) | Deliverable J | Hardware/software prerequisites, configuration keys, local setup, Docker Compose (`dc-internal` zero-egress), GitHub Actions CI/CD, backup/restore, monitoring, troubleshooting, and technical debt (`TD-01`..`TD-04`). |
| **`11-research-mapping`** | [Academic Research Mapping](11-research-mapping/academic-research-mapping.md) | Section 5 (DSRM Mapping) | Design Science Research Methodology (DSRM) framing, research problem, objectives (`RO-1`..`RO-4`), questions (`RQ-1`..`RQ-4`), system-integration contribution, artifact description, and metric mapping. |
| **`12-evidence`** | [Evidence & Completeness Report](12-evidence/evidence-and-completeness-report.md) | Section 8 (Completeness Report) | Scope & Evidence Register (`EV-01`..`EV-14`), Open Questions Register (`OQ-01`..`OQ-06`), and final Documentation Completeness Report. |

