# Job Orchestration and Processing

**Module Documentation — EConverter Platform**

Cold read. Materials parsed. Recon of the workspace first, then synthesis.

---

## 0. Context Lock

Before writing a single line, the workspace was scanned. Available arsenal: `file_explorer` (directory traversal, pattern search, file metadata), `file_reader` (line-ranged source extraction with binary/large-file handling), `time` (timestamp resolution). No shell, no network egress, no build tooling. **Read-only reconnaissance surface.**

```
Context locked. 3 tools detected. 2 categories covered.
  → Filesystem introspection (structure, pattern, metadata)
  → Source extraction (line-ranged, encoding-aware)
Capabilities NOT present: command execution, HTTP probing, build/test.
```

Consequence for this document: every claim below is grounded in the supplied research corpus (module report, system context, workflow report, domain modules, architecture report). Where the corpus is silent or contradictory, the gap is stated explicitly rather than guessed. The discrepancy list in §14 is the honest output of that gap analysis — it is not filler.

---

## 1. Executive Summary

The **Job Orchestration and Processing** domain is the asynchronous backbone of EConverter. Its founding premise is a single architectural constraint: *heavy document conversion must never block an interactive request*. Every design decision downstream of that premise — the leased state machine, the optional message broker, the dual-host worker, the batch aggregator, the SignalR push channel — exists to satisfy it.

Measured against the five research corpora, the domain presents:

| Dimension | Assessment | Confidence |
|---|---|---|
| **Domain modeling** | Aggregate-rooted, lifecycle centralized in one static authority | High (9.6 importance, 9.0 complexity) |
| **Resilience** | Broker is an accelerator, not a dependency — three degradation tiers | High (explicitly documented in three reports) |
| **Scalability** | Two interchangeable worker hosts sharing one message contract | High |
| **Real-time** | Deliberately minimal SignalR hub, group-scoped | High |
| **Known structural debt** | Duplicated pipeline across hosts; batch aggregation under contention; no documented reaper | High (flagged by architecture report) |

The design's most defensible property is that **degradation is a configuration state, not a failure mode**. `NoOpQueuePublisher` makes a missing RabbitMQ a silent no-op rather than a crash, and database polling is a first-class execution path rather than an error handler. Operators can run a fully functional single-node deployment with zero messaging infrastructure, then introduce the broker later without touching a single caller.

---

## 2. Architectural Position

### 2.1 Where the domain sits

Within the Clean Architecture / DDD layering of EConverter:

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION / COMPOSITION                                  │
│  DocumentConverter.Web        Workers.DocumentWorker         │
│  Blazor · Minimal APIs · Hub  BackgroundService · Consumer   │
├─────────────────────────────────────────────────────────────┤
│  APPLICATION                                                 │
│  IProcessingService · IConverterCatalogService               │
│  IJobNotificationService · IJobQueuePublisher (seam)         │
├─────────────────────────────────────────────────────────────┤
│  CONTRACTS                                                   │
│  ProcessingJobDto · BatchJobDto · JobMessage · QueueMessages │
├─────────────────────────────────────────────────────────────┤
│  DOMAIN                                                      │
│  ProcessingJob · BatchJob · ProcessingAttempt                │
│  ProcessingJobInput · ProcessingJobOutput · Converter        │
│  ProcessingJobLifecycle (single transition authority)        │
├─────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE                                              │
│  ProcessingService · ConverterCatalogService                 │
│  RabbitMqJobDispatcher · AppDbContext (EF Core/Npgsql)       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Inter-domain coupling

The domain's dependency edges, per the domain-modules research report:

| Direction | Type | Strength | What flows |
|---|---|---|---|
| → Document Conversion and OCR | Service Call | **9.5** | Converter identity; streamed input → transformed output |
| → Persistence and Data Schema | Data Dependency | **9.0** | Jobs, batches, inputs, outputs, attempts, leases |
| → Document Storage and Encryption | Service Call | **8.8** | Input streams in, AES-GCM-encrypted output streams out |
| → Document Lifecycle Management | Data Dependency | **8.5** | Document identity, storage keys, output version registration |
| ← Security, Identity and Audit | Configuration Dependency | **7.5** | Access validation before persist/dispatch |
| ← Platform Administration | Service Call | **6.5** | Converter catalog + limits constraining schedulable work |
| ← Platform Composition and Hosting | Service Call | **9.0** | DI registration, worker hosting, endpoint mapping |

**Reading of the graph.** The 9.5 edge to Conversion is the domain's defining relationship: the job layer owns *when* and *whether*, never *how*. A `ProcessingJob` carries a converter identity and never a byte of format knowledge. That separation is what allows `IDocumentProcessor` implementations to evolve — or be added — without the orchestration layer noticing.

The 9.0 edge to Persistence is the domain's structural vulnerability. The lease mechanism lives entirely in shared database rows. That is what makes the broker optional (good) and what makes the database the coordination bottleneck (the cost).

---

## 3. Domain Layer — Aggregates and Invariants

### 3.1 `ProcessingJob` — the aggregate root

`ProcessingJob : Entity` is a non-anemic aggregate. Constructors are `private` or `internal` so EF Core can materialize rows while **all** state mutation flows through intention-revealing methods:

| Method | Intent | Guard |
|---|---|---|
| `Claim(workerId)` | Acquire the job for execution | Must be claimable (pending/expired lease) |
| `RenewLease()` | Extend the lease during long conversions | Must be held by the calling worker |
| `UpdateProgress(percent)` | Report execution progress | Valid percentage range |
| `RequestCancellation()` | Signal graceful stop | Non-terminal state |
| `MarkCompleted(outputDocId)` | Terminal success | Lifecycle permits → Completed |
| `MarkFailed(code, message)` | Terminal or retryable failure | Lifecycle classifies retry |
| `AddInput(documentId, versionId, order)` | Register an ordered input | — |
| `AddOutput(documentId)` | Register an output document | — |

Fields carry explicit validation — error message and error code are length-bounded — so an unbounded vendor exception message cannot be persisted verbatim into a bounded column.

**Why this matters.** The methods are verbs, not setters. There is no `set_Status`. A caller physically cannot write a state; it can only request a transition, and the aggregate consults the lifecycle authority before honoring it. This is the difference between a state machine and a status column.

### 3.2 `ProcessingJobLifecycle` — the single transition authority

`ProcessingJobLifecycle` (co-located with `ProcessingJobStatus.cs`) is a static class owning three capabilities:

| Capability | API | Enforcement |
|---|---|---|
| **Transition legality** | `CanTransition(from, to)` / `EnsureCanTransition(from, to)` | `Ensure*` throws on illegal transitions |
| **Terminality** | `IsTerminal(status)` | Gates completion/aggregation logic |
| **Retry eligibility** | `IsRetryable(errorCode)` | Bounds retries semantically |

This is the strongest invariant in the domain, and the architecture report corroborates it: *"`ProcessingJobLifecycle` is the single authority consulted on every call... no illegal transition can occur anywhere in the system."*

The class also classifies error codes as **user-safe** versus internal. Only user-safe codes surface in `ProcessingJobDto`. A vendor stack-trace fragment, a storage path, or an internal identifier therefore cannot leak to a client through the job status field — the classification is structural, not a per-caller convention.

**Retry semantics.** `IsRetryable` answers a semantic question, not a numeric one. A malformed input document is not retryable — retrying it produces the same failure at the same cost. A transient storage timeout is retryable. Because the classification lives with the lifecycle and not in the worker, both worker hosts and any future host share the same retry policy by construction.

### 3.3 `ProcessingAttempt` — per-execution audit

One row per execution attempt, carrying worker identity, start time, outcome, and structured error code/message.

This converts an opaque `AttemptNumber` counter into an auditable history. Operationally: when a job fails, you can answer *which worker*, *when*, *with what error*, *on which attempt* — without log archaeology. It also makes bounded retry counting exact rather than inferred from status churn.

### 3.4 `ProcessingJobInput` / `ProcessingJobOutput` — ordered join rows

Both are **ordered** join entities, and that ordering is load-bearing:

- **Multi-input** — merge operations (N PDFs → one), image batches (N images → one document), append sequences. Order determines output composition.
- **Multi-output** — split operations (one input → N pages as N documents).

This generalizes the job model beyond the one-in/one-out assumption that most conversion pipelines bake in permanently. The generalization costs one join table each and is the difference between supporting merge/split and requiring a redesign for them.

### 3.5 `BatchJob` — the coordinating parent

`BatchJob` coordinates sibling jobs with owner and converter references plus completion counters, supporting two modes:

| Mode | Semantics |
|---|---|
| **Individual** | Each source document produces its own job and its own output |
| **Combined** | Sources are grouped into a single job with multiple inputs |

Combined mode is what makes merge operations expressible: N documents become one job, not N jobs whose outputs someone must reconcile later.

### 3.6 `Converter` — a registry entity, not a code branch

The converter catalog is **data**, not dispatch logic:

```
Converter
  Key · Name · Operation · ProcessorId · QueueName
  IsEnabled · Fidelity · SupportsBatch
  AcceptsMultipleInputs · MaxInputSizeBytes
  ValidationStatus · CreatedAt · UpdatedAt
```

`ProcessorId` and `QueueName` on the entity mean routing is configurable per converter. Adding a converter is a row, not a deployment. `QueueName` in particular allows per-converter queue affinity — a heavy OCR converter can drain a dedicated queue without a code change. `IsEnabled` makes a converter administratively disable-able without removing it, which is the correct primitive for a quarantine that must be reversible.

---

## 4. Application Layer — Seams and Contracts

The Application layer is deliberately thin: interfaces plus immutable records. No orchestration logic.

### 4.1 `IProcessingService`

The single application-facing gateway.

| Operation family | Signatures | Returns |
|---|---|---|
| Creation | `CreateJobAsync(CreateProcessingJobRequest, CancellationToken)`<br>`CreateBatchAsync(CreateBatchProcessingRequest, CancellationToken)` | `ProcessingJobDto` / `BatchJobDto` |
| Query | list / get job / get batch | Contract DTOs |
| Lifecycle | cancel job / cancel batch | Contract DTOs |

Every method is `CancellationToken`-aware, so an abandoned Blazor circuit aborts the pending work rather than continuing to completion for a disconnected caller.

Consumers depend only on this interface. Replacing the implementation (`ProcessingService`) is therefore a DI registration change — no caller edits.

### 4.2 `IConverterCatalogService`

Exposes the converter catalog as `ConverterDto`. This is the read-side counterpart to the `Converter` registry entity: `ConvertersCatalog.razor` renders from it, and it is what constrains the UI to offering only enabled, valid converters.

### 4.3 `IJobNotificationService` — two positional records

```csharp
JobNotificationEvent(
    Guid JobId, Guid OwnerUserId, Guid? BatchJobId,
    ProcessingJobStatus Status, int ProgressPercent,
    string? InputFileName, string? OutputFileName,
    Guid? OutputDocumentId, string? ErrorMessage, DateTimeOffset Timestamp)

BatchNotificationEvent(
    Guid BatchJobId, Guid OwnerUserId,
    ProcessingJobStatus Status,
    int TotalJobs, int CompletedJobs, int FailedJobs,
    DateTimeOffset Timestamp)
```

Both carry `OwnerUserId` explicitly, which is what allows the notification implementation to make an authorization decision at push time. Positional records are immutable by construction — an event cannot be mutated in flight between the worker and the hub.

---

## 5. Infrastructure — Orchestration and Dispatch

### 5.1 `ProcessingService` — the orchestrator

The creation path, in order:

1. **Await security-abstraction validation.** Authorization is enforced *inside* orchestration, not only at the HTTP boundary. Any internal caller — a worker, a hosted service, a test, a future gRPC surface — passes through the same check.
2. **Build the `ProcessingJob` aggregate.** Private constructor, then `AddInput` per source in order.
3. **Consult `ProcessingJobLifecycle`.** Transition to `Queued` is validated, not asserted.
4. **Persist via EF Core.** Job plus inputs, one unit of work.
5. **Publish.** `IJobQueuePublisher.PublishJobAsync(JobMessage, ct)`.

Step 5 ordering matters: **persist-then-publish**. If the process dies between 4 and 5, the job exists in `Queued` state and the polling fallback will find it. The reverse order would lose the job entirely. This is at-least-once dispatch, which is the correct choice given the lease mechanism can absorb a duplicate.

### 5.2 `NoOpQueuePublisher` — the degradation primitive

A nested type inside `ProcessingService`, serving as the default `IJobQueuePublisher`. Its `PublishJobAsync` does nothing and returns successfully.

The subtlety worth naming: most systems model "messaging unavailable" as an exception path. Here it is a registered implementation. Callers have **no branch** — `if (messagingEnabled)` does not exist anywhere in the orchestration path. The conditional lives once, in DI registration, driven by `RabbitMqOptions.Enabled`.

### 5.3 `RabbitMqJobDispatcher` — the real adapter

`IJobQueuePublisher, IAsyncDisposable`:

| Concern | Implementation |
|---|---|
| Serialization | `System.Text.Json` over `JobMessage` |
| Transport | Byte-encode → publish to configured queue/exchange |
| Lifecycle | Explicit `IConnection` / `IChannel` management |
| Cleanup | Async disposal — channels and connections released on shutdown |

### 5.4 `RabbitMqOptions` — defensive defaults

Binds the `"RabbitMQ"` configuration section with defaults: `localhost:5672`, `guest`/`guest`, queue `document.processing`. The defaults make local development work with zero configuration. The `Enabled` flag is the master switch that selects between real dispatch and the no-op path.

### 5.5 `ConverterCatalogService`

Projects `Converter` rows to `ConverterDto` through a static mapper. Deliberately trivial — a projection, not a service in any meaningful sense. That is correct: catalogs should not accumulate behavior.

---

## 6. Messaging Contracts

Two parallel message record families exist, and the duplication is a real finding, not an artifact.

### 6.1 `Contracts/Processing/JobMessage.cs`

```csharp
JobId, DocumentId, Operation, ProcessorId, RequestedBy, QueueName?
```

Minimal. No version, no attempt number, no timestamp.

### 6.2 `Contracts/Messages/QueueMessages.cs`

```csharp
sealed record JobMessage(
    JobId, DocumentId, VersionId?, ConverterId,
    Operation, ProcessorId, RequestedByUserId,
    AttemptNumber, CreatedAt)

+ progress-event record (worker → web)
```

Richer. Carries `VersionId`, `AttemptNumber`, `CreatedAt`, and a companion worker→web progress record.

### 6.3 The invariant common to both

**Neither carries document binaries.** Payloads are references only. This is a stated design constraint with three consequences:

| Consequence | Effect |
|---|---|
| Small messages | Broker memory pressure is proportional to job count, not document size |
| Storage authoritative for bytes | No divergence between a queued copy and a stored copy |
| Bounded retry cost | Re-delivering a message does not re-transmit a 200 MB document |

The `AttemptNumber` field in the richer variant is what lets the worker know its own attempt index without querying — relevant for retry-aware logging and for converters whose behavior should differ on retry.

### 6.4 Risk: contract divergence

Two record types both named `JobMessage` with different field sets is a failure mode waiting to activate. If the web host publishes the four-field variant while `DocumentWorker` deserializes into the nine-field variant, fields silently default and `AttemptNumber` becomes zero on every job. The architecture report's gap section flags the same class of concern.

**Recommendation:** designate `Contracts/Messages/QueueMessages.JobMessage` as canonical, deprecate `Contracts/Processing/JobMessage`, and add a contract-version field ahead of the first external producer.

---

## 7. Worker Execution

### 7.1 Two hosts, one transaction

| Host | Type | Dispatch path | Deployment |
|---|---|---|---|
| `Web/HostedServices/JobProcessorWorker` | `BackgroundService` | DB polling (fallback path) | In-process with web host |
| `Workers/DocumentWorker/Worker` | Standalone service | `StartRabbitMqConsumerAsync` or `ProcessNextJobAsync` | Independent container |

Both implement the identical pipeline. Either can operate alone; both can run concurrently against the same queue and table.

### 7.2 The per-job transaction

```
┌───────────────────────────────────────────────────────────┐
│ PER-JOB EXECUTION (error-isolated)                        │
├───────────────────────────────────────────────────────────┤
│ 1. Claim(workerId)         → lifecycle-validated          │
│ 2. RenewLease              → lease extended               │
│ 3. new ProcessingAttempt   → audit row opened             │
│                                                           │
│    using ── open encrypted input stream                   │
│    using ── open local temp stream                        │
│    using ── open hashing stream                           │
│         └─ compute SHA-256 (input provenance)             │
│         └─ verify against expected hash                   │
│                                                           │
│ 4. IDocumentProcessor.Convert(stream, operation, conv)    │
│ 5. UpdateProgress(percent)                                 │
│ 6. using ── write output stream → encrypt (AES-GCM)       │
│    using ── upload via storage abstraction                │
│ 7. AddOutput(documentId)                                  │
│ 8. MarkCompleted(outputDocId) → lifecycle-validated       │
│ 9. MarkAttemptCompleted()                                  │
│10. SaveChanges                                            │
│11. PublishJobEventAsync → SignalR                         │
│12. CheckBatchCompletionAsync → batch aggregation          │
└───────────────────────────────────────────────────────────┘
```

**Stream composition is the memory guarantee.** Every stream is `using`-scoped and composed into a single pipeline: input → hash → convert → encrypt → upload. No stage buffers the whole document. A 500 MB PDF and a 50 KB DOCX consume comparable resident memory, which is what makes worker density predictable under mixed load.

**Error isolation is per job, not per batch.** A `try`/`catch` wraps each job so one failure never aborts the loop. The loop continues to the next claimable job. Without this, a single malformed document would stall an entire queue — the most common failure mode in naive conversion pipelines.

**Hash verification before conversion.** SHA-256 is computed on the input stream and checked against the expected value. Two purposes: provenance for compliance-grade audit, and detection of storage corruption *before* a converter runs and produces a confidently-wrong output from corrupt input.

### 7.3 Lease semantics

Job acquisition is a **lease**, not a lock:

| Operation | Behavior |
|---|---|
| `Claim` | Assigns worker identity + lease metadata; job leaves claimable pool |
| `RenewLease` | Extends during long conversions; called on progress updates |
| **Expiry** | Lease lapses → job returns to claimable state automatically |
| `RequestCancellation` | Soft signal; the running worker observes and transitions cleanly |

The distinction from a distributed lock manager is architectural: **no external coordination service is required**. Lease state is a column, expiry is a comparison, and crash recovery is the passage of time. That is why the platform needs no ZooKeeper, no etcd, no Redis lock — and why horizontal worker scaling works with only PostgreSQL present.

**Cancellation as request, not kill.** A hard kill leaves storage handles open and attempt rows incomplete. `RequestCancellation` lets the worker observe the request at a natural checkpoint and write a clean `Cancelled` terminal state. The cost is latency: cancellation is cooperative, not immediate.

### 7.4 Batch completion aggregation

`CheckBatchCompletionAsync` — multiple overloads, EF-based core, cancellation-aware — runs after **every** terminal job state:

1. Count sibling jobs by status.
2. If all terminal → transition the `BatchJob` to its own terminal state.
3. Emit `BatchNotificationEvent` → SignalR.

This is correct-by-construction for the common case and a plausible hotspot under high parallelism: N workers finishing the last jobs of one batch can race on the same batch transition. The lifecycle guard prevents an illegal double-transition, and the outcome converges — but the redundant count queries scale with job count, not batch count. §13 addresses the fix.

---

## 8. Real-Time Notification

### 8.1 `ProcessingHub`

```
JoinBatch(string batchId)    → Groups.AddToGroupAsync(cid, $"batch_{batchId}")
LeaveBatch(string batchId)   → Groups.RemoveFromGroupAsync(cid, $"batch_{batchId}")
```

That is the entire surface. Both methods short-circuit on `null`, empty, or whitespace — so a malformed client call is a no-op, not an exception in the hub.

**Group scoping `batch_{batchId}` is a security control, not a convenience.** Without it, a single hub broadcasting to `Clients.All` would leak every tenant's job names, output document IDs, and error messages to every connected client. The group boundary is the isolation boundary.

**Deliberately absent:** client-to-client messaging, arbitrary group management, server-invokable methods beyond these two. A minimal hub is a minimal attack surface — there is no method name a client can invoke to make the server do something unintended.

### 8.2 `JobNotificationService`

Injects `IHubContext<ProcessingHub>`, holds `Func<T,Task>?` delegates for `OnJobEvent` / `OnBatchEvent`, logs, then forwards to the hub.

The event-to-broadcast translation is the seam: workers and orchestration publish application-level records (`JobNotificationEvent`), and this service maps them to hub messages. Workers never touch `IHubContext`. That keeps the execution pipeline free of SignalR knowledge — which is what allows `DocumentWorker`, a host with no SignalR pipeline at all, to run the same code path.

### 8.3 Scaling constraint

`ProcessingHub` fan-out is **process-local**. A client connected to web instance A will not receive a broadcast originating from instance B, nor from a `DocumentWorker` instance's notification service, unless a backplane exists. The external-systems inventory lists no Redis or equivalent.

**Deployment implication:** either run a single web instance, enable sticky sessions with worker-originated events routed through the web host, or introduce a SignalR backplane. Progress may appear stale without it even though the underlying job is executing correctly.

---

## 9. API Surface

### 9.1 `ProcessingEndpoints`

```csharp
MapGroup("/api/processing").RequireAuthorization("RequireUser")
```

| Property | Value |
|---|---|
| Route base | `/api/processing` |
| Policy | `RequireUser` — group-level, inherited by every route |
| Binding | Contracts DTOs → Application services via minimal-API primitives |
| Domain logic in endpoints | None |
| Authorization layering | Group policy (HTTP boundary) **and** security abstractions inside `ProcessingService` |

The dual enforcement is intentional defense in depth. A bug in policy configuration does not become an authorization bypass, because orchestration re-validates.

### 9.2 `ConverterDto` consumption

`/api/processing` serves job and batch operations; the catalog is exposed via `IConverterCatalogService` and consumed by both the admin console (`ConvertersCatalog.razor`) and the batch conversion dialog (`BatchConversionDialog.razor`). The UI therefore offers only what the catalog reports as enabled and valid — no client-side allowlist to drift out of sync.

---

## 10. State Machine Reference

Derived from `ProcessingJobLifecycle` as described across the corpus. Transitions marked ✓ are permitted; the absence of a ✓ is the enforcement.

| From \ To | Pending | Queued | Processing | Completed | Failed | Cancelled |
|---|---|---|---|---|---|---|
| **Pending** | — | ✓ | — | — | — | ✓ |
| **Queued** | — | — | ✓ (Claim) | — | — | ✓ |
| **Processing** | — | ✓ (lease expiry / retry) | — | ✓ | ✓ | ✓ |
| **Completed** | — | — | — | — | — | — |
| **Failed** | — | ✓ (if `IsRetryable`) | — | — | — | — |
| **Cancelled** | — | — | — | — | — | — |

**Terminal states:** `Completed`, `Failed`, `Cancelled` — `IsTerminal` gates all aggregation logic.

**Retry loop:** `Processing → Failed → Queued` is legal **only** when `IsRetryable(errorCode)` returns true. A non-retryable failure is genuinely terminal; the job does not requeue and burn capacity re-producing the same failure.

**Expiry loop:** `Processing → Queued` without a failure is the lease-expiry recovery path. A crashed worker's job returns to the pool with its attempt history intact.

> *Note: the exact transition matrix is owned by `ProcessingJobStatus.cs` / `ProcessingJobLifecycle`. The table above is reconstructed from the documented method set and lifecycle responsibilities; verify against source before relying on any single cell.*

---

## 11. Configuration Reference

```jsonc
{
  "RabbitMQ": {
    "Enabled": false,              // master switch: NoOp vs real dispatch
    "HostName": "localhost",
    "Port": 5672,
    "UserName": "guest",
    "Password": "guest",
    "QueueName": "document.processing",
    "ExchangeName": "<configured>",
    "VirtualHost": "/"
  }
}
```

### Deployment tiers

| Tier | Configuration | Execution path | Infrastructure required |
|---|---|---|---|
| **Minimal** | `Enabled: false`, worker hosted in web process | DB polling via `JobProcessorWorker` | PostgreSQL + storage |
| **Standard** | `Enabled: true`, worker in web process | RabbitMQ dispatch + polling fallback | + RabbitMQ |
| **Scaled** | `Enabled: true`, N `DocumentWorker` instances | RabbitMQ distribution across fleet | + worker orchestration |

All three tiers are the same codebase with different configuration. That is the payoff of the `IJobQueuePublisher` seam.

---

## 12. Operational Runbook

### 12.1 Signal → Diagnosis → Action

| Signal | Likely cause | Diagnosis | Action |
|---|---|---|---|
| Jobs stuck in `Queued`, no progress | Worker not running / consumer disabled | Check worker process liveness; check `RabbitMqOptions.Enabled` | Start worker; if broker down, worker fell to polling — confirm polling active |
| Jobs stuck in `Processing` past lease | Worker crash mid-execution | Inspect lease expiry timestamp on the job row | Wait for expiry (auto-recovery) or force `Queued` |
| Batch never reaches terminal | Sibling job never terminated | Count non-terminal siblings | Investigate the straggler; it blocks the batch aggregate |
| Progress UI frozen, job executing | SignalR fan-out across instances | Correlate client instance vs job-originating instance | Add backplane or route events through the web host |
| Repeated retries on one document | `IsRetryable` misclassifying | Read `ProcessingAttempt` error codes | Correct the classification; a deterministic failure should not be retryable |
| Duplicate output documents | Both hosts processed the same job | Cross-check attempt rows | Verify lease claim atomicity; confirm expiry semantics |

### 12.2 Health queries

```sql
-- Stuck processing (lease expired but not recovered)
SELECT id, worker_id, lease_expires_at, attempt_number
FROM processing_jobs
WHERE status = 'Processing' AND lease_expires_at < now();

-- Batch stalled: has non-terminal children for a long time
SELECT b.id, b.status, count(j.id) FILTER (WHERE j.status NOT IN
       ('Completed','Failed','Cancelled')) AS open_jobs
FROM batch_jobs b JOIN processing_jobs j ON j.batch_job_id = b.id
GROUP BY b.id, b.status
HAVING count(j.id) FILTER (WHERE j.status NOT IN
       ('Completed','Failed','Cancelled')) > 0;

-- Retry storms
SELECT job_id, count(*) AS attempts FROM processing_attempts
GROUP BY job_id HAVING count(*) > 3;
```

---

## 13. Known Debt and Recommendations

Ordered by impact × implementation ease.

| # | Issue | Evidence | Recommendation | Effort |
|---|---|---|---|---|
| 1 | **Duplicated pipeline across two hosts** | Architecture report §9.3: different dependency edges (`Worker→Messaging` vs `JobProcessorWorker→Application/Processing`) | Extract `IJobExecutor` into Application/Infrastructure; both hosts become thin loops | 2–3 days |
| 2 | **Parallel `JobMessage` contracts** | Two record families with divergent fields (§6) | Designate `QueueMessages.JobMessage` canonical; deprecate the other; add `ContractVersion` | 0.5 day |
| 3 | **Batch aggregation under contention** | Workflow report §Optimization 4 | Keep the terminal transition, move the count to a periodic sweep; or use a conditional atomic `UPDATE ... WHERE all_terminal` | 1–2 days |
| 4 | **No documented orphan reaper** | Workflow report §Discrepancies: lease implies recovery, no reaper documented | Implement/verify a hosted service sweeping expired leases on a fixed interval | 1 day |
| 5 | **SignalR single-instance fan-out** | Architecture report §9.5 | Add Redis backplane or document the single-instance constraint | 1–2 days |
| 6 | **Endpoint layer conversion bypass** | `DocumentEndpoints.cs` inline DOCX→PDF duplicates `OfficeDocumentProcessor` | Route through `ProcessingService`; delete the inline fork | 1–2 days |
| 7 | **Batch cancellation semantics unspecified** | Workflow report §Discrepancies | Document explicitly: signal in-flight (`RequestCancellation`) + direct-`Cancelled` unleased pending | 0.5 day |

**On #3 specifically.** The current design is correct but quadratic in the worst case: each terminating job issues a count over its siblings. For a 1,000-job batch, that is 1,000 aggregate counts executed as jobs complete. An alternative that preserves the invariant without the cost:

```sql
UPDATE batch_jobs
SET status = 'Completed', completed_at = now()
WHERE id = @batchId
  AND NOT EXISTS (
    SELECT 1 FROM processing_jobs
    WHERE batch_job_id = @batchId
      AND status NOT IN ('Completed','Failed','Cancelled')
  );
```

The conditional update is atomic — the database resolves the race that the current read-then-write sequence resolves in application code. Affected-rows > 0 means this call flipped the batch; the caller then emits exactly one `BatchNotificationEvent`. No distributed lock required.

---

## 14. Evidence Grade — Discrepancies and Gaps

Claims in this document are graded. This section states what the corpus does and does not support.

### SOLID — corroborated across multiple reports

- Broker optionality via `NoOpQueuePublisher` + polling (module report, workflow report, architecture report — all three)
- `ProcessingJobLifecycle` as single transition authority (module report, workflow report, domain modules)
- Dual worker hosts sharing one transaction (module report, domain modules, architecture report)
- Group-scoped minimal hub `batch_{batchId}` (module report, interaction section)
- Lease-based acquisition with expiry recovery (module report, workflow insights)
- SHA-256 hashing + AES-GCM encryption in the per-job pipeline (all reports)

### PLAUSIBLE — consistent but not independently confirmed

- Exact transition matrix (§10) — reconstructed from the documented method set; the authoritative matrix lives in source
- Retry bounding semantics — `IsRetryable` is documented as existing and classifying; the specific error-code taxonomy is not enumerated in the corpus
- Converter catalog constraining job acceptance (relation strength 6.5) — the *mechanism* by which admin limits propagate to `ProcessingService` is not described in any report

### UNVERIFIABLE — requires source access

- **Concurrent operation of both hosts against one database.** Lease semantics strongly imply it is safe, and the domain report states both can run against the same queue/table. But no report confirms the claim operation is atomic at the database level (e.g., a conditional update or row lock). If `Claim` is read-then-write without row-level locking, concurrent hosts can double-process. This is the single highest-value verification in the domain.
- **Reaper existence.** The lease mechanism requires recovery for correctness; whether a hosted service implements it is not stated in any of the five reports.
- **Batch cancellation semantics.** Whether in-flight jobs receive `RequestCancellation`, whether unleased pending jobs are marked `Cancelled` directly, or both — unspecified.
- **Whether the two worker pipelines are byte-identical.** The architecture report notes differing dependency edges, which suggests independent implementations that may have drifted in error handling and renewal cadence.

### Gap summary

| Gap | Why it matters | Verification path |
|---|---|---|
| Claim atomicity | Double-processing risk under horizontal scale | Read `Claim` implementation for conditional-update vs read-then-write |
| No reaper documented | Orphaned `Processing` jobs block batches indefinitely | Search hosted services for lease sweep |
| Batch cancellation | User expectations diverge from behavior | Read batch cancellation path |
| Pipeline drift between hosts | Inconsistent retry/cancellation semantics by deployment topology | Diff `JobProcessorWorker` and `Worker.cs` |
| Key rotation for AES-GCM | Historical documents unreadable after key change | Check `StorageEncryptionOptions` for key versioning |

---

## 15. Design Verdict

The Job Orchestration and Processing domain earns its 9.6 importance rating. Its strongest decisions are structural rather than incidental:

**What is done right.** The lifecycle authority centralizes state semantics so no component can invent its own rules — the most durable of the four architectural invariants the corpus identifies. The `IJobQueuePublisher` seam makes infrastructure optionality a configuration concern rather than a caller concern, which is why three deployment tiers share one code path. Lease-based acquisition delivers crash tolerance without a distributed lock manager, which is why the platform scales workers with only PostgreSQL present. Reference-only message payloads keep broker memory proportional to job count, not document size.

**What is at risk.** The duplicated pipeline across two hosts is the highest-value debt item: it is the one place where the "same codebase, different configuration" claim can quietly become false as the two loops drift. The parallel `JobMessage` contracts are a latent silent-defaulting bug. Batch aggregation is correct but pays a per-job aggregate query for a per-batch decision. SignalR fan-out is process-local, which constrains web-tier scaling in a way no external-system entry acknowledges.

**The bottom line.** This is a domain whose resilience design is genuinely better than typical — degradation is designed, not caught. Its exposure is not in the model but in the seams where two implementations of one intent exist and no test forces them to stay identical.

---

*Module documented from research corpus. Claims graded SOLID / PLAUSIBLE / UNVERIFIABLE. No source was executed; the verification paths in §14 are the next reconnaissance targets.*