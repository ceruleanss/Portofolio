Context locked. 10 AI core modules + contracts + domain aggregates + persistence schema read. Writing the module documentation now.

---

# Document Intelligence and RAG
### Module documentation — EConverter platform

**Module name:** Document Intelligence and RAG
**Domain type:** Core Business Domain
**Layers touched:** Domain · Application · Contracts · Infrastructure · Web
**Documented:** 2026-11-19 · Confidence: 0.88 (source-verified)

---

## 1. Purpose and scope

Document Intelligence and RAG is the domain that turns a static document archive into an **answerable corpus**. It answers a single product question: *given documents a user already owns, how do we return a precise, source-attributed answer without shipping those documents to a third party?*

The module owns five capabilities end to end:

| # | Capability | Primary artefact |
|---|---|---|
| 1 | Asynchronous indexing of persisted documents | `IDocumentIndexingChannel` + `DocumentIndexingWorker` |
| 2 | OCR-aware extraction, language detection, multilingual chunking | `DocumentIntelligenceService`, `MultilingualChunkProcessor`, `DocumentLanguageDetector` |
| 3 | Dual retrieval structures — dense graph + lexical inverted index | `HnswVectorIndex`, `Bm25InvertedIndex` |
| 4 | Multi-stage retrieval orchestration and grounded generation | `RagRetrievalOrchestrator`, `ExternalAiClient` |
| 5 | Conversational continuity, evaluation gate, deliverable export | Copilot aggregates, `RagBenchmarkRunner`, `CopilotDossierExporter` |

**Explicitly out of scope:** the document aggregate itself (owned by *Document Lifecycle and Storage Management*), file conversion engines, encryption, and identity. This module *consumes* those domains and never redefines them.

---

## 2. Architectural placement

The module follows the solution-wide four-layer split. Dependencies point inward only.

```
DocumentConverter.Web            Blazor pages, /api/documents endpoints, DocumentIndexingWorker
        │  (depends on Application contracts)
        ▼
DocumentConverter.Application    IDocumentIntelligenceService · IDocumentIndexingChannel · IExternalAiClient
        ▲  (Infrastructure implements; Domain depends on nothing outward)
        │
DocumentConverter.Infrastructure HNSW · BM25 · Orchestrator · ExternalAiClient · Benchmark · Exporter
        │
        ▼
DocumentConverter.Domain         DocumentVectorChunk · DocumentIntelligenceIndex · Copilot* aggregates
                                 (+ DocumentConverter.Contracts.AI — shared DTO vocabulary)
```

**Placement rules that matter in practice:**

- **Contracts/AI is the shared vocabulary.** `SemanticChunkDto`, `GlobalSemanticCitationDto`, `RagRetrievalTraceDto`, `VectorSourceTypes`, `AiProviders` all live there because Web, Application, and Infrastructure must speak the same shapes. `AiProviders` also carries the only two *behavioural* members in the contract layer: `GetDefaultModel(provider)` and `GetDefaultEndpoint(provider)`.
- **Domain has no outward references.** `DocumentVectorChunk` and `DocumentIntelligenceIndex` are plain guarded aggregates; they never import an AI library.
- **Web is a composition root, not a logic host.** `DocumentIntelligenceEndpoints` binds a request contract, calls exactly one Application method, and maps the result.

### 2.1 DI lifetimes (from `Program.cs`)

| Registration | Lifetime | Why |
|---|---|---|
| `IDocumentIndexingChannel` → `DocumentIndexingChannel` | **Singleton** | Producer (upload endpoints) and consumer (worker) must share one buffer |
| `DocumentIndexingWorker` | Hosted service | Long-running background consumer |
| `IDocumentIntelligenceService` → `DocumentIntelligenceService` | **Scoped** | Depends on `AppDbContext` |
| `IExternalAiClient` → `ExternalAiClient` | Scoped | Http-backed transport |
| `GlobalHnswIndex`, `GlobalBm25Index` | **Static process-wide** | Cross-document indexes must outlive any request scope |

> **Deployment implication (must be understood before scaling out):** `HnswVectorIndex`, `Bm25InvertedIndex`, and `DocumentIndexCache` are `static` fields on `DocumentIntelligenceService`. They are hydrated from PostgreSQL at startup by `HydrateCorpusVectorIndexAsync`, so correctness is preserved — but each application instance maintains its **own** in-memory graph. A newly indexed document is visible to the instance that indexed it immediately, and to its peers only after their own hydration or on-demand indexing. Horizontal scaling therefore trades recall freshness across instances for in-process query speed.

---

## 3. The retrieval atom and the contract surface

### 3.1 `DocumentVectorChunk` — the unit of retrieval

`src/DocumentConverter.Domain/Documents/DocumentVectorChunk.cs`

| Property | Type | Constraint |
|---|---|---|
| `DocumentId` | `Guid` | FK → Documents (cascade delete) |
| `ChunkIndex` | `int` | ≥ 0, ordinal within document |
| `PageNumber` | `int` | ≥ 1 |
| `SectionTitle` | `string?` | max 300 |
| `Content` | `string` | required |
| `Embedding` | `byte[]` | 256 × `float32` = **1024 bytes** |
| `Language` | `string` | max 10 — `"orig"`, `"en"`, `"id"` |
| `ParentChunkId` | `Guid?` | links a translation back to its source chunk |
| `IsEnriched` | `bool` | true for RAG-enrichment-generated chunks |
| `VectorSource` | `string` | max 80 — `"Actual Document"` or `"RAG Enrichment"` |
| `ExtractionMethod` | `string` | max 100 — e.g. `"MarkItDown"`, `"MarkItDown + OCR"` |
| `EnrichmentMetadataJson` | `string?` | structured enrichment payload |

Every value is normalised in the private `Create` factory: over-long strings are truncated, `Language` is lower-cased and trimmed, `VectorSource` is inferred from `IsEnriched` when blank.

**Why 1024 bytes matters:** the embedding is a fixed-width binary blob, so a chunk row is a single cheap `bytea` read. Hydration does `float[] ↔ byte[]` conversion in memory with no deserialisation framework.

### 3.2 `VectorSourceTypes` — the provenance flag

```csharp
ActualDocument    = "Actual Document"   // verbatim extracted text
EnrichmentProcess = "RAG Enrichment"    // generated/synthetic content
All               = "All"
AllFilterOptions  = [All, ActualDocument, EnrichmentProcess]
```

Provenance is carried **all the way through** the pipeline: `SemanticChunkDto` → `HnswSearchResult` → `SemanticSearchResultDto` / `GlobalSemanticCitationDto`, and it is re-checked at validation time. An enrichment-derived passage can never be presented as a verbatim document quote without the caller being able to see the distinction.

### 3.3 `IDocumentIntelligenceService` — the application contract

`src/DocumentConverter.Application/AI/IDocumentIntelligenceService.cs`

Grouped by concern:

| Group | Members |
|---|---|
| Analysis | `AnalyzeDocumentAsync`, `AnalyzeDocumentStreamAsync` |
| Q&A | `AskQuestionAsync`, `AskQuestionOnStreamAsync`, `AskRepositoryQuestionAsync` |
| Search | `SemanticSearchAsync`, `SemanticSearchRepositoryAsync`, `FacetedSemanticSearchRepositoryAsync` |
| Lifecycle | `EnsureDocumentIndexedAsync`, `ReindexDocumentAsync`, `HydrateCorpusVectorIndexAsync`, `PrewarmCorpusIndexAsync` |
| Corpus insight | `GetRepositoryCorpusStatsAsync`, `GetEnterpriseKnowledgeGraphAsync`, `CompareDocumentsAsync` |
| Providers (BYOK) | `GetAiProviderConfigAsync`, `SaveAiProviderConfigAsync`, `TestAiConnectionAsync` |
| Chat & memory | `ListChatSessionsAsync`, `GetChatSessionAsync`, `CreateChatSessionAsync`, `UpdateChatSessionMetadataAsync`, `DeleteChatSessionAsync`, `ClearAllChatSessionsAsync`, `SendChatMessageAsync`, `StreamChatMessageAsync`, `GetCopilotMemoryProfileAsync`, `UpdateCopilotMemorySettingsAsync`, `AddCopilotMemoryAsync`, `ToggleCopilotMemoryAsync`, `DeleteCopilotMemoryAsync` |
| Diagnostics | `UnderstandQueryAsync`, `GetRecentRetrievalTraces`, `RunRetrievalBenchmarkAsync` |
| Export | `ExportCopilotDossierAsync`, `ExportCorpusFinancialTableToExcelAsync` |

Two contract-level guarantees are built in, not left to callers:

1. **Access control is a precondition.** Every content-returning member resolves the ambient user and verifies ownership before reading. The implementation throws `UnauthorizedAccessException` with the message `Access denied: Document '{id}' does not belong to the current user.`
2. **Cancellation actually reaches the iterator.** The project imports `System.Runtime.CompilerServices` so streaming members annotate with `[EnumeratorCancellation]`. A token passed by the caller propagates into the async iterator instead of being captured at call time.

### 3.4 `IDocumentIndexingChannel` — the backpressure contract

```csharp
bool      TryQueueDocument(Guid documentId);                       // non-blocking
ValueTask QueueDocumentAsync(Guid id, CancellationToken ct = default); // awaiting path
IAsyncEnumerable<Guid> ReadAllAsync(CancellationToken ct = default);   // consumer
int       QueuedCount { get; }                                     // backlog observability
```

`TryQueueDocument` returning `false` **is** the backpressure signal. The caller decides whether to fail fast, retry, or index inline — the channel never blocks an upload thread.

---

## 4. Indexing intake and the background pipeline

### 4.1 `DocumentIndexingChannel` — bounded, non-blocking

`src/DocumentConverter.Infrastructure/AI/DocumentIndexingChannel.cs`

| Setting | Value | Rationale |
|---|---|---|
| Capacity | **10,000** | Bounded — an upload storm cannot exhaust memory |
| `FullMode` | `Wait` | `QueueDocumentAsync` awaits; `TryQueueDocument` returns `false` |
| `SingleReader` | `true` | Exactly one worker consumes; enables channel fast paths |
| `SingleWriter` | `false` | Any number of upload/conversion threads may enqueue |
| Logger | `NullLogger` fallback | The type is constructible and testable with no logging stack |

`QueuedCount` is maintained with `Interlocked` / `Volatile`, incremented on write and decremented on read, so backlog monitoring is lock-free.

### 4.2 `DocumentIndexingWorker` — the consumer

`src/DocumentConverter.Web/HostedServices/DocumentIndexingWorker.cs`

**Startup sequence:**

1. Create one DI scope, resolve `IDocumentIntelligenceService`, call `HydrateCorpusVectorIndexAsync`.
2. Query `DocumentIntelligenceIndices` for already-indexed IDs and `Documents` where `Status == Available`, ordered newest-first.
3. Enqueue every Available document **not yet indexed** via `TryQueueDocument`.
4. Failures during hydration are logged as a warning and the worker continues — indexing simply falls back to on-demand.

**Consumption loop:**

```
await foreach (documentId in _queue.ReadAllAsync(stoppingToken))
    → process with Parallel.ForEachAsync(MaxDegreeOfParallelism = clamp(ProcessorCount, 2, 4))
        → per item: create a fresh DI scope → resolve service
        → EnsureDocumentIndexedAsync(docId, ct)
        → per-item try/catch + warning log
```

**Isolation guarantees:**

- **One scoped `DbContext` per work item.** No shared context, no cross-document tracking bleed.
- **One malformed document cannot kill the consumer.** Each item has its own try/catch; a failure logs and the loop continues.
- **Cancellation propagates through every `await`** and is distinguished from genuine faults (`OperationCanceledException when stoppingToken.IsCancellationRequested` is info-level; anything else is error-level).

> **Implementation note worth recording.** The batch-drain loop in the current source contains an unconditional `break`, so a dequeue is always processed as a single-item batch rather than the intended up-to-8 grouping. Parallelism is therefore still correct and safe, but the batching optimisation is inert. This is a performance note, not a correctness defect.

**Net effect:** document persistence latency is fully decoupled from indexing cost, and indexing runs at background priority rather than on the request thread.

---

## 5. Analysis pipeline and concurrency hygiene

`DocumentIntelligenceService` is the largest single type in the solution (~8,500 lines) and the module's orchestrator.

### 5.1 Access check first, always

`AnalyzeDocumentInternalAsync` opens with `EnsureUserHasDocumentAccessAsync`. If `IUserContext.UserId` is absent (background context), the check is skipped by design so the worker can index without an HTTP principal — but the *foreground* paths always carry a user.

### 5.2 Two-tier read path

```
AnalyzeDocumentAsync(documentId)
  ├─ 1. DocumentIndexCache hit?            → return immediately (in-process)
  ├─ 2. DocumentIntelligenceIndices row?   → rehydrate chunks from DB,
  │                                          re-upsert into HNSW + BM25,
  │                                          return without touching storage
  └─ 3. Cold path                          → download stream → extract → chunk →
                                             embed → upsert → persist → return
```

Tier 2 is the important one: a cold-started instance answers from the persisted index **without re-parsing the PDF or DOCX stream**.

### 5.3 Concurrency and EF hygiene

| Mechanism | Purpose |
|---|---|
| `AcquireDbLockAsync` | Per-`DbContext` `SemaphoreSlim` held in a `ConditionalWeakTable<AppDbContext, SemaphoreSlim>`, so two operations on the same context never interleave |
| `SemaphoreReleaser` | Nested `IDisposable` that releases via `Interlocked.Exchange`, making double-dispose impossible |
| `NoopDisposable` | Zero-allocation release when no `DbContext` is present (unit-test path) |
| `ClearChangeTrackerSafely` | `ChangeTracker.Clear()` after bulk read/write phases so long-lived scoped contexts do not accumulate tracked graphs; wrapped in try/catch for disposed contexts |
| Atomic upsert | `UpsertDocumentToGlobalIndexes` writes HNSW and BM25 in the same operation, so neither structure ever observes a half-written document |

### 5.4 The `fastOnDemandMode` flag

The same internal analysis routine serves two callers:

- **`fastOnDemandMode: false`** — the ordinary path. Enables cloud translation, neural embeddings, full enrichment.
- **`fastOnDemandMode: true`** — used by `EnsureDocumentIndexedAsync` and the worker. Skips the provider config lookup, disables neural translation and neural embeddings, and uses only the deterministic in-process pipeline.

This is what makes the background worker viable offline: it always has a complete, dependency-free route to a usable index.

---

## 6. Chunking, language detection and cross-lingual alignment

### 6.1 Extraction

`ExtractPagesFromStream` dispatches by real content signature and extension, producing `ExtractedPage(PageNumber, Text, ExtractionMethod)`:

| Source | Path |
|---|---|
| PDF | MarkItDown first, then per-page PdfPig text; OCR on embedded images only when the page has **< 200 characters** of selectable text **and** the OCR budget (< 2500 ms) is not exhausted |
| Scanned PDF page | < 25 chars of selectable text → Tesseract OCR, heading `# Page N (Scanned PDF OCR)` |
| DOCX / XLSX | MarkItDown; OpenXML fallback for XLSX that reconstructs each worksheet as a Markdown table with a header separator row |
| PPTX | MarkItDown per slide, with OCR for slide images |
| Images | Magic-byte sniffing (PNG, JPEG, TIFF, BMP, GIF, RIFF/WebP) → OCR |

The OCR budget is deliberate: a 400-page scanned PDF cannot hold a request thread hostage.

### 6.2 Language detection — pure local, sub-millisecond

`DocumentLanguageDetector` is a `static` classifier with two `readonly HashSet<string>` corpora (Indonesian and English) compared with `OrdinalIgnoreCase`. It tokenises, counts matches, and returns `"id"` or `"en"`. No model, no network, no heavy allocation. Detection runs on a sample of the first five pages.

The Indonesian corpus is intentionally broad — it includes colloquial forms (`nggak`, `gimana`, `ngerjain`), legal/business terms, and question words — because informal and formal Indonesian appear in the same corpus.

### 6.3 Chunk construction

`BuildSemanticChunks` splits on word windows and resolves each window's heading via `ExtractPageHeadingAnchors` + `ResolveChunkHeadingTitle`:

- Heading anchors are **word-offset** positions, not character offsets.
- Recognised heading forms: Markdown (`# …`, up to 4 levels), numbered clauses (`4.2`), `SECTION`/`ARTICLE`/`CHAPTER`/`BAB`/`PASAL` prefixes, all-caps lines, and short title-case lines (≤ 8 words).
- The resolution order is: headings **inside** the window (joined with ` — `, max 2) → otherwise the most recent heading **before** the window, carried forward.

That carry-forward is what gives a chunk on page 7 the section title introduced on page 5 — essential for citation quality.

`TryParseSectionHeadingLine` deliberately rejects lines ending in `.`, `,`, or `;` so ordinary prose is not mistaken for a heading.

### 6.4 Cross-lingual representation

`MultilingualChunkProcessor` produces *additional* chunk rows rather than replacing content:

| Source language | Generated representations |
|---|---|
| `id` | one `en` child chunk |
| `en` | one `id` child chunk |
| anything else | **both** `en` and `id` children |

Each child carries `ParentChunkId = <source chunk id>` and a section title suffixed `(EN)` / `(ID)`. Source chunks keep `Language = "orig"`.

**Two translation paths, deliberately:**

1. **Neural batch path** — used only when `CloudEnabled` is on and a key (or the `Custom` provider) is present. Batches of 5 chunks, 45-second timeout per batch, delimiter-framed prompt (`<<<CHUNK_n>>>`), parsed by `ParseBatchTranslationResponse`. The prompt instructs the model to preserve dates, currency codes, numeric values, clause identifiers, and proper names verbatim.
2. **Deterministic offline projection** — `ProjectOfflineTranslation` performs dictionary plus token-preserving projection using `IdToEnDictionary` / `EnToIdDictionary` (~150 lexical-concept pairs covering contract, finance, HR, insurance/IFRS, and research terminology). **Identical input always yields identical output**, which is precisely what makes the benchmark reproducible.

**Failure behaviour is graceful and partial:** three consecutive neural batch failures stop further batches, keep everything already translated, and project the remainder offline. Translation is an enhancement, never a hard dependency.

---

## 7. Retrieval structures

### 7.1 `HnswVectorIndex` — dense approximate nearest neighbour

`src/DocumentConverter.Infrastructure/AI/Hnsw/HnswVectorIndex.cs`

**Construction parameters:** `m = 16` (graph degree, floored at 4), `efConstruction = 64` (floored at `m`), `m0 = 2m` (layer-0 degree), `mL = 1 / ln(m)` (level decay), `Random(42)` seeded for reproducible level assignment, level capped at 16.

**Concurrency:** a `ReaderWriterLockSlim` guards all state. Searches take read locks; insert/remove/clear take write locks. `Dispose` disposes the lock.

**Multi-dimension support — the key design choice.** The index is **not** single-dimension. `_dimensionEntries` maps `dimension → (entryPointId, maxLevel)`. A 256-dimension hash embedding and, say, a 1536-dimension neural embedding coexist in one graph, each with its own entry point. **Searches are dimension-scoped**, so a mismatched query vector is rejected rather than silently scored against the wrong space.

**Insert (`Add`):**

1. **Idempotent dedupe** — if the same `ChunkId` is already active with the same `DocumentId` *and* identical content, return the existing internal id. Otherwise mark the old node deleted.
2. Assign a random level.
3. Greedy-descend from the dimension's entry point down to `level + 1` (`GreedySearchLayer`).
4. From `min(maxLevel, level)` down to 0: gather `efConstruction` candidates (`SearchLayer`), select ≤ `m0`/`m` neighbours, add bidirectional edges, then `ShrinkNeighbors` on the neighbour if it exceeds budget.
5. Update the dimension entry point if the new node's level exceeds the current maximum.

**Search (`Search`):**

1. Reject empty queries, `topK <= 0`, and an empty candidate document set.
2. Return empty if no entry point exists **for the query's dimension**.
3. Greedy-descend layers `maxLevel → 1`.
4. Layer-0 beam search with an adaptive width:
   - No document filter → `efSearch` (default 32), floored at `topK * 2`.
   - With document filter → `max(efSearch * 3, max(topK * 4, 64))` — a wider beam compensates for post-filter attrition.
5. Filter out deleted nodes, dimension mismatches, and unauthorised documents.
6. **Targeted brute-force escalation:** if a *tight* scope (≤ 24 documents) still yields fewer than `topK` hits, the search brute-forces the node lists of those documents directly with `CosineSimilarity`. This is the safety net that makes single-folder Q&A reliable — approximate recall cannot be allowed to produce a false "not found".
7. Sort descending, round similarity to 3 decimals, materialise `HnswSearchResult`.

**`HnswSearchResult`** is a record carrying `ChunkId, DocumentId, PageNumber, SectionTitle, Content, SimilarityScore, IsEnriched, VectorSource, ExtractionMethod, EnrichmentMetadataJson, Embedding, Language, ParentChunkId`. **The graph vertex is the retrieval payload** — a hit needs no secondary lookup to build a citation.

**Lifecycle:** `ContainsDocument`, `RemoveDocument` (soft: adds to `_deletedNodes`, rebuilds the dimension entry point if the deleted node was an entry, then prunes maps), `Clear`, `Count`, `GetIndexedDimensions`.

**Scoring:** `CosineSimilarity` is `static`, shared by build and query paths, and **clamps the dot product to `[0, 1]`**.

> **Honest note on the clamp.** Because hash embeddings are non-negative and L2-normalised, the clamp is safe and keeps scores interpretable. It does mean genuinely opposed vectors are reported as `0.0` rather than negative similarity — the index is a similarity ranker over non-negative feature spaces, not a signed-distance metric.

> **Memory note.** `_nodes` is append-only; deletion marks nodes rather than compacting the list. Long-lived instances that re-index repeatedly grow monotonically unless `Clear` is called. This is the correct trade-off for read-mostly corpora and should be revisited only if re-index churn becomes high.

### 7.2 `Bm25InvertedIndex` — lexical Okapi BM25

`src/DocumentConverter.Infrastructure/AI/Bm25InvertedIndex.cs` — `sealed partial class`, `IDisposable`, held entirely in memory.

**Parameters:** `K1 = 1.2`, `B = 0.75` (standard Okapi defaults).

**Thread safety by discipline:** every mutation of an `IndexedEntry` flows through an `*Unlocked` helper (`MarkEntryDeletedUnlocked`, `RemoveDocumentUnlocked`) that is only ever called while the write lock is held. Public methods acquire the lock themselves. There is no path by which a reader can observe a torn posting list.

**Field boosts applied at index time** (`BuildWeightedTermFrequencies`):

| Field | Weight |
|---|---|
| Chunk content | 1.0 |
| Section title | **2.0** |
| Document name (without extension) | 1.2 |
| Enrichment technical identifiers & terms | **1.6** |
| Enrichment generated search questions | 1.3 |
| Enrichment contextual summary | 1.2 |

Because these land in the *same* term-frequency map, a title term and a body term carry different weight for the same query — no separate field-index bookkeeping required.

**Stemming:** `StemToken` is a regex/suffix-based stemmer handling English (`-ations`, `-ation`, `-ments`, `-ment`, `-ness`, `-ing`, `-ed`, plural `-ies`/`-es`/`-s` with `ss`/`us`/`is` exclusions) **and Indonesian** (`-kan`, `-an`, `-nya`). Stems are stored under a `~` prefix at **0.65 ×** the base weight, and the *same* normaliser runs at query time — matching is symmetric, which is the property that makes stemming actually help rather than hurt.

**Scoring (`Search`):**

```
idf  = ln(1 + (N − df + 0.5) / (df + 0.5))
dlNorm = 1 − B + B × (tokenLength / avgDocLength)
term = idf × ((tf × (K1+1)) / (tf + K1 × dlNorm)) × queryTermWeight
queryTermWeight = 1.0 primary token · 0.65 stem token
```

Then the raw score is normalised into a comparable band:

```
normalized = saturation × 0.45 + relative × 0.30 + coverage × 0.25
             saturation = raw / (raw + 3.5)
             relative   = raw / maxRaw
             coverage   = matchedDistinctTerms / primaryQueryCount
```

Mixing saturation, relative rank, and term coverage prevents a single very-high-IDF rare term from dominating a multi-term question.

**Exact-phrase bonus:** for queries ≥ 8 characters with ≥ 2 tokens, an exact substring match in the normalised searchable text adds **+0.14**, capped at 0.99. This is what makes literal clause identifiers and invoice numbers rank top-1.

**Deletion is soft.** `RemoveDocument` flips `IsDeleted`, decrements `_activeCount`, subtracts `TokenLength` from the corpus total, and removes the entry from every posting map, dropping any posting list that becomes empty. Readers never see a torn list, and a document update never forces a full rebuild.

**Stopwords** cover both English function words and Indonesian function words, plus legal/document boilerplate (`section`, `party`, `shall`, `pursuant`, `hereof`, `provisions`) — these add IDF noise without adding discrimination.

---

## 8. Retrieval orchestration — Components A through H

`RagRetrievalOrchestrator` (`src/DocumentConverter.Infrastructure/AI/RagRetrievalOrchestrator.cs`) implements a lettered pipeline. Its contract types live in `DocumentIntelligenceDtos.cs`, and **each component is configured by a DTO** so ablations are configuration rather than forked code.

| Component | Responsibility | Configuration DTO |
|---|---|---|
| **A** | Query understanding | `QueryUnderstandingOptionsDto` |
| **B** | Concurrent multi-query retrieval | `MultiQueryRetrievalOptionsDto` |
| **C** | Reciprocal rank fusion + MMR | `RrfFusionOptionsDto` |
| **D** | LLM reranking | `LlmRerankOptionsDto` |
| **E** | Context expansion | `ContextExpansionOptionsDto` |
| **F** | Quality assessment + adaptive control | `AdaptiveRetrievalOptionsDto` |
| **G** | Grounded generation + citation verification | — |
| **H** | Observability trace | — |

### Component A — Query understanding

Enriches a raw question into intent, entities, technical terms, synonyms, relevant document categories, alternative search queries, required information, complexity, and cross-lingual requirements. The contract uses explicit `JsonPropertyName` bindings (`intent`, `entities`, `technical_terms`, `synonyms`, `relevant_document_categories`, `search_queries`, `required_information`, `query_complexity`).

**Two paths, one output shape:**

- **LLM path** — `SynthesizeStructuredJsonAsync` with a strict schema and a call timeout (default 12 s). Output passes `TryParseAndValidateQueryUnderstandingJson`, which requires an `intent`, clamps list sizes (entities ≤ 10, technical terms ≤ 10, synonyms ≤ 10, categories ≤ 5, queries ≤ `maxAlternativeQueries`, required info ≤ 8), normalises invalid complexity values to `single_fact`/`multi_fact`, and backfills missing queries/requirements.
- **Deterministic fallback** — `BuildDeterministicQueryUnderstanding`, always computed, always available.

When the LLM path succeeds, the two are **merged**: LLM synonyms ∪ local synonyms (≤ 18), LLM technical terms ∪ local terms (≤ 12), and — for document-overview questions — a scope/objective query is prepended. The system therefore gets LLM nuance *and* guaranteed lexical coverage.

The local synonym map (`DomainSynonymsMap`) is a bilingual, domain-tuned expansion table covering HR/leave, contracts and termination, financial/pricing/tax, SLA/security/operations, research methodology, and insurance/IFRS terminology. It is the single biggest reason cross-lingual retrieval works offline: `"sisa cuti"` expands to `remaining, unused, balance, carry-forward, accrued`.

### Component B — Multi-query retrieval

Executes up to `MaxTotalQueriesExecuted` (default 8) queries with `MaxConcurrency` (default 4) across **four independent strategies**:

| Source constant | Strategy |
|---|---|
| `HNSW_Vector` | Dense ANN search on each query variant |
| `BM25_Lexical` | Lexical scoring with field boosts |
| `Synthetic_Question` | Match against enrichment-generated questions |
| `Metadata_Entity` | Match against extracted entities and technical terms |

Each hit is a `MultiQueryCandidateHitDto` carrying `RetrievalSource`, `QueryInputType` (`Original_Question`, `Normalized_Question`, `Semantic_Expansion`, `Keyword_Expansion`, `Alternative_Question`, `Synthetic_Question_Match`), `RankInSourceList`, `RawScore`, and the canonical chunk identity.

**Canonical collapse happens here.** `ResolveCanonicalOriginalChunk` maps an enriched or translated chunk back to its original source chunk, so the same passage retrieved via three representations counts once — with the best score, and with `TranslatedContent` attached where a translation existed.

The HNSW/lexical hybrid score for a candidate is:

```
combined = clamp(max(normalized, normalized × 0.72 + hybrid × 0.28), 0, 0.99)
discard if combined < 0.12
```

i.e. the lexical/hybrid signal can only ever *raise* a dense score, never suppress it.

### Component C — Reciprocal rank fusion and MMR

```
contribution = sourceWeight × (1 / (K + rank)),  K = 60
```

Per-source weights are configurable (`SourceWeights`), resolved by exact key then by `::`-prefix, so a caller can weight `HNSW_Vector` above `Metadata_Entity` without touching code. Duplicates within a single source list are deduplicated so they cannot double-count.

The fused score is then normalised into a comparable band:

```
blendedNormalized = normalizedRatio × 0.65 + sourceConsensus × 0.15 + maxRawScore × 0.20
```

`sourceConsensus` is the fraction of active non-empty source lists that voted for the candidate. **This is what makes a cosine score and a BM25 score comparable without arbitrary weight tuning** — agreement across independent retrieval strategies is treated as evidence in itself.

**MMR diversification** (`ComputeRankFusionAsync` → MMR) then re-ranks by relevance minus redundancy:

```
mmr = λ × relevance − (1 − λ) × maxRedundancy,   λ = 0.72
```

with `MaxChunksPerDocument = 3` by default. Two refinements matter operationally:

- **Dominant-document detection** — if the top document's score leads the second by ≥ 0.14, it is treated as the dominant subject and its per-document cap is raised to `max(cap + 2, 5)`. Otherwise a single highly relevant document would be artificially starved of passages.
- **Diversity guard** — a candidate over its per-document cap is only skipped while *other* documents remain unrepresented, guaranteeing multi-document coverage without discarding genuinely superior single-document evidence.

When only one document is in scope, MMR is bypassed entirely.

### Component D — LLM reranking

`RerankCandidatesAsync` sends a batch (default ≤ 10 candidates) as structured JSON and receives `ranked_candidates` with `chunk_id`, `relevance_score`, and `relevance_reason`.

**The critical safeguard:** every returned `chunk_id` is validated against the **authorised candidate set**. A model that hallucinates an identifier cannot inject a passage into the result. Results are cached (`RerankCache`, keyed by question + candidate signature) so retries within the adaptive controller do not re-bill the provider. The minimum relevance threshold (default 0.16) prunes low-value candidates before expensive context expansion.

`RerankedCandidateDto` records `UsedExternalLlmReranker`, so a report can distinguish an LLM-reranked result from a fused-only result.

### Component E — Context expansion

Widens each surviving candidate into a self-contained passage:

| Option | Default | Effect |
|---|---|---|
| `MaxTotalWordsBudget` | 1600 | Hard ceiling on assembled context |
| `MaxNeighborWindowRadius` | 1 | Adjacent chunk on each side |
| `IncludeParentSection` | true | Parent section summary |
| `IncludeDocumentSummary` | true | Document executive summary |
| `IncludeRelatedSourceChunks` | true | Sibling representations of the same source |

`ExpandedContextChunkDto` preserves full provenance: `HeadingHierarchy`, `PrimaryContent`, `ExpandedContent`, `TranslatedContent`, `ParentSectionSummary`, `DocumentExecutiveSummary`, `IncludedNeighborChunkIndices`, `RelatedSourceChunkIds`, `WordCount`. The budget is enforced by word count, which keeps prompt cost predictable regardless of source document size.

### Component F — Quality assessment and adaptive controller

Assesses sufficiency as structured JSON: `sufficient_context`, `missing_information`, `evidence_coverage` (per-requirement, with an optional supporting chunk id), `recommended_action`, `has_conflicting_evidence`, `conflict_details`. The orchestrator then applies its **own** validation and records both `RecommendedAction` and `ValidatedAction`, so the decision trail shows whether the model's advice was followed.

Actions: `SEARCH_AGAIN`, `EXPAND_CONTEXT`, `GENERATE_ANSWER`, `INSUFFICIENT_EVIDENCE`.

The adaptive controller (defaults: `MaxAdditionalAttempts = 2`, total timeout 60 s, minimum evidence coverage 0.50, minimum top-candidate score 0.16) responds:

| Action | Adaptation |
|---|---|
| `SEARCH_AGAIN` | Prioritise `MissingInformation` queries **at the front** of the bounded query budget, add their synonyms, and add an offline-projected cross-lingual variant |
| `EXPAND_CONTEXT` | `neighborRadius + 1` (max 3), `wordBudget + 800` (max 3200) |
| `INSUFFICIENT_EVIDENCE` | Re-run uncached; after the final pass, abstain |
| Early stop | If a missing entity term has **zero occurrences** in the scoped corpus, retries cannot help — the controller stops immediately and records why |

That early-stop rule is a genuine cost and latency saver: it prevents burning two extra full retrieval passes on a question the corpus provably cannot answer.

### Component G — Grounded generation and citation verification

Every surviving citation is resolved through `VerifyAndResolveCitations`, which enforces two rules:

1. The citation's document id must be in the authorised set.
2. The citation must resolve to a real retrieved chunk — citations are produced *from* verified chunks, never parsed out of generated prose.

`VerifiedSourceCitationDto` carries `VerbatimSnippet`, optional `TranslatedSnippet`, `RelevanceScore`, and `RelevanceReason`.

**Abstention is a first-class outcome.** When citations are empty, the top similarity is below ~0.14, or the validated action is `INSUFFICIENT_EVIDENCE`, the service produces an explicit abstention message rather than a plausible-sounding answer. Confidence is set to 0.15 and citations are cleared.

### Component H — Observability

`RagRetrievalTraceDto` is the full audit record: query understanding, generated queries, candidate counts by source, fusion summary, rerank summary, context expansion summary, quality assessment, per-attempt traces (`RetrievalAttemptTraceDto`), retry decisions, total latency, external LLM latency, estimated tokens, final answer status, and verified citation count.

The most recent 100 traces are retained in a `ConcurrentQueue` and exposed read-only via `GetRecentRetrievalTraces`. Traces are **redacted for safe telemetry** — no raw document content is exported.

---

## 9. Provider-abstracted generation transport

`ExternalAiClient` is the **single outbound AI transport** in the entire module. Everything else is local computation. That is what makes the no-egress posture structural rather than aspirational.

### 9.1 Provider policy is configuration, not code

| Provider constant | Default model | Default endpoint |
|---|---|---|
| `OpenAI` | `gpt-4o-mini` | `https://api.openai.com/v1` |
| `Google Gemini` | `gemini-1.5-flash` | `https://generativelanguage.googleapis.com/v1beta` |
| `Anthropic Claude` | `claude-3-5-haiku-20241022` | `https://api.anthropic.com/v1` |
| `Custom / Local OpenAI-Compatible` | `llama3.2` | `http://localhost:11434/v1` |

The `Custom` entry points at a local Ollama-style runtime by default — a fully on-premise generation path with no internet egress at all.

### 9.2 Streaming normalisation

Three divergent dialects are flattened into one contract:

```csharp
public sealed record ExternalAiStreamChunk(
    string? ReasoningDelta = null,
    string? ContentDelta = null);
```

| Provider | Parser | Shape |
|---|---|---|
| OpenAI (+ Custom) | `TryParseOpenAiChunk` | SSE `data:` frames with `choices[0].delta` |
| Anthropic Claude | `TryParseClaudeSseChunk` | SSE `event:`/`data:` frame pairs |
| Google Gemini | `AppendGeminiElementChunks` | JSON arrays of `candidates[0].content.parts[]`, not line-delimited |

**A consumer never branches on provider.** It reads whichever delta is non-null.

**Reasoning isolation.** `ExtractOpenAiChoiceReasoning` reads `reasoning_content`, `reasoning`, or `thinking` from the delta/message object; Gemini's `thought: true` part flag routes a segment to `ReasoningDelta` instead of `ContentDelta`. Chain-of-thought tokens therefore never leak into the answer body, while remaining available for a UI that wants to show them.

**Partial-line buffering.** An SSE frame split across two TCP reads is carried forward rather than dropped. This is the detail that separates a streaming implementation that works from one that intermittently truncates tokens under load.

### 9.3 Resilience

- **Transport retry** with a 500 ms back-off on `429`, `502`, `503`, `504`.
- **Connection test** with a 12-second timeout that returns a structured `TestAiConnectionResponse` (success flag, human-readable message, measured latency) instead of throwing — including distinct messaging for timeout vs HTTP-status failure.
- **Embeddings** are batched (32 per request for OpenAI, `batchEmbedContents` for Gemini), L2-normalised on receipt, and re-ordered by the provider's `index` field so results never silently misalign with inputs.
- **Claude returns `null` for embeddings** — Anthropic has no native embeddings endpoint. Callers must handle this, and the pipeline does: neural embeddings are an enhancement over the deterministic 256-dimension baseline, never a requirement.

Error messages carry provider context so a mid-stream failure surfaces with enough information to diagnose.

---

## 10. Conversation state, memory and personalisation

### 10.1 Domain aggregates

| Aggregate | Bounded columns | Mutation surface |
|---|---|---|
| `CopilotChatSession` | `Title` 200, `FolderScopeName` 150 | `AssignUser`, `UpdateTitle`, `SetPinned`, `UpdateScope`, `Touch`, `AddMessage` |
| `CopilotChatMessage` | `Role` 20, `ModelUsed` 100, `QuestionAsked` 1000 | `MarkSavedToMemory` |
| `CopilotMemoryItem` | `Category` 80, `SourceSessionTitle` 200, `Fact` 2000 | `AssignUser`, `SetActive`, `UpdateFact` |
| `CopilotUserSettings` | `ResponsePersona` 100 | `Update(memoryEnabled, persona, instructions)` |

All four expose **private parameterless and `Guid` constructors for ORM rehydration only**. State change flows exclusively through intention-revealing methods that enforce invariants and update `UpdatedAtUtc`/`Touch` semantics. `CopilotChatSession` is the only entry point for thread mutation — messages are reachable only via the read-only `Messages` collection and are added through `AddMessage`.

`CopilotMemoryItem` carries `IsActive`, giving operators a **soft toggle** rather than forcing deletion of a learned rule.

### 10.2 Personas

```csharp
ExecutiveAnalyst   = "Executive Analyst"        // default
LegalCounsel       = "Legal & Risk Counsel"
FinancialAuditor   = "Financial Auditor"
ConciseSummary     = "Concise Bullet Summary"
```

Personas visibly change synthesis: the lead finding is labelled *Primary Contractual Provision & Liability Signal* for Legal Counsel, *Primary Financial & Billing Exposure* for Financial Auditor, and the Concise Summary persona switches to a bul lleted snippet list. Applied memories and custom instructions are rendered into the response under an *Applied AI Memory & Context* section, so the user can see exactly which stored rules influenced the answer.

### 10.3 Memory selection

`SelectRelevantMemories` picks the memory facts relevant to the current question and retrieved citations rather than injecting every stored fact. This keeps prompt cost bounded and prevents unrelated organisational rules from diluting an answer. Which memories were applied is persisted on the message (`AppliedMemoriesJson`), making personalisation auditable after the fact.

### 10.4 Multi-turn history handling

The synthesis prompt receives the prior turns, with a specific correction: if the last prior message is the *same* user question as the current turn (because of persistence timing), it is trimmed to avoid duplicating the question in context. When a genuine prior user turn exists, the response opens with a *Continuing analysis from prior turn ("…")* line so the conversation reads coherently.

### 10.5 Persistence configuration

`CopilotConfigurations` implements all four `IEntityTypeConfiguration<T>` interfaces in one sealed class:

| Table | Key relationships | Indexes |
|---|---|---|
| `CopilotChatSessions` | `User` (Restrict), `Folder` (SetNull), `Messages` (Cascade, `PropertyAccessMode.Field`) | `(UserId, UpdatedAtUtc)`, `IsPinned`, `FolderScopeId` |
| `CopilotChatMessages` | via `SessionId` | `(SessionId, TimestampUtc)` |
| `CopilotMemoryItems` | `UserId` | `(UserId, IsActive)` |
| `CopilotUserSettings` | `UserId` | unique `UserId` |

Column constraints mirror the domain length constants exactly, and JSON payload columns (`CitationsJson`, `SuggestedFollowUpsJson`, `AppliedMemoriesJson`) are stored without a length cap. Persistence is configured **outside** the domain, so the aggregates stay plain and the mapping stays swappable.

> **Operational note.** The service also maintains in-memory `ChatSessionsStore` / `MemoriesStore` with a `SemaphoreSlim` write lock and mirrors state to a JSON file under `app_data/copilot/copilot_state.json`. The EF-mapped tables are the durable system of record; the in-memory store is the low-latency read path.

---

## 11. Persistence schema

Two tables carry the retrieval substrate (`init-postgres.sql`):

```sql
"DocumentIntelligenceIndices"
  DocumentId uuid UNIQUE → Documents(Id) ON DELETE CASCADE
  DocumentCategory        varchar(120) NOT NULL
  CategoryConfidence      real         NOT NULL
  ExecutiveSummary        text         NOT NULL
  KeyEntitiesJson         text         NOT NULL
  KeyTopicsJson           text         NOT NULL
  TotalWordCount, TotalPages, ChunkCount  int
  AnalyzedAtUtc           timestamptz

"DocumentVectorChunks"
  Id uuid PK  ·  DocumentId uuid → Documents(Id) ON DELETE CASCADE
  ChunkIndex int  ·  PageNumber int  ·  SectionTitle varchar(300)
  Content text  ·  Embedding bytea
  Language varchar(10) DEFAULT 'orig'
  ParentChunkId uuid  ·  IsEnriched bool DEFAULT FALSE
  VectorSource varchar(80) DEFAULT 'Actual Document'
  ExtractionMethod varchar(100) DEFAULT 'MarkItDown'
  EnrichmentMetadataJson text  ·  CreatedAt timestamptz
```

Indexes are chosen for the access patterns actually used:

| Index | Serves |
|---|---|
| `DocumentVectorChunks(DocumentId)` | Rehydrate one document |
| `DocumentVectorChunks(DocumentId, ChunkIndex)` | Ordered replay |
| `DocumentVectorChunks(DocumentId, IsEnriched)` | Provenance-filtered hydration |
| `DocumentVectorChunks(DocumentId, Language)` | Language-scoped retrieval |
| `DocumentVectorChunks(ParentChunkId)` | Translation ↔ source resolution |
| `DocumentIntelligenceIndices(DocumentId)` **unique** | One intelligence row per document |

**Both tables cascade on document delete**, so purging a document removes its vectors. Reconciliation with the in-memory indexes is performed by `EnsureUserHasDocumentAccessAsync` filters combined with `RemoveDocument` on both structures — and `HydrateCorpusVectorIndexAsync` only loads documents with `Status == Available`, so deleted or unavailable documents never re-enter the graph at startup.

---

## 12. HTTP and UI surface

### 12.1 Endpoints (`DocumentIntelligenceEndpoints.cs`)

All routes sit under `RequireAuthorization()`.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/documents/{id}/ai/insights` | Category, summary, entities |
| `POST` | `/api/documents/{id}/ai/ask` | Single-document RAG Q&A (citations clamped 1–20, min confidence clamped 0–1) |
| `GET` | `/api/documents/{id}/ai/search` | In-document semantic search (`topK` default 5) |
| `GET` | `/api/documents/ai/search` | Repository search, optional `folderId` (default `topK` 10) |
| `POST` | `/api/documents/ai/ask-global` | Cross-repository Q&A (default `topK` 6) |
| `GET` | `/api/documents/ai/corpus-stats` | Corpus metrics and category distribution |
| `POST` | `/api/documents/ai/prewarm` | Warm the full corpus vector cache |
| `POST` | `/api/documents/ai/compare` | Multi-document comparison matrix (requires ≥ 2 ids) |
| `POST` | `/api/documents/ai/export-dossier` | PDF / DOCX / XLSX / Markdown download |
| `GET` | `/api/documents/ai/knowledge-graph` | Entities, topic cloud, financial obligations |
| `POST` | `/api/documents/ai/faceted-search` | Faceted semantic search |
| `GET`/`POST`/`DELETE` | `/api/documents/ai/chat/sessions[/{id}]` | Session listing, retrieval, deletion |
| `POST` | `/api/documents/ai/chat/message` | Send a message with memory + persona |
| `GET` | `/api/documents/ai/memory` | Memory profile |
| `POST` | `/api/documents/ai/understand-query` | Component A standalone |
| `GET` | `/api/documents/ai/traces` | Component H traces (default 20) |
| `POST` | `/api/documents/ai/benchmark` | **Administrator only** |
| `GET`/`POST` | `/api/ai/config` | Read (masked key) / write config — **write is Administrator only** |
| `POST` | `/api/ai/test-connection` | **Administrator only** |

**Endpoint discipline:** handlers bind a contract, call one service method, and map the result. No business logic lives at the HTTP layer. Input validation is minimal and deliberate — non-empty question, non-empty query, ≥ 2 documents for comparison — with all clamping delegated to the service.

**Key handling:** `GET /api/ai/config` never returns the stored API key. It returns a masked form (`first4...last4`, or `********` when short) so configuration can be inspected without credential exposure.

### 12.2 UI surface

| Component | Route | Notes |
|---|---|---|
| `Copilot.razor` | `/copilot` | `[Authorize]`, `IDisposable`; the largest component in the directory (~3,100 lines), reflecting substantial state management across scope selection, comparison, knowledge graph, faceted search, export and chat |
| `AiChatPage.razor` | `/ai-chat` | Thin shell hosting the enterprise chat workspace |
| `EnterpriseAiChatService` | — | Bridges `Radzen.Blazor.ChatMessage` to the application message model, holds session state in `ConcurrentDictionary`, manages folder/document/hybrid scoping, and caches reasoning separately from visible text |

`EnterpriseAiChatService` caching reasoning separately is what allows the UI to render or suppress chain-of-thought independently of the answer body — the same separation the transport enforces with `ReasoningDelta` / `ContentDelta`.

---

## 13. The quality gate

`RagBenchmarkRunner` is not a demo harness — it is the merge gate.

### 13.1 Deterministic corpus and questions

- **10 enterprise documents**, each with sections, executive summary, entities, synthetic questions, key facts, and technical terms.
- **11 questions per document (110 answerable)** covering categories: `exact_identifier`, `colloquial_synonym`, `multi_fact_neighbor`, `deep_chunk_6_to_9`, and `cross_lingual_id_en`.
- **10 unanswerable guardrail questions** → **120 total**, matching the `Math.Clamp(count, 10, 200)` bound.
- **Deterministic identities:** `CreateDeterministicGuid(major, minor)` derives stable GUIDs from integer pairs, so the same evaluation run produces byte-identical IDs on any machine. Metric deltas between runs are therefore attributable to the code change and nothing else.

The corpus deliberately includes a **bilingual Indonesian/English contract** with `TranslatedEnglish` representations, so cross-lingual retrieval is measured, not assumed.

### 13.2 Five stage modes

| Stage | Configuration |
|---|---|
| `1_Baseline_SingleQuery_Top5_Vector` | Single-query vector search, no fusion, no expansion |
| `2_ComponentAB_QueryUnderstanding_MultiQuery` | + Components A and B |
| `3_ComponentABC_ReciprocalRankFusion` | + Component C (RRF k=60, MMR) |
| `4_ComponentABCDE_Reranked_And_ContextExpanded` | + Components D and E |
| `5_Full_Orchestrated_Pipeline_ABCDEFG` | + quality assessment, adaptive retries, citation verification |

Because each mode is a configuration set, **ablation is a parameter, not a branch.**

### 13.3 Metrics

| Retrieval | Generation |
|---|---|
| Recall@5, Recall@10 | Context recall |
| Precision@5 | Context precision |
| MRR | Faithfulness |
| nDCG@10 | Citation accuracy |
| Mean end-to-end latency | Estimated LLM API cost (USD) |

Verification honesty: stages 2–4 force `EnableExternalLlmWhenConfigured: false`, so those numbers measure the **deterministic** pipeline. Stage 5 exercises the full adaptive path. Cloud usage inside the benchmark is opt-in via `allowCloudInBenchmark`, and when disabled the config is transformed with `CloudEnabled = false` rather than being trusted to behave.

A change that improves Recall@5 while regressing Faithfulness is visible in the same report — which is the entire point of running the stages together.

---

## 14. Deliverable export

`CopilotDossierExporter` renders analysis into artefacts stakeholders actually consume.

| Format | Path |
|---|---|
| Markdown | `BuildDossierMarkdown` → UTF-8 bytes, `text/markdown` |
| DOCX | `BuildWordDossier` — OpenXML `Wordprocessing` tree built directly via `AppendWordHeading`, `AppendWordParagraph`, `AppendWordTableRow` |
| XLSX | `BuildExcelDossier` — OpenXML `Spreadsheet` tree via `AppendExcelRow` |
| PDF | Markdown → temp `.md` → `MarkdownHtmlPdfConverter` → PDF bytes → temp files deleted in a `finally` block |

`ExportFinancialGraphToExcel(EnterpriseKnowledgeGraphDto)` produces a three-sheet workbook:

1. **Financial obligations** — counterparty, amount, parsed value, currency, key date, obligation type
2. **Extracted entities** — category, type, value, document count, referencing documents
3. **Corpus topics** — topic and document frequency, ordered descending

Filenames are sanitised and timestamped (`yyyyMMdd-HHmm`), and all numeric formatting goes through `CultureInfo.InvariantCulture` so an exported workbook is locale-independent.

Export request types accept an optional `ComparisonResult` and `SearchResults`, so a single dossier can carry an answer, its citations, a comparison matrix, and raw search results.

---

## 15. Security posture

### 15.1 Zero egress by default — structurally, not by policy

| Component | Network dependency |
|---|---|
| HNSW index | **None** — in-process over locally held vectors |
| BM25 index | **None** — in-process over locally held tokens |
| Language detection | **None** — stopword corpora |
| Offline translation | **None** — dictionary projection |
| Chunk embedding (256-dim) | **None** — feature hashing in-process |
| OCR | **None** — local Tesseract |
| `ExternalAiClient` | **The only network path in the module** |

There is no vector database and no search cluster to reach. Because the single egress point is one class, the no-egress posture can be enforced by configuration rather than by auditing call sites.

The `docker-compose.yml` worker placement reflects this: `document-worker` is attached to `dc-internal` and `web-gateway` only, with an explicit comment that it has **zero external Internet egress**, while the web host also joins `dc-public`.

### 15.2 Access enforcement at three layers

| Layer | Enforcement |
|---|---|
| **Routing** | `RequireAuthorization()` on the endpoint group; `RequireRole("Administrator")` on benchmark and AI-configuration writes |
| **Service** | `EnsureUserHasDocumentAccessAsync` before any content is read; throws `UnauthorizedAccessException` |
| **Retrieval** | Explicit rejection of candidates failing document-authorisation or provenance checks — *rejected, not silently filtered*, so a caller can distinguish an empty corpus from a denied one |

Repository search applies strict user scoping (`OwnerUserId == currentUserId`) and returns **empty** rather than falling back to the unscoped corpus when the authorised scope is empty. There is no "widen the search" fallback that could leak cross-tenant content.

### 15.3 Prompt-injection resistance during enrichment

Chunk enrichment treats source text as **untrusted data**. `SanitizeUntrustedChunkData` scans for injection patterns and replaces matches with `[UNTRUSTED_DIRECTIVE_REDACTED]`, and the result is flagged on the metadata as `SanitizedUntrustedDirectives`. Enrichment follows a documented 10-rule discipline, of which the security-relevant ones are: treat source as data not instructions; extract only explicitly stated facts paired with verbatim `SourceEvidence`; state ambiguities rather than inferring; never repeat the entire source in the summary.

Any directive embedded inside a document is neutralised before it can reach a prompt.

### 15.4 Telemetry hygiene

`RagRetrievalTraceDto` is explicitly documented as redacted for safe telemetry. Traces expose counts, scores, decisions and timings — **not** raw document content.

---

## 16. Operational reference

### 16.1 Configuration knobs

| Setting | Default | Location |
|---|---|---|
| Embedding dimensions | 256 (1024 bytes) | `DocumentIntelligenceService.EmbeddingDimensions` |
| Channel capacity | 10,000 | `DocumentIndexingChannel` |
| HNSW `m` / `efConstruction` | 16 / 64 | `HnswVectorIndex` constructor |
| HNSW `efSearch` | 32 (wider when filtered) | `Search` parameter |
| BM25 `K1` / `B` | 1.2 / 0.75 | `Bm25InvertedIndex` |
| RRF `K` | 60 | `RrfFusionOptionsDto` |
| MMR `λ` / chunks per doc | 0.72 / 3 | `RrfFusionOptionsDto` |
| Query understanding queries / timeout | 4 / 12 s | `QueryUnderstandingOptionsDto` |
| Multi-query concurrency / candidates | 4 / 15 | `MultiQueryRetrievalOptionsDto` |
| Rerank batch / threshold / timeout | 10 / 0.16 / 6 s | `LlmRerankOptionsDto` |
| Context budget / neighbour radius | 1600 words / 1 | `ContextExpansionOptionsDto` |
| Adaptive attempts / timeout / coverage | 2 / 60 s / 0.50 | `AdaptiveRetrievalOptionsDto` |
| Traces retained | 100 | `RagRetrievalOrchestrator` |
| Chat reasoning cache | 500 entries | `EnterpriseAiChatService` |
| Benchmark question count | 120 (clamped 10–200) | `RunBenchmarkAsync` |
| Provider config | BYOK, per-deployment | `AiProviderConfigDto` |

### 16.2 Where to extend

| To add… | Touch |
|---|---|
| A retrieval strategy | `RetrievalStrategySources` + a new branch in Component B |
| A provider | `AiProviders`, `GetDefaultModel`, `GetDefaultEndpoint`, plus a parse/stream branch in `ExternalAiClient` |
| A persona | `CopilotPersonas` + the persona switch in local synthesis |
| A document type | `ExtractPagesFromStream` dispatch |
| A benchmark category | A new `QuestionTemplates` entry in a `BenchmarkDocumentSpec` |
| An export format | `CopilotDossierExporter.ExportDossierAsync` format switch |
| A synonym domain | `DomainSynonymsMap` (orchestrator) and `IdToEnDictionary`/`EnToIdDictionary` (chunker) |

### 16.3 Failure modes and designed responses

| Failure | Response |
|---|---|
| PDF/DOCX extraction fails | OpenXML / PdfPig fallback; page recorded as empty rather than aborting |
| Scanned page, OCR budget exhausted | Page indexed with whatever selectable text exists |
| Neural translation batch fails ×3 | Stop batches, keep partial results, project remaining offline |
| Neural embeddings unavailable (e.g. Claude) | Fall back to deterministic 256-dim vectors |
| LLM query understanding times out | Deterministic analyser runs; result merged only if valid |
| LLM reranker hallucinates a chunk id | Id fails authorisation validation; candidate dropped |
| Quality assessment says insufficient | Bounded retries, then explicit abstention |
| Query vector dimension mismatch | Search rejected for that dimension, not mis-scored |
| Tight scope yields too few ANN hits | Targeted brute-force escalation over ≤ 24 documents |
| One document corrupt during background indexing | Per-item catch; consumer loop survives |
| Cold start | Hydration from PostgreSQL rebuilds HNSW and BM25 without re-parsing storage |
| Deleting a document | Cascade delete of chunks and index row; `Status == Available` filter excludes it from hydration |

---

## 17. Summary

Document Intelligence and RAG converts a document archive into verified, cited answers while keeping the corpus on-premise by construction. Its design invariants are:

1. **Indexing never blocks the user.** A bounded channel, a hosted consumer, and per-item isolation turn a heavy operation into a background concern.
2. **Every retrieval atom carries its provenance.** `VectorSource`, `ExtractionMethod`, `Language`, `ParentChunkId`, and page number survive every transformation and reach the citation intact.
3. **Two complementary indexes beat one.** Dense HNSW handles paraphrase; BM25 handles identifiers, clause numbers and figures. Reciprocal rank fusion makes their scores comparable without weight guessing.
4. **Approximation is always backed by a fallback.** Dimension-scoped search, brute-force escalation on tight scopes, and adaptive retries mean "not found" is a measured conclusion, not an artefact of the approximate index.
5. **Abstention is a valid answer.** Insufficient evidence produces an explicit message rather than a confident fabrication.
6. **Third-party AI is a single, optional chokepoint.** One class, four providers, one normalised stream contract — disable it and the system still indexes, retrieves, ranks and answers deterministically.
7. **Quality is measured, not asserted.** A 120-question deterministic benchmark with five stage modes gates every retrieval change.
8. **Access is enforced three times, and failures are loud.** Routing, service, and retrieval each check; denied candidates are rejected rather than quietly dropped.

The net effect is a retrieval system whose default posture requires no external service, whose answers can be traced back to a page in a specific document, and whose behaviour is measurable run over run.