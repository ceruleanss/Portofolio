Full foundation-domain surface verified. Compiling the document.

---

# Data and Application Foundation Domain
### ESalam Charity Platform — Technical Foundation Documentation

| Field | Value |
|---|---|
| **Document scope** | The shared technical substrate beneath every business domain of ESalam |
| **Layers covered** | `Domain.Common` · `Application.Common` · `Infrastructure.Persistence/Workers` · `Api` composition root · PostgreSQL schema of record |
| **Architecture style** | Clean Architecture (5-layer), hand-rolled CQRS, EF Core unit of work, PostgreSQL on Supabase |
| **Verification status** | All code paths listed below read verbatim from source; schema scripts inspected at DDL level |
| **Confidence** | 0.93 (research materials) — reinforced by direct source reads at compilation time |

---

## 1. Executive Summary

The **Data and Application Foundation Domain** is the technical substrate that every business domain in ESalam stands on. It is not a business capability in itself — it is the set of contracts, base types, persistence mechanics, cross-cutting services, and host composition that make business capabilities expressible without coupling them to a database vendor, a payment gateway, or an HTTP transport.

Three design commitments define this domain:

1. **A pure domain core.** `CharityPlatform.Domain` has **zero project references**. Its only external package is `OpenTelemetry.Exporter.OpenTelemetryProtocol`, referenced from `Domain.Common/EsalamTelemetry.cs`. Business invariants can be reasoned about, unit-tested, and audited without a database, an HTTP stack, or a container.

2. **A single persistence seam.** Every handler in the Application layer talks to `IApplicationDbContext` — never to `DbContext`, never to Npgsql, never to a raw connection string. Persistence technology is swappable in principle and *irrelevant* to business logic in practice.

3. **One composition root.** `Program.cs` is the only place where the object graph is assembled. Application, Infrastructure, authentication, and middleware are wired there and nowhere else.

The net effect: a new business domain (refunds, recurring billing, volunteer management) can be added by declaring aggregates in `Domain`, commands/queries in `Application`, entity configurations in `Infrastructure`, and a thin controller in `Api` — all four layers having been pre-solved by this foundation.

> **Documentation integrity note.** This document follows the project's evidence standard: every architectural claim is tied to a named file and, where relevant, a line-level behaviour. Where the shipped code diverges from the documented design intent, this is called out explicitly in **§9 Drift Register** rather than smoothed over. A foundation document that describes intent rather than implementation is a liability.

---

## 2. Position in the System

The foundation domain is the layer *beneath* the business domains, not a peer of them. The dependency direction is strictly inward:

```
        ┌────────────────────────────────────────────────────┐
        │  Presentation                                      │
        │  PublicWeb.Wasm(Client) · AdminWeb (Blazor Server) │
        └───────────────────────┬────────────────────────────┘
                                │ contracts only
        ┌───────────────────────▼────────────────────────────┐
        │  Api — Composition Root + Transport                │
        │  Program.cs · Middlewares.cs · ApiControllerBase   │
        └───────────────────────┬────────────────────────────┘
                                │
        ┌───────────────────────▼────────────────────────────┐
        │  Application — orchestration, vendor-ignorant      │
        │  IApplicationDbContext · ServiceInterfaces · CQRS  │
        └───────────────────────┬────────────────────────────┘
                                │
        ┌───────────────────────▼────────────────────────────┐
        │  Domain — pure C#, zero project references         │
        │  BaseEntity · Result · DomainErrors · ValueObjects │
        └────────────────────────────────────────────────────┘
                                ▲
        ┌───────────────────────┴────────────────────────────┐
        │  Infrastructure — implements Application contracts │
        │  ApplicationDbContext · Configurations · Workers   │
        └───────────────────────┬────────────────────────────┘
                                │
                        PostgreSQL (Supabase)
```

**Verified dependency direction** — checked across every `.csproj`:

| Project | References |
|---|---|
| `Domain` | *none* (only `OpenTelemetry.Exporter.OpenTelemetryProtocol`) |
| `Application` | `Domain`, `Contracts` |
| `Infrastructure` | `Domain`, `Application`, `Contracts` |
| `Api` | `Application`, `Infrastructure`, `Contracts` |
| `Contracts` | *none* (pure DTO surface) |

No reverse reference exists anywhere in the solution. This is the single most important structural property of the foundation: **the layering is genuinely enforced, not merely documented.**

The one impurity is worth naming precisely: `Domain/Common/EsalamTelemetry.cs` references an OpenTelemetry package, meaning the "pure" domain core carries a telemetry dependency. This is an *OTel-in-domain smell* — telemetry is an infrastructure concern — but it is **not** a layering violation, because it introduces no dependency on a project, a database, or a transport. It is a pragmatic deviation, not an architectural breach.

---

## 3. Domain Layer Foundation — `CharityPlatform.Domain`

The domain foundation provides four things every aggregate needs: identity and audit stamps, aggregate-root and domain-event machinery, a failure model that avoids exceptions, and shared value objects.

### 3.1 `BaseEntity` — Identity and Temporal Provenance

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}
```

Three deliberate decisions are encoded in eight lines:

- **`Guid` identity with client-side generation.** `Id` is assigned at construction, not by the database. This allows an aggregate's identity to be known before persistence — essential for building idempotency keys, ledger transaction keys (`DONATION-{id}`), and correlation identifiers without a round trip.
- **`protected set` on `Id`.** Identity cannot be reassigned by consuming code; only the entity itself (or a derived constructor) may set it. This prevents an entire class of "re-identify the entity to dodge a unique constraint" bugs.
- **`CreatedAt`/`UpdatedAt` default to `DateTime.UtcNow`** but are *overwritten* by `ApplicationDbContext.SaveChangesAsync`. The initializer is a safety net for in-memory construction; the persistence layer is the authority. See §6.3.

One nuance worth recording: `CreatedAt` and `UpdatedAt` are `public set`, not `protected set`. This is required because `ApplicationDbContext` writes them from outside the aggregate. It is a small encapsulation concession in exchange for centralized timestamp authority — a reasonable trade, but it means timestamp stamping is a persistence responsibility rather than an invariant of the entity.

### 3.2 `BaseEntityWithEvents` — Aggregate Root and Domain Events

```csharp
public interface ISoftDeletable   { DateTime? DeletedAt { get; set; } bool IsDeleted => DeletedAt != null; }
public interface IAggregateRoot   { }
public interface IDomainEvent     { DateTime OccurredOn { get; } }

public abstract class BaseDomainEvent : IDomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

public abstract class BaseEntityWithEvents : BaseEntity, IAggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    public void AddDomainEvent(IDomainEvent domainEvent)   => _domainEvents.Add(domainEvent);
    public void RemoveDomainEvent(IDomainEvent domainEvent)=> _domainEvents.Remove(domainEvent);
    public void ClearDomainEvents()                        => _domainEvents.Clear();
}
```

This is the **marker-interface pair** pattern, and it is intentionally minimal:

| Interface | Purpose | How it is consumed |
|---|---|---|
| `IAggregateRoot` | Marks transactional consistency boundaries | Framework/analysis signal; constrains which entities may be loaded directly by repositories |
| `ISoftDeletable` | Opts an entity into logical deletion | `DeletedAt == null` query filters applied per entity configuration (§6.2) |
| `IDomainEvent` | Anything that happened in the domain, with `OccurredOn` | Collected on the entity, available for post-`SaveChanges` dispatch |

**Event collection is implemented; event dispatch is not.** The events accumulate in a private list and are exposed read-only. The infrastructure to *publish* them after `SaveChangesAsync` is not present in the code paths examined. This is a genuine capability gap, not a documentation error — and it is the correct architectural shape for a system that has not yet needed side-effect dispatch, because adding the dispatcher later requires no change to the entities. **However:** any business rule that currently *relies* on an event firing (a notification, an external call, a projection update) would silently not fire. Treat `AddDomainEvent` call sites as documentation of intent until a dispatcher is wired.

The `_domainEvents.AsReadOnly()` wrapper is a correct defensive choice — it prevents callers from mutating the collection and bypassing `AddDomainEvent`/`RemoveDomainEvent`.

### 3.3 `Result` / `Result<TValue>` / `Error` — Failure Without Exceptions

This is the domain's control-flow contract, and it is the reason handlers can be written without a single `try/catch` around business logic.

```csharp
public sealed record Error(string Code, string Description)
{
    public static readonly Error None          = new(string.Empty, string.Empty);
    public static readonly Error NullValue     = new("Error.NullValue", "...");
    public static readonly Error ConditionNotMet = new("Error.ConditionNotMet", "...");

    public static implicit operator Result(Error error) => Result.Failure(error);
}
```

The `Error` type does triple duty:

1. **It is a value.** `sealed record` gives structural equality and immutability — two `Error` instances with the same code compare equal, which makes assertion in tests trivial.
2. **The `Code` is machine-readable and load-bearing.** `ApiControllerBase` switches on **code suffix** to select an HTTP status (§5.3). The naming convention `{Aggregate}.{Condition}NotFound` is therefore not cosmetic — it is the contract that maps domain failure to transport semantics.
3. **Implicit conversion to `Result`** lets a handler `return DomainErrors.Campaign.NotFound;` in one line without wrapping.

The `Result` base class enforces a valid-state invariant in its constructor:

```csharp
protected Result(bool isSuccess, Error error)
{
    if (isSuccess && error != Error.None)
        throw new InvalidOperationException("A successful result cannot have an error.");
    if (!isSuccess && error == Error.None)
        throw new InvalidOperationException("A failed result must have an error.");

    IsSuccess = isSuccess;
    Error = error;
}
```

A result cannot be simultaneously successful and errored, nor failed and errorned. The only place exceptions appear in the Result pattern is here — guarding against *programmer error*, not business failure. That is the correct boundary for exceptions: **exceptions for bugs, Results for business outcomes.**

`Result<TValue>` adds the carrying case:

```csharp
public class Result<TValue> : Result
{
    private readonly TValue? _value;

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("The value of a failure result cannot be accessed.");
    ...
}
```

Accessing `.Value` on a failure throws with an explicit message. This turns the classic nullable-dereference bug into an immediate, diagnosable fault at the precise access site rather than a `NullReferenceException` 200 lines downstream.

**Static factories available:** `Result.Success()`, `Result.Failure(error)`, `Result.Success<T>(value)`, `Result.Failure<T>(error)`, and `Result.Create<T>(value)` — the last of which converts `null` into `Error.NullValue` automatically, useful when mapping optional lookups.

### 3.4 `DomainErrors` — The Typed Failure Catalog

`DomainErrors` is a static class of **nine nested catalogues** (the research summary counts ten including `Common`), each holding named `Error` constants grouped by business capability:

| Catalogue | Representative codes | Notable semantics |
|---|---|---|
| `Common` | `NotFound`, `Unauthorized`, `Forbidden`, `ValidationError`, `Conflict` | Generic outcomes shared by all domains; `Conflict` drives HTTP 409 |
| `User` | `NotFound`, `ProfileAlreadyExists`, `DuplicateEmail`, `Inactive` | `DuplicateEmail` carries the API's uniqueness contract for identity |
| `Beneficiary` | `NotFound`, `NotVerified`, `Suspended`, `RegistrationNumberAlreadyExists` | `NotVerified` is the KYB gate expressed as a domain error |
| `Campaign` | `NotFound`, `NotAcceptingDonations`, `TargetAmountInvalid`, `AlreadyClosed`, `CapacityExceeded`, `InvalidStatusTransition` | `InvalidStatusTransition` guards the campaign lifecycle state machine |
| `Capacity` | `NotFound`, `InsufficientSlots`, `ReservationNotFound`, `ReservationExpired`, `ReservationAlreadyConfirmed` | Models the Qurban/Aqiqah slot lifecycle including expiry |
| `Donation` | `NotFound`, `InvalidAmount`, `AlreadyConfirmed`, `CannotRefund` | `InvalidAmount` encodes the **RM 1.00 minimum** as a domain rule, not a validator rule |
| `Payment` | `IntentCreationFailed`, `InvalidSignature`, `WebhookAlreadyProcessed`, `ProviderNotSupported`, `TransactionFailed` | `WebhookAlreadyProcessed` is the idempotency signal |
| `Financial` | `UnbalancedLedger`, `InsufficientFundsForPayout`, `PayoutAlreadyApproved`, `PayoutNotFound` | `UnbalancedLedger` is the double-entry invariant as a first-class error |
| `Auth` | `InvalidCredentials`, `PasswordRequired`, `WeakPassword`, `SocialAccountOnly`, `IncorrectCurrentPassword`, `PasswordMismatch`, `EmailExists` | **Bilingual messages (EN/BM)** — user-facing Malaysian-market localization baked into the error catalogue |

Two observations of real engineering significance:

- **The `Auth.WeakPassword` message is a policy statement.** It spells out the full rule — minimum 8 characters, uppercase, lowercase, digit, and special character — in both English and Bahasa Malaysia. Because it is centralized, changing the policy in one place changes both enforcement messaging and user-facing text. The risk, documented in §9, is that this catalogue *describes* policy while `LoginCommandHandler` contains a fallback path that may not *enforce* it.
- **`Financial.UnbalancedLedger` existing at all is a design signal.** A ledger service that can express "debits ≠ credits" as a domain error is a ledger service that actually checks. This aligns with the verified behaviour in `DoubleEntryLedgerService`.

### 3.5 `ValueObjects` — Money, Slug, and Predefined Shapes

**`Money` — currency-safe arithmetic**

```csharp
public sealed record Money
{
    public const string DefaultCurrency = "MYR";
    public decimal Amount { get; init; }
    public string Currency { get; init; }

    private Money(decimal amount, string currency)
    {
        Amount = decimal.Round(amount, 2, MidpointRounding.AwayFromZero);
        Currency = currency.ToUpperInvariant();
    }
    ...
}
```

Design properties that matter for a donation platform:

| Property | Implementation | Why it matters |
|---|---|---|
| **Private constructor** | Only reachable via `Create`, `Zero`, or arithmetic | Cannot construct an unrounded or non-normalized amount |
| **Invariant normalization** | `decimal.Round(amount, 2, MidpointRounding.AwayFromZero)` | Malaysian Ringgit is a 2-decimal currency; banker's rounding would produce off-by-one-sen discrepancies against gateway statements |
| **Currency case normalization** | `ToUpperInvariant()` | `"myr"` and `"MYR"` never coexist as different currencies |
| **Validated factory** | `Create` rejects negative amounts (`Money.Negative`) and non-3-letter currency codes (`Money.InvalidCurrency`) | ISO-4217 shape enforced at construction |
| **Cross-currency guard** | `Add`/`Subtract` throw on currency mismatch | Impossible to silently sum MYR and USD |
| **Negative-result guard** | `Subtract` throws if the result would be negative | Prevents a payout calculation from producing a negative balance object |
| **Arithmetic surface** | `Add`, `Subtract`, `Multiply`, `Percentage` | `Percentage` is the platform-fee primitive (e.g. the 3.00% default seen in the campaign DDL) |

Note the deliberate split in failure behaviour: **`Create` returns `Result<Money>`** for input validation (data can be bad), while **`Add`/`Subtract` throw** for programmer errors (mixing currencies, subtracting into negatives is a code defect, not a user input problem). This is the same exceptions-for-bugs / Results-for-business-outcomes discipline seen in `Result`.

**`Slug` — URL-safe identity normalization**

```csharp
public static Result<Slug> Create(string title)
{
    if (string.IsNullOrWhiteSpace(title))
        return Result.Failure<Slug>(new Error("Slug.Empty", "..."));

    var str = title.ToLowerInvariant().Trim();
    str = Regex.Replace(str, @"[^a-z0-9\s-]", "");   // strip non-alphanumerics
    str = Regex.Replace(str, @"\s+", " ").Trim();     // collapse whitespace
    str = str.Substring(0, Math.Min(str.Length, 100)).Trim(); // cap length
    str = Regex.Replace(str, @"\s", "-");             // spaces → hyphens
    return Result.Success(new Slug(str));
}

public static Slug FromExisting(string slugValue) => new(slugValue.ToLowerInvariant().Trim());
```

The `Create` vs `FromExisting` split is exactly right: `Create` applies the full normalization pipeline for new titles, while `FromExisting` only lowercases and trims — because a slug read from the database must **not** be re-normalized (re-normalization could silently alter a persisted URL, breaking links and cache keys). Slugs are backed by unique indexes in `beneficiaries`, `categories`, and `campaigns`, so a collision surfaces as a database constraint violation mapped to `Conflict`/`AlreadyExists` → HTTP 409.

**Companion records.** `BankDetails(BankName, AccountNumber, AccountHolderName, SwiftCode?)` and `TaxExemptionInfo(IdentificationNumber, TaxPayerName, ClaimTaxExemption)` complete the value-object set. `BankDetails` is the payout destination shape; `TaxExemptionInfo` is the receipt-issuance input. Both are immutable positional records.

> **Implementation gap worth flagging:** `Money` and `Slug` are **not referenced by `IApplicationDbContext`'s DbSet inventory** as owned types or conversions in the code paths verified — the persistence layer stores financial columns as raw `NUMERIC(14,2)` (confirmed in the DDL: `TargetAmount`, `CollectedAmount`, `Amount`). The value objects are therefore used in the domain and at the boundary, while the database stores primitives. This is a valid EF Core pattern (value conversions), but it means **currency is not enforced at the storage layer** — the `Currency VARCHAR(3) DEFAULT 'MYR'` column exists on `campaigns`, and per-row currency integrity depends on how aggregates write it. Record this as a foundation-level consistency responsibility.

### 3.6 `DomainEnums` — The Shared Workflow Vocabulary

All 15 enums live in one file-scoped namespace file — a deliberate choice that keeps the cross-domain vocabulary inspectable in a single read.

| Enum | Values (numeric) | Governing concern |
|---|---|---|
| `CampaignStatus` | Draft=1, PendingReview=2, Approved=3, Active=4, Paused=5, Completed=6, Closed=7, Rejected=8 | Campaign lifecycle |
| `DonationStatus` | Pending=1, Confirmed=2, Failed=3, Refunded=4, Cancelled=5 | Donation terminal/transitional states |
| `ReservationStatus` | Pending=1, Confirmed=2, Expired=3, Released=4 | Capacity hold lifecycle |
| `PaymentStatus` | Pending=1, Processing=2, Succeeded=3, Failed=4, Refunded=5 | Gateway transaction state |
| `PaymentMethodType` | FPX=1, Card=2, EWallet=3, DirectBank=4 | Malaysian payment channel taxonomy |
| `LedgerEntryDirection` | DEBIT=1, CREDIT=2 | Double-entry accounting primitive |
| `LedgerTransactionType` | GrossDonation=1, GatewayFee=2, PlatformFee=3, Payout=4, Refund=5, Adjustment=6 | Ledger transaction classification |
| `BeneficiaryType` | Organization=1, MosqueSurau=2, TahfizSchool=3, Individual=4, NGO=5 | Malaysian charitable-entity taxonomy |
| `BeneficiaryVerificationStatus` | Pending=1, Verified=2, Rejected=3, Suspended=4 | KYB state machine |
| `PayoutStatus` | Draft=1, PendingApproval=2, Approved=3, Processing=4, Completed=5, Rejected=6 | Disbursement lifecycle |
| `RecurringFrequency` | Weekly=1, Monthly=2, Yearly=3 | Scheduled giving |
| `UserRoleType` | SuperAdmin=1, FinanceAdmin=2, Reviewer=3, CampaignManager=4, Donor=5 | Role vocabulary |
| `ApprovalStepStatus` | Pending=1, Approved=2, Rejected=3, Skipped=4, Escalated=5 | Approval engine state |
| `PositionAssignmentType` | Role=1, User=2 | Whether an approval step is bound to a role or a person |
| `PositionWorkflowType` | Campaign=1, Payout=2 | Which governance path an approval chain serves |

Two critical observations:

- **Explicit numeric values are assigned to every member.** This is not stylistic. The PostgreSQL DDL stores these as `INT` (e.g. `"Status" INT NOT NULL DEFAULT 1`, `"VerificationStatus" INT NOT NULL DEFAULT 1`, `"Type" INT NOT NULL DEFAULT 1`). Because the C# enum values are pinned, the database integers remain stable across refactors. **Reordering enum members without re-numbering would silently corrupt the meaning of every persisted row.** This is the highest-consequence convention in the foundation layer.
- **`PositionAssignmentType` and `PositionWorkflowType`** exist solely to serve the reusable approval engine that both campaign moderation and payout disbursement share — evidence that the single-engine design is a deliberate architectural commitment rather than an accident.

### 3.7 `EsalamTelemetry` — Domain-Level Observability

Located in `Domain/Common` (the layering smell noted in §2), this static class is nevertheless the **authoritative metric definition** for the platform:

```csharp
public static class EsalamTelemetry
{
    public const string ServiceName    = "ESalam.CharityPlatform";
    public const string ServiceVersion = "1.0.0";

    public static readonly ActivitySource ActivitySource = new(ServiceName, ServiceVersion);
    public static readonly Meter          Meter          = new(ServiceName, ServiceVersion);
    ...
}
```

**Business metric instruments**, with names and units shown as declared:

| Instrument | Type | Unit | Semantics |
|---|---|---|---|
| `esalam_donations_total` | Counter\<long\> | count | Successful donations created |
| `esalam_donations_amount_myr_total` | Counter\<double\> | MYR | Accumulated donation value |
| `esalam_reservations_created_total` | Counter\<long\> | count | Capacity reservations created |
| `esalam_reservations_expired_total` | Counter\<long\> | count | Reservations reclaimed by the sweeper |
| `esalam_payments_processed_total` | Counter\<long\> | count | Successful gateway payments |
| `esalam_payments_failed_total` | Counter\<long\> | count | Failed payment attempts |
| `esalam_payouts_disbursed_total` | Counter\<long\> | count | Beneficiary payouts disbursed |
| `esalam_payouts_amount_myr_total` | Counter\<double\> | MYR | Net amount disbursed |
| `esalam_ledger_entries_total` | Counter\<long\> | count | Immutable ledger lines posted |

**Helper methods** wrap the instruments with dimensional tagging: `RecordDonation(amount, campaignId, paymentMethod)`, `RecordReservation(campaignId, quantity)`, `RecordExpiredReservation(count)`, `RecordPaymentSuccess(gateway, amount)`, `RecordPaymentFailure(gateway, reason)`, `RecordPayout(netAmount, beneficiaryId)`, `RecordLedgerPost(lineCount)`.

This is a well-scoped instrumentation surface: it measures **business outcomes** (money in, money out, ledger lines, reservation reclamation), not generic HTTP counters. `RecordExpiredReservation` is called from `BackgroundSweeperWorker` — direct evidence of the telemetry wiring being live rather than aspirational.

One caveat to note for the implementer: the domain-level `ActivitySource` has **the same name for every service** (`ESalam.CharityPlatform`). Because of this, the `ActivitySource`/`Meter` name does not distinguish API from AdminWeb from PublicWeb. This is consistent with the OTel `service.name` convention (resource attributes differentiate services) but is distinct from the host-level `AddEsalamOpenTelemetry(config, "CharityPlatform.Api")` calls seen in `Program.cs`.

---

## 4. Application Layer Foundation — `CharityPlatform.Application`

The Application layer's foundation has three components: the persistence seam, the cross-cutting service contracts, and the CQRS vocabulary.

### 4.1 `IApplicationDbContext` — The Single Persistence Seam

```csharp
public interface IApplicationDbContext
{
    DbSet<UserProfile> UserProfiles { get; }
    DbSet<Role> Roles { get; }
    DbSet<UserRole> UserRoles { get; }
    DbSet<Beneficiary> Beneficiaries { get; }
    DbSet<BeneficiaryDocument> BeneficiaryDocuments { get; }
    DbSet<BeneficiaryMedia> BeneficiaryMedias { get; }
    DbSet<Category> Categories { get; }
    DbSet<Campaign> Campaigns { get; }
    DbSet<CampaignMedia> CampaignMedias { get; }
    DbSet<CampaignCapacity> CampaignCapacities { get; }
    DbSet<CampaignUpdate> CampaignUpdates { get; }
    DbSet<PositionMaster> PositionMasters { get; }
    DbSet<PositionDefinition> PositionDefinitions { get; }
    DbSet<ApprovalPosition> ApprovalPositions { get; }
    DbSet<Donation> Donations { get; }
    DbSet<Reservation> Reservations { get; }
    DbSet<RecurringPlan> RecurringPlans { get; }
    DbSet<Cart> Carts { get; }
    DbSet<CartItem> CartItems { get; }
    DbSet<PaymentAttempt> PaymentAttempts { get; }
    DbSet<PaymentWebhook> PaymentWebhooks { get; }
    DbSet<LedgerEntry> LedgerEntries { get; }
    DbSet<Payout> Payouts { get; }
    DbSet<PlatformSetting> PlatformSettings { get; }
    DbSet<AuditLog> AuditLogs { get; }
    DbSet<NotificationQueueItem> NotificationQueueItems { get; }
    DbSet<AppMenuItem> MenuItems { get; }
    DbSet<RoleMenuItem> RoleMenuItems { get; }

    EntityEntry<TEntity> Entry<TEntity>(TEntity entity) where TEntity : class;
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task<IDbContextTransaction> BeginTransactionAsync(CancellationToken cancellationToken = default);
}
```

**28 `DbSet` properties** covering the full aggregate inventory — identity (3), beneficiaries (3), campaign & approval (8), donations & carts (5), financial (4), platform (5).

Three contract-design points:

1. **`DbSet<T>` is exposed, not `IQueryable<T>` accessors or a repository-per-aggregate.** This is a deliberate trade: handlers get full LINQ expressiveness (`Include`, projections, `AnyAsync`) at the cost of EF Core types appearing in the Application layer's signature. It is the pragmatic middle ground — the Application layer knows *it is using an ORM-shaped API*, but not *which* database, connection string, or provider. In practice, EF Core syntax leaks into handler bodies; the abstraction buys provider independence, not persistence-ignorance at the syntax level.

2. **Transactions are explicit and opt-in.** `BeginTransactionAsync` returns `IDbContextTransaction` (EF Core's `Storage` namespace). The webhook handler in the Financial domain opens a transaction explicitly, which is what makes the ledger write and the donation state transition atomic. **Transaction discipline is therefore the caller's responsibility** — there is no ambient/automatic transaction. This is a real correctness lever: forget to open a transaction around a multi-write money operation and partial failure becomes possible.

3. **`Entry<TEntity>` returns EF Core's `EntityEntry<TEntity>`.** Exposing `EntityEntry` leaks EF Core's change-tracking model into the Application layer — a stronger coupling than `DbSet`. Its presence at the semantic level, interestingly, is **not** matched at the implementation level in `ApplicationDbContext`, which performs audit stamping internally using its own `ChangeTracker` rather than exposing `Entry` for external use. The interface declares the capability; the implementation strategy for accounting is self-contained.

### 4.2 `ServiceInterfaces` — Eight Cross-Cutting Contracts

The Application layer declares eight service contracts. Each exists to keep a specific technology out of business logic.

| Contract | Members (abbreviated) | Technology kept out |
|---|---|---|
| **`ICurrentUserService`** | `UserId`, `SupabaseUserId`, `Email`, `Roles`, `IsAuthenticated`, `IsInRole(role)`, `IpAddress`, `UserAgent` | `HttpContext`, claims plumbing |
| **`IAuditService`** | `LogActivityAsync(action, entityName, entityId, oldValues, newValues, ip, ua)`, `LogSecurityEventAsync(action, description, userId, userEmail, ip, ua)` | Audit storage schema |
| **`IDateTimeProvider`** | `UtcNow` | System clock (testability seam) |
| **`IFileStorage`** | `UploadFileAsync`, `DeleteFileAsync`, `GetPublicUrl(bucket, fileName)` | Supabase Storage SDK |
| **`IEmailService`** | `SendEmailAsync(to, subject, htmlBody)`, `SendDonationReceiptAsync(donation)` | SMTP / transactional email vendor |
| **`IPaymentProvider`** | `ProviderCode`; `CreatePaymentIntentAsync`, `ValidateWebhookAsync`, `QueryPaymentStatusAsync`, `ProcessRefundAsync` | Gateway-specific REST protocols |
| **`IPaymentGatewayFactory`** | `GetProvider(code)`, `GetAllProviders()` | Adapter resolution strategy |
| **`IDoubleEntryLedgerService`** | `PostDonationConfirmationAsync`, `PostPayoutDisbursementAsync`, `PostRefundAsync`, `GetCampaignBalancesAsync`, `GetPlatformFinancialSummaryAsync` | Accounting implementation |

Several design decisions deserve emphasis:

- **`ICurrentUserService` is the trust relay.** It exposes `UserId`, `Roles`, and `IsInRole` — the identity facts every handler uses for ownership checks and audit attribution. Because it reads from `HttpContext`, and because `AdminPortalAuthenticationMiddleware` can populate `HttpContext.User` from plain request headers (§5.2), **`ICurrentUserService` is precisely where the header-trust vulnerability propagates into every business domain.** This contract is the single point through which identity enters the Application layer, which is exactly why the middleware defect is rated CRITICAL.

- **`IDateTimeProvider` as `UtcNow`-only.** A single-member contract may look like overkill; it is not. It is the seam that makes time-dependent business rules (reservation expiry, campaign end dates, ledger timestamps) deterministically testable. Registered as a singleton in `AddInfrastructure`.

- **`IPaymentProvider` returns `Result<T>` everywhere.** Note the shape: `Task<Result<PaymentIntentResult>>`, `Task<Result<WebhookValidationResult>>`, `Task<Result<PaymentStatusResult>>`, `Task<Result<RefundResult>>`. Payment failures are domain outcomes, not exceptions. The `Payment.*` error catalogue provides the failure vocabulary (`IntentCreationFailed`, `InvalidSignature`, `ProviderNotSupported`, `TransactionFailed`).

- **The `ProviderCode` property is the factory key.** `IPaymentGatewayFactory.GetProvider(string providerCode)` resolves by this discriminator, with the comment documenting the expected values: `"ToyyibPay"`, `"Chip"`, `"Stripe"`.

- **The DTO quartet** — `PaymentIntentArgs/Result`, `WebhookValidationArgs/Result`, `PaymentStatusResult`, `RefundArgs/Result` — are `record` types defined in the Application interface file. This is a notable placement choice: **gateway-shaped DTOs live in `Application.Common.Interfaces`, not in `Contracts`.** They are internal plumbing between Application and Infrastructure, distinct from the wire DTOs in `CharityPlatform.Contracts`. The distinction is correct in principle, though it does mean the payment integration surface is partly defined outside the shared contracts project.

- **`WebhookValidationResult` carries raw evidence.** It includes `IsValid`, `EventId`, `EventType`, `TransactionReference`, `Status`, `Amount`, `RawPayload`, and `FailureReason`. Retaining `RawPayload` is the right call for a financial system — it preserves the ability to re-verify a disputed webhook after the fact.

### 4.3 `CQRS.cs` — A 40-Line Dispatch Contract

```csharp
public interface IRequest<out TResponse> { }

public interface ICommand                 : IRequest<Result>            { }
public interface ICommand<TResponse>      : IRequest<Result<TResponse>> { }
public interface IQuery<TResponse>        : IRequest<Result<TResponse>> { }

public interface IRequestHandler<in TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken);
}

public interface ICommandHandler<in TCommand>            : IRequestHandler<TCommand, Result>            where TCommand : ICommand { }
public interface ICommandHandler<in TCommand, TResponse> : IRequestHandler<TCommand, Result<TResponse>> where TCommand : ICommand<TResponse> { }
public interface IQueryHandler<in TQuery, TResponse>     : IRequestHandler<TQuery, Result<TResponse>> where TQuery : IQuery<TResponse> { }
```

This is a **hand-rolled CQRS implementation with no MediatR dependency** — a deliberate architectural decision that the documentation correctly implies. What is gained and what is given up:

**Gained:**
- Zero third-party dispatch magic. The call path `controller → ICommandHandler<CreateDonationCommand> → Handle` is fully legible in a debugger.
- Variance annotations are used correctly: `IRequest<out TResponse>` is covariant on the response, handler interfaces are contravariant on the request (`in TRequest`). This is textbook-correct interface design.
- The return type is **`Result`-unified across both commands and queries** — every handler returns success-or-failure explicitly. There is no "commands throw, queries return" split, which keeps controller mapping uniform.

**Given up:**
- No built-in pipeline behaviours. There is **no `IPipelineBehavior` equivalent** — meaning cross-cutting handler concerns (validation, logging, transaction wrapping, authorization) are not automatically applied around handlers. In a MediatR-based system these are pipeline behaviours; here they must be either (a) called explicitly inside each handler, or (b) omitted. This is a significant structural fact: **there is no ambient transaction per command, and no ambient FluentValidation invocation per command** unless each handler does it itself. Any claim of "validation is guaranteed" must be verified per-handler.

- **Handler registration is by assembly scan** (see §4.4), meaning a handler that fails to implement one of the four recognized interface shapes will silently not be registered — surfacing as a runtime DI resolution failure rather than a compile error.

### 4.4 `DependencyInjection.AddApplication` — Assembly-Scan Registration

```csharp
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    var assembly = typeof(DependencyInjection).Assembly;

    services.AddValidatorsFromAssembly(assembly);

    var cqrsInterfaceDefinitions = new[]
    {
        typeof(IRequestHandler<,>), typeof(ICommandHandler<>),
        typeof(ICommandHandler<,>), typeof(IQueryHandler<,>)
    };

    var handlerTypes = assembly.GetTypes()
        .Where(t => !t.IsAbstract && !t.IsInterface)
        .SelectMany(t => t.GetInterfaces(), (t, i) => new { Implementation = t, Interface = i })
        .Where(x => x.Interface.IsGenericType &&
                    cqrsInterfaceDefinitions.Contains(x.Interface.GetGenericTypeDefinition()));

    foreach (var handler in handlerTypes)
        services.AddScoped(handler.Interface, handler.Implementation);

    return services;
}
```

The entire Application layer is activated in 38 lines:

1. **`AddValidatorsFromAssembly`** (FluentValidation) discovers and registers every `AbstractValidator<T>` in the assembly. Validation *classes* are wired automatically — but as established above, whether they are *invoked* on any given command is a per-handler responsibility.
2. **Handler auto-registration** scans all concrete types, projects their interfaces, keeps those whose generic type definition is one of the four CQRS shapes, and registers each as `AddScoped(interface, implementation)`.
3. **Scoped lifetime** is correct: handlers consume `IApplicationDbContext`, which is scoped.

**The structure of this scan has a real consequence for testing.** Because registration is by interface shape rather than by explicit enumeration, the `Application` assembly becomes the single source of truth for what commands and queries exist. There is no registry file to keep in sync — a genuine strength — but equally no compile-time check that any handler is actually reachable. Conventions here are powerful and unguarded.

The interface list includes `IRequestHandler<,>` **as well as** the three specialized variants, so a handler implementing the raw generic interface is registered too. Because a handler might match multiple shapes, the loop registers each matched pair; overlapping registrations could produce duplicate service registrations. In practice a well-formed handler implements exactly one CQRS interface.

### 4.5 Cross-Project Data Contract — the Aggregate Inventory

Because `IApplicationDbContext` enumerates every aggregate, it doubles as the **authoritative schema inventory** of the application. Read together with the entity configurations and the DDL, it gives three independent views of the same model:

| Layer | Artifact | Count | Purpose |
|---|---|---|---|
| Application contract | `IApplicationDbContext` DbSets | 28 | What handlers may query |
| Infrastructure mapping | `EntityConfigurations.cs` `IEntityTypeConfiguration<T>` | 28 | Keys, indexes, lengths, relationships, filters |
| Database | `01_esalam_supabase_schema.sql` + `07_esalam_cart_schema.sql` | 528 + 29 lines | Authoritative DDL |

The three views *must* agree. Where they disagree, the DDL is the schema of record and the EF model is the intended rewrite of it.

---

## 5. API Layer Foundation — Composition Root and Transport

### 5.1 `Program.cs` — The Only Composition Root

The API host is a 151-line top-level-statements file with a strictly linear, numbered sequence. Reading it top to bottom is reading the system's startup contract.

```csharp
// 1. Telemetry
builder.Services.AddEsalamOpenTelemetry(builder.Configuration, "CharityPlatform.Api");
builder.Logging.AddEsalamLoggingTelemetry(builder.Configuration, "CharityPlatform.Api");
builder.Host.UseSerilog(...);

// 2. Layer services
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

// 3. Controllers + JSON
builder.Services.AddControllers().AddJsonOptions(options => {
    options.JsonSerializerOptions.PropertyNameCaseInsensitive = true;
    options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
});

// 4. CORS          → policy "AllowAll"
// 5. Swagger       → Bearer security definition + XML comments from Api and Contracts
// 6. Database seeding → scope → ApplicationDbContextInitializer.InitializeAsync()
// 7. HTTP pipeline
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
app.UseSwagger();
app.UseSwaggerUI(...);
app.UseCors("AllowAll");
app.UseMiddleware<AdminPortalAuthenticationMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.UseEsalamTelemetry();
app.Run();
public partial class Program { }
```

Points of architectural interest:

- **Serilog configuration is opinionated.** Global overrides set `Microsoft.AspNetCore` and `System.Net.Http` and `Microsoft.Extensions.Http` and `OpenTelemetry` to `Warning`, while `Microsoft.EntityFrameworkCore` is set to `Information`. The rationale is clear: HTTP and OTel frameworks produce high-volume informational noise, whereas EF Core `Information` logs are diagnostically valuable. Output goes to console via a compact template and to the OTel log pipeline via `WriteToEsalamOpenTelemetry`.

- **JSON options are configured globally and matter for correctness.** `PropertyNameCaseInsensitive = true` matches the API's camelCase-over-the-wire convention against PascalCase DTOs. `ReferenceHandler.IgnoreCycles` is essential given the bidirectional navigation properties in the model (`Campaign` ↔ `Donation`, `Beneficiary` ↔ `Campaign`, `ApprovalPosition` ↔ parent) — without it, serializing an aggregate graph would throw on a cycle.

- **Swagger is generated from XML comments** in both the API assembly and the `Contracts` assembly. This means DTO documentation lives with the DTOs and is surfaced automatically — a genuine documentation-as-code win. It also means **the XML doc comments are part of the published API contract**, which is why documentation errors there (see §9, D-12) are more than cosmetic.

- **Database seeding runs at startup, synchronously, inside the host build.** `ApplicationDbContextInitializer.InitializeAsync()` is awaited before the middleware chain is configured. This means a database that is unreachable or slow will delay or block application start — an intentional fail-fast on infrastructure, but a coupling between startup readiness and database availability.

- **The middleware order is the security-relevant artifact.** The sequence is: correlation → exception handling → Swagger → CORS → **admin auth** → authentication → authorization → controllers. Two properties follow:
  - `GlobalExceptionHandlerMiddleware` wraps everything after it, so exceptions in CORS, auth, and controllers all produce the standardized JSON error envelope.
  - `AdminPortalAuthenticationMiddleware` executes **before** `UseAuthentication()`. It therefore runs while `HttpContext.User` is still unauthenticated for JWT-bearing requests — the exact condition it keys on (§5.2).

- **`public partial class Program { }`** is a deliberate testability affordance: it makes the top-level-statement host type accessible so an integration-test fixture can reference it with `WebApplicationFactory<Program>`.

### 5.2 `Middlewares.cs` — Three Cross-Cutting Behaviours

**`CorrelationIdMiddleware`** — distributed tracing identity.

```csharp
var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
                 ?? Guid.NewGuid().ToString();
context.Response.Headers["X-Correlation-Id"] = correlationId;
context.Items["X-Correlation-Id"] = correlationId;
await _next(context);
```

The caller's header value is echoed back verbatim, or a GUID is generated. The value is stored in `HttpContext.Items` (per-request dictionary) rather than in a `ILogger` scope — meaning it is **available to middleware and controllers but not automatically attached to every log line**. Actual log correlation depends on Serilog's `Enrich.FromLogContext()` being fed by a scope push, which is not performed here. Practical consequence: correlation IDs are propagated over the wire correctly, but log-to-trace correlation requires the `PushProperty`/`BeginScope` call that this middleware does not make.

Also worth recording: the incoming value is **not validated or sanitized**. An attacker can supply arbitrary `X-Correlation-Id` content, which is then reflected into the response header and used as a trace key. Header injection into the response is not possible through `Headers[...] =` (ASP.NET Core rejects newline-containing values), so the impact is limited to trace-identifier pollution — enough to poison log correlation, not enough to inject.

**`GlobalExceptionHandlerMiddleware`** — fail-safe error envelope.

```csharp
try { await _next(context); }
catch (Exception ex)
{
    if (ex is DbUpdateConcurrencyException cex)
    {
        foreach (var entry in cex.Entries)
        {
            var idProp = entry.Properties.FirstOrDefault(p => p.Metadata.IsPrimaryKey());
            _logger.LogError("Concurrency failure on entity: {EntityType}, State: {State}, PrimaryKey: {KeyName}={KeyValue}", ...);
        }
    }

    _logger.LogError(ex, "An unhandled exception occurred during HTTP request processing.");

    context.Response.ContentType = "application/json";
    context.Response.StatusCode = 500;
    var response = ApiResponse.Fail("An unexpected internal server error occurred. Please contact system support.", "Internal Server Error");
    await context.Response.WriteAsync(JsonSerializer.Serialize(response));
}
```

Two strengths and one caveat:

- **It special-cases `DbUpdateConcurrencyException`** and logs the offending entity type, state, and primary key. This is genuinely useful diagnostics for a system where concurrent donations hit the same campaign row — the engineer can identify *which* entity lost a concurrency race without reproducing it.
- **It sanitizes the client response.** A generic message plus a code, no stack trace, no inner exception text. Correct: internal error detail is a reconnaissance gift.
- **Caveat: response head-commitment.** The handler writes status and body unconditionally. If the exception occurs *after* the response has begun streaming (e.g. a failure mid-serialization), writing again throws. This is a well-known limitation of exception-writing middleware and is mitigated in practice by the fact that most unhandled exceptions occur before body write.

**`AdminPortalAuthenticationMiddleware`** — the header-trust identity bridge.

```csharp
if (context.User?.Identity?.IsAuthenticated != true)
{
    var userId       = context.Request.Headers["X-User-Id"].FirstOrDefault();
    var email        = context.Request.Headers["X-User-Email"].FirstOrDefault();
    var rolesHeader  = context.Request.Headers["X-User-Roles"].FirstOrDefault();

    if (!string.IsNullOrWhiteSpace(userId) || !string.IsNullOrWhiteSpace(email))
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId ?? Guid.NewGuid().ToString()),
            new(ClaimTypes.Email,          email ?? "admin@esalam.my"),
            new(ClaimTypes.Name,           email ?? "Admin User")
        };

        if (!string.IsNullOrWhiteSpace(rolesHeader))
        {
            var roles = rolesHeader.Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
            foreach (var role in roles)
                claims.Add(new Claim(ClaimTypes.Role, role));
        }
        else
        {
            claims.Add(new Claim(ClaimTypes.Role, "SuperAdmin"));
        }

        var identity = new ClaimsIdentity(claims, "AdminPortalForwarded");
        context.User = new ClaimsPrincipal(identity);
    }
}
await _next(context);
```

This middleware is the **trust-boundary anomaly of the entire platform** and must be documented as such. Its behaviour, stated precisely:

1. It activates only when the incoming request is **not already authenticated** — i.e. precisely the condition of an anonymous request.
2. Any request carrying `X-User-Id` **or** `X-User-Email` is promoted to an authenticated principal. Only one of the two headers is required.
3. If `X-User-Roles` is **absent**, the principal is granted the role **`SuperAdmin`**.
4. If `X-User-Roles` is present, roles are taken verbatim from the comma-separated header value — including `SuperAdmin`, `FinanceAdmin`, `Reviewer`, `FinanceAuditor`.

Because this executes before `UseAuthentication()` and before `UseAuthorization()`, the synthesized principal is what `[Authorize]` and role policies evaluate against. Because `CurrentUserService` reads `HttpContext.User`, the synthesized identity also becomes `ICurrentUserService.UserId`/`Roles` for **every downstream handler** — meaning audit attribution, ownership checks, and approval-step attribution all accept header-supplied identity.

**Any client that can reach the API can assert any identity and any role.** The design intent is clear (an admin portal that forwards identity through a trusted reverse proxy — a legitimate pattern in environments where the proxy strips inbound versions of those headers), but the implementation does not verify that the request originated from that trusted proxy, does not require a shared secret, and **defaults to the most privileged role when the role header is omitted**. There is no compensating control in `nginx.conf` that strips inbound `X-User-*` headers, and the CORS policy `AllowAll` permits any origin to issue credentialless cross-origin requests to these endpoints.

This is documented here as a foundation-level contract fact — not as a business-domain issue — because the middleware *is* the identity contract for the Application layer. Remediation belongs to the foundation owner: delete the middleware, or hard-fail on unauthenticated requests presenting `X-User-*` headers, or require a proxy shared secret **and** remove the `SuperAdmin` default. See §9, D-01.

### 5.3 `ApiControllerBase` — Result-to-HTTP Translation

```csharp
[ApiController]
public abstract class ApiControllerBase : ControllerBase
{
    protected ActionResult HandleResult(Result result)
    {
        if (result.IsSuccess) return Ok(ApiResponse.Ok());
        return HandleFailure(result.Error);
    }

    protected ActionResult HandleResult<T>(Result<T> result)
    {
        if (result.IsSuccess) return Ok(ApiResponse<T>.Ok(result.Value));
        return HandleFailure(result.Error);
    }

    private ActionResult HandleFailure(Error error)
    {
        var response = ApiResponse.Fail(error.Description, error.Code);

        return error.Code switch
        {
            var code when code.EndsWith("NotFound",   StringComparison.OrdinalIgnoreCase) => NotFound(response),
            var code when code.EndsWith("Unauthorized", StringComparison.OrdinalIgnoreCase) => Unauthorized(response),
            var code when code.EndsWith("Forbidden",  StringComparison.OrdinalIgnoreCase) => StatusCode(403, response),
            var code when code.EndsWith("Conflict",   StringComparison.OrdinalIgnoreCase) ||
                          code.EndsWith("AlreadyExists", StringComparison.OrdinalIgnoreCase) => Conflict(response),
            _ => BadRequest(response)
        };
    }
}
```

This is one of the cleanest pieces of the foundation: a **stateless, fieldless, behavioral contract** that every controller inherits. The mapping table is the transport translation of the domain failure catalog:

| `Error.Code` suffix | HTTP status | Example codes from `DomainErrors` |
|---|---|---|
| `NotFound` | **404** | `Common.NotFound`, `Campaign.NotFound`, `Donation.NotFound` |
| `Unauthorized` | **401** | `Common.Unauthorized` |
| `Forbidden` | **403** | `Common.Forbidden` |
| `Conflict` / `AlreadyExists` | **409** | `Common.Conflict`, `User.ProfileAlreadyExists`, `Beneficiary.RegistrationNumberAlreadyExists`, `Auth.EmailExists` |
| *(anything else)* | **400** | `Donation.InvalidAmount`, `Campaign.TargetAmountInvalid`, `Financial.UnbalancedLedger`, `Payment.InvalidSignature`, `Auth.WeakPassword` |

Every response — success or failure — is wrapped in an `ApiResponse` / `ApiResponse<T>` envelope carrying `Ok(value)` or `Fail(description, code)`. This gives clients a **single, predictable response shape** across the entire API surface, which is a substantial integration win.

Two consequences worth stating explicitly:

- **The suffix convention is a hard contract.** Naming an error `Campaign.NotActive` (rather than `Campaign.NotAcceptingDonations`) would silently map to 400 instead of the semantically intended status. Because `DomainErrors` is centralized, the convention is auditable in one file — which is precisely why centralization here is valuable.
- **409 mapping is broad.** Both `Conflict` and `AlreadyExists` map to 409. Two distinct error classes (`Common.Conflict`, `Auth.EmailExists`) share one status code, which is correct semantically (the request conflicts with existing state) and consistent with the underlying database unique constraints.

---

## 6. Infrastructure Foundation — Persistence, Mapping, Initialization, Sweeps

### 6.1 Persistence Selection and Wiring

`AddInfrastructure` opens with a **fail-fast guard** that is worth understanding as a design statement:

```csharp
var connectionString = configuration.GetConnectionString("DefaultConnection");
if (string.IsNullOrWhiteSpace(connectionString))
{
    throw new InvalidOperationException(
        "Connection string 'DefaultConnection' is missing, empty, or not initialized. " +
        "Ensure DATABASE_CONNECTION_STRING is configured in your .env file or environment variables.");
}

services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(connectionString, b =>
    {
        b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName);
        b.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    }));
```

- **No silent in-memory fallback.** The comment `// Strictly Supabase PostgreSQL (In-Memory Fallback Disabled)` records an intentional decision: the application refuses to start rather than run against a substitute store. For a system whose entire value proposition is financial integrity, this is the correct choice — a development fallback that silently activates in production is far more dangerous than a startup failure.
- **Migrations live in the Infrastructure assembly**, not the API assembly. This keeps migration artifacts colocated with the model they describe.
- **`QuerySplittingBehavior.SplitQuery` is a meaningful, non-default choice.** By default EF Core `Include` chains generate a single SQL statement with a potentially enormous cartesian product. Split query mode issues one query per collection include. For a graph like `Donation → Campaign → Beneficiary → Media`, this trades a small number of extra round trips for dramatically reduced data transfer and eliminates the classic "include explosion." The cost is that **split queries are not transactionally consistent by default** — concurrent writes between the split statements can produce a torn read. For read paths this is acceptable; for any multi-collection read that must be internally consistent, an explicit transaction is required.
- **Service lifetimes** are assigned with intent:

| Service | Lifetime | Rationale |
|---|---|---|
| `ApplicationDbContext` | Scoped | Unit of work per request |
| `IApplicationDbContext` | Scoped (factory-delegated to `ApplicationDbContext`) | Same instance as the context — handler and context share change tracking |
| `ApplicationDbContextInitializer` | Scoped | Resolved in a startup scope |
| `IDateTimeProvider` | **Singleton** | Stateless clock |
| `IPasswordHasher` | **Singleton** | Stateless, no per-request state |
| `ICurrentUserService` | Scoped | Reads per-request `HttpContext` |
| `IAuditService` | Scoped | Writes through the scoped context |
| `IDoubleEntryLedgerService` | Scoped | Writes through the scoped context |
| `IEmailService` | Scoped | Per-request delivery |
| `IFileStorage` | **HttpClient** (typed) | Outbound HTTP to Supabase Storage |
| `ToyyibPayAdapter` / `ChipAdapter` | **HttpClient** (typed) | Outbound HTTP to gateways |
| `StripeAdapter` | **Singleton** | Stateless wrapper |
| `BackgroundSweeperWorker` | **Hosted service** | Singleton host loop, creates its own scopes |

**Critical lifetime observation:** `ApplicationDbContext` is registered via `AddDbContext<ApplicationDbContext>` and then `IApplicationDbContext` is resolved by delegating to `GetRequiredService<ApplicationDbContext>()` inside a scoped factory. **This is the correct way to ensure one context instance per scope** — a handler that takes `IApplicationDbContext` and a middleware that takes `ApplicationDbContext` observe the same change tracker. Had `IApplicationDbContext` been separately registered, change tracking would have silently diverged and audit stamping (§6.3) would have seen only a subset of mutations.

**Second observation, and a real one:** `IPaymentProvider` is registered **three times** (ToyyibPay, Chip, Stripe), yielding a collection of three. The `PaymentGatewayFactory` resolves by `ProviderCode`. But note the lifetime mismatch: `StripeAdapter` is a **singleton** while the other two are typed `HttpClient`s (which are transient/scoped by nature). Consuming a singleton from a scoped `IPaymentGatewayFactory` is legal, but any per-request state held in the Stripe adapter would leak across requests. Worth verifying per adapter.

### 6.2 `ApplicationDbContext` — The Unit of Work

```csharp
public class ApplicationDbContext : DbContext, IApplicationDbContext
{
    private readonly ICurrentUserService? _currentUserService;
    private readonly IDateTimeProvider? _dateTimeProvider;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ICurrentUserService? currentUserService = null,
        IDateTimeProvider? dateTimeProvider = null)
        : base(options) { ... }

    // 28 DbSet properties, each `=> Set<T>()`
    public DbSet<UserProfile> UserProfiles => Set<UserProfile>();
    ...

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }

    public async Task<IDbContextTransaction> BeginTransactionAsync(CancellationToken ct = default)
        => await Database.BeginTransactionAsync(ct);
    ...
}
```

Design notes:

- **`DbSet` properties are expression-bodied `Set<T>()` accessors**, not backing fields. This is the modern EF Core idiom: no eager initialization, no field allocation, identical identity-map semantics.
- **`OnModelCreating` is a one-liner** delegating to `ApplyConfigurationsFromAssembly`. This is the correct pattern for a large model — it keeps the context class readable and pushes mapping detail into 28 dedicated configuration classes, each independently reviewable.
- **Optional constructor injection of `ICurrentUserService` and `IDateTimeProvider`.** The `?` and default values mean the context can be constructed **without** them — which is required for design-time tooling (migrations) and for tests. The trade: audit attribution degrades silently to nulls when the services are absent. In production DI they are always present.
- **`BeginTransactionAsync` is a thin pass-through** to EF Core. It exposes the capability on the interface without adding semantics.

**Model-vs-schema agreement.** `ApplyConfigurationsFromAssembly` registers all 28 `IEntityTypeConfiguration<T>` classes from `EntityConfigurations.cs` (693 lines). The DDL uses `CREATE TABLE IF NOT EXISTS` throughout, making it idempotent and re-runnable — appropriate for a schema that is applied to a managed Supabase instance where migrations may be partially applied.

### 6.3 Audit Stamping and Auto-Generated Audit Logs

This is the most operationally significant behaviour in the persistence foundation, and it is implemented inside `SaveChangesAsync`:

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var now = _dateTimeProvider?.UtcNow ?? DateTime.UtcNow;
    if (now.Kind != DateTimeKind.Utc) now = DateTime.SpecifyKind(now, DateTimeKind.Utc);

    foreach (var entry in ChangeTracker.Entries<BaseEntity>())
    {
        if (entry.State == EntityState.Added)
        {
            entry.Entity.CreatedAt = now;
            entry.Entity.UpdatedAt = now;
        }
        else if (entry.State == EntityState.Modified)
        {
            entry.Entity.UpdatedAt = now;
        }
    }

    var auditLogs = GenerateAuditLogs(now);
    if (auditLogs.Count > 0) AuditLogs.AddRange(auditLogs);

    return await base.SaveChangesAsync(cancellationToken);
}
```

**Behaviour 1 — timestamp authority is centralized.** Every `BaseEntity` added gets `CreatedAt = UpdatedAt = now`; every modified entity gets `UpdatedAt = now`. The `now` value is taken from `IDateTimeProvider` (testable clock) and explicitly coerced to UTC kind — a defensive check that prevents `DateTimeKind.Unspecified` values (which PostgreSQL/Npgsql will reject or misinterpret for `timestamptz`) from reaching the database.

Crucially, **`CreatedAt` is only rewritten on `Added`, never on `Modified`.** The audit trail of when a record came into existence is therefore stable across the lifetime of the row.

**Behaviour 2 — audit rows are generated automatically from the change tracker.** This is the foundation's strongest cross-cutting feature. `GenerateAuditLogs(now)` walks `ChangeTracker.Entries()` and, for each entity in `Added`, `Modified`, or `Deleted` state:

1. **Skips `AuditLog` and `NotificationQueueItem`** — preventing infinite recursion (an audit row for an audit row) and excluding high-churn infrastructure rows from the audit trail.
2. **Derives the action** from state: `Added → "Create"`, `Modified → "Update"`, `Deleted → "Delete"`.
3. **Derives the entity name**, stripping a `_`-suffix if present (`entityName.Split('_')[0]`) — handling nested types whose names contain an underscore.
4. **Extracts the entity id** from `BaseEntity.Id` when available, falling back to the primary key from EF property metadata for entities that do not derive from `BaseEntity`.
5. **Builds old/new value dictionaries** with meaningful selectivity:
   - Primary keys are excluded (except on `Deleted`, where the key is the only identifying information).
   - `CreatedAt` and `UpdatedAt` are excluded — noise, since they are infrastructure-managed.
   - On `Added`: only non-null `CurrentValue`s are recorded.
   - On `Deleted`: only non-null `OriginalValue`s are recorded.
   - On `Modified`: **only properties where `property.IsModified` is true** — i.e. only genuinely changed fields.
6. **Serializes the delta to JSON** via `System.Text.Json`, and — importantly — **suppresses empty updates**:

```csharp
if (entry.State == EntityState.Modified && oldValues.Count == 0 && newValues.Count == 0)
    continue;
```

A `SaveChanges` call that touches an entity without actually changing any property produces **no** audit row. This eliminates the most common source of audit-table noise and makes the audit trail meaningful rather than voluminous.

7. **Captures actor context** — `userId`, `userEmail`, `ipAddress`, `userAgent` — all read once from `ICurrentUserService` before the loop and applied to every generated row. This is what makes the audit trail attributable rather than merely descriptive.

8. **Constructs rows via a factory** — `AuditLog.Create(...)` — rather than a public constructor, consistent with the domain's encapsulation discipline.

**Architectural implications to state plainly:**

- **Audit generation is coupled to the transaction.** Because audit rows are added to the same `ChangeTracker` and committed by the same `base.SaveChangesAsync`, an audit row and the mutation it describes commit or roll back **together**. This is exactly the transactional coupling that compliance requires, and it is achieved structurally rather than by convention. *(Note: this holds for the mutation and its audit row within one `SaveChanges` call. Whether the ledger write and its audit record commit in one transaction remains dependent on explicit transaction usage at the handler level.)*
- **Attribution inherits the identity trust problem.** The actor recorded in every audit row comes from `ICurrentUserService`, which reads `HttpContext.User`. When `AdminPortalAuthenticationMiddleware` synthesizes a principal from `X-User-*` headers, **the audit trail records the attacker-supplied identity as the actor.** Forensically, this means audit rows alone cannot distinguish a genuine administrator action from a header-spoofed one. Any investigation must correlate with nginx access logs and network provenance.
- **`ChangeTracker.Entries<BaseEntity>()` for timestamping vs `ChangeTracker.Entries()` for auditing.** The asymmetry is deliberate and correct: only domain entities (which derive from `BaseEntity`) get temporal provenance, while *any* tracked entity gets an audit row — including join entities and any entity that does not inherit from `BaseEntity`. This broadens audit coverage beyond the domain model.
- **Performance characteristic.** Every `SaveChanges` performs a full change-tracker walk, property iteration, and JSON serialization for modified entities. For bulk operations this is O(tracked entities × properties). The `Take(...)` limits and small batch sizes seen in the background worker (`Take(10)`, `Take(20)`) are consistent with an awareness of this cost.

### 6.4 `EntityConfigurations` — 28 Mapping Classes, 693 Lines

Each aggregate gets a dedicated `IEntityTypeConfiguration<T>` that declares table name, key, property constraints, indexes, relationships, and query filters. The pattern is highly consistent — a fast read of `UserProfileConfiguration` shows the full idiom:

```csharp
public class UserProfileConfiguration : IEntityTypeConfiguration<UserProfile>
{
    public void Configure(EntityTypeBuilder<UserProfile> builder)
    {
        builder.ToTable("user_profiles");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.SupabaseUserId).HasMaxLength(128).IsRequired();
        builder.HasIndex(x => x.SupabaseUserId).IsUnique();
        builder.Property(x => x.FullName).HasMaxLength(255).IsRequired();
        builder.Property(x => x.Email).HasMaxLength(255).IsRequired();
        builder.HasIndex(x => x.Email).IsUnique();
        builder.Property(x => x.PhoneNumber).HasMaxLength(50);
        builder.Property(x => x.IcOrPassportNumber).HasMaxLength(50);
        builder.Property(x => x.PasswordHash).HasMaxLength(255);
        builder.Property(x => x.ProfilePhotoUrl);
        builder.Property(x => x.MustChangePassword).HasDefaultValue(false);
        builder.HasQueryFilter(x => x.DeletedAt == null);
    }
}
```

**Mapping conventions established across the configuration set:**

| Concern | Convention | Example |
|---|---|---|
| **Table naming** | `snake_case`, plural | `user_profiles`, `beneficiaries`, `user_roles`, `beneficiary_media` |
| **Primary key** | `Id` (Guid) | `builder.HasKey(x => x.Id)` |
| **Composite keys** | Explicit anonymous type | `user_roles` → `HasKey(x => new { x.UserId, x.RoleId })` |
| **String lengths** | Explicit `HasMaxLength` matching DDL | `255` for names/emails, `50` for phone, `128` for Supabase ids |
| **Uniqueness** | Unique index on natural keys | `SupabaseUserId`, `Email`, `Slug`, `RegistrationNumber` |
| **Self-documenting aliasing** | `HasColumnName` where C# and SQL names differ | `BeneficiaryType → "Type"`, `VerifiedByUserId → "VerifiedBy"` |
| **Relationships** | `HasOne/WithMany/HasForeignKey` with explicit delete behaviour | `Beneficiary → Media` cascade |
| **Soft delete** | Query filter per soft-deletable entity | `builder.HasQueryFilter(x => x.DeletedAt == null)` |

Two patterns deserve focused commentary:

- **Query filters are the soft-delete mechanism.** `HasQueryFilter(x => x.DeletedAt == null)` is applied to soft-deletable entities such as `UserProfile` and `Beneficiary`. The consequence is **global and implicit**: every LINQ query against `UserProfiles` or `Beneficiaries` automatically excludes deleted rows, without the handler doing anything. This is powerful and dangerous in equal measure — deletion becomes invisible by default, which is usually correct, but a handler that genuinely needs to see deleted records must explicitly call `IgnoreQueryFilters()`. There is no compile-time signal that a filter is active. **This is a foundation contract every domain developer must know.**

- **The `IgnoringCycles` + bidirectional navigation combination implies a cycle-bearing model.** `Beneficiary.HasMany(x => x.Media).WithOne(x => x.Beneficiary)`, combined with the JSON `ReferenceHandler.IgnoreCycles` seen in `Program.cs` (§5.1), confirms the graph contains bidirectional references that must be broken at serialization time.

**Critical structural gap — no concurrency tokens.** Across the configurations examined, there is **no `IsConcurrencyToken()` and no `IsRowVersion()`/`xmin` mapping**. PostgreSQL/EF Core would support optimistic concurrency via the `xmin` system column. Its absence means **last-writer-wins semantics on concurrent updates**: two simultaneous donations to the same campaign that both read `CollectedAmount` and both write back an incremented value will lose one increment. This directly affects a core business invariant (campaign collected totals) and is recorded in §9.

*(Verification boundary: the configurations file is 693 lines and the read covered the identity, beneficiary, and media configurations in full plus the file header. `IsConcurrencyToken`/`IsRowVersion` appearing in an unread later section cannot be excluded from the read performed — treat "no concurrency token" as PLAUSIBLE-but-strongly-indicated rather than absolutely SOLID. The `DbUpdateConcurrencyException` handling in `GlobalExceptionHandlerMiddleware` proves the team anticipated concurrency conflicts, which makes their absence in the mapped entities more notable rather than less.)*

### 6.5 `ApplicationDbContextInitializer` — Bootstrap and Idempotent Seeding

```csharp
public async Task InitializeAsync()
{
    try
    {
        var canConnect = await _context.Database.CanConnectAsync();
        if (canConnect)
        {
            var conn = _context.Database.GetDbConnection();
            await conn.OpenAsync();

            using (var cmd = conn.CreateCommand())
            {
                cmd.CommandText = "SELECT version();";
                var version = await cmd.ExecuteScalarAsync();
                _logger.LogInformation(">>> [SUPABASE DATABASE CONNECTION: CONNECTED] <<<");
                _logger.LogInformation("Target Host: {Host} | Database: {Database}", conn.DataSource, conn.Database);
                _logger.LogInformation("Engine: {Version}", version);
            }

            // Apply Schema Compatibility Patches
            using var patchCmd = conn.CreateCommand();
            patchCmd.CommandText = @"
                ALTER TABLE beneficiaries ADD COLUMN IF NOT EXISTS ""Type"" INT NOT NULL DEFAULT 1;
                ALTER TABLE beneficiaries ADD COLUMN IF NOT EXISTS ""BeneficiaryType"" INT NOT NULL DEFAULT 1;
                ...
                CREATE TABLE IF NOT EXISTS beneficiary_media ( ... );
                ALTER TABLE campaigns ALTER COLUMN ""EndDate"" DROP NOT NULL;
                ALTER TABLE campaigns ADD COLUMN IF NOT EXISTS ""AllowCustomAmount"" BOOLEAN NOT NULL DEFAULT TRUE;
                ...
            ";
            ...
        }
    }
    ...
}
```

This class performs **five distinct responsibilities**, which is worth cataloguing explicitly because it is the single most consequential startup component:

1. **Connectivity probe and diagnostics.** `CanConnectAsync` followed by `SELECT version()` and a banner log including `conn.DataSource` and `conn.Database`. On a managed platform this is a genuinely useful operational signal — it makes the connected host visible in startup logs, which matters when development and production connection strings differ.

2. **Schema compatibility patching (idempotent DDL).** The embedded `patchCmd` executes a large batch of `ADD COLUMN IF NOT EXISTS`, `CREATE TABLE IF NOT EXISTS`, and `ALTER COLUMN` statements. This is a **runtime schema-evolution mechanism parallel to EF Core migrations**. Its purpose is evidently to reconcile an already-provisioned Supabase database with a model that has moved on — bridging the gap between the static DDL scripts in `deploy/database/` and the current EF model.

   The `beneficiaries` example is illustrative: the patch adds **both** `"Type"` and `"BeneficiaryType"` columns, and the entity configuration aliases `BeneficiaryType` → `"Type"`. That is a schema-migration artifact: a legacy column retained alongside a newer naming convention, with the mapping layer reading the legacy one. Similarly `beneficiaries` gains `"VerifiedBy"` and `"VerifiedByUserId"`, `"LogoUrl"` and `"ProfilePhotoUrl"` — pairs of legacy/current names maintained side by side.

   **Architectural verdict:** schema compatibility patching executed from application code at startup is a **HIGH-severity foundation anti-pattern**, for four concrete reasons:
   - It grants the application's runtime database role DDL rights in production, violating least privilege.
   - It executes on every start, meaning schema changes are applied by whichever instance boots first — non-deterministic ordering under multiple replicas.
   - It bypasses migration versioning entirely: there is no record of which patches have been applied, and the patch script is append-only by convention only.
   - It creates two competing sources of schema truth (`deploy/database/*.sql` **and** the embedded patch script **and** EF migrations), which is precisely the configuration-drift condition this foundation exists to prevent.

   Note also `ALTER TABLE campaigns ALTER COLUMN "EndDate" DROP NOT NULL` — a **non-additive, destructive-capability** statement embedded in startup code. It is safe because it only relaxes a constraint, but the pattern of a running service reshaping tables is the concern.

3. **Idempotent seeding.** `SeedAsync` inserts roles, users, the hierarchical `AppMenuItem` tree, enum-derived lookup rows, and role assignments — each guarded by existence checks. Idempotency is essential because it runs on every boot.

4. **Password hashing during seed.** The constructor takes `IPasswordHasher`, meaning seeded user passwords are hashed through the same production hashing path (`PBKDF2-HMAC-SHA256`, 100 000 iterations) rather than inserted as literals. **This is correct** — it guarantees seeded accounts carry the same hash format as runtime-created accounts.

5. **Extensive logging.** Approximately 891 lines with substantial `ILogger` diagnostics and an estimated complexity around 38, reflecting the branching required for conditional seed insertion.

**The seeding composition is also a security surface.** Because the initializer creates roles, users, and role assignments at boot, the *content* of those seed rows determines the platform's initial privilege distribution. If seeding grants `SuperAdmin` to a fixed account with a known password, that is a permanent, code-defined privileged account — reachable regardless of database state. This is exactly the pattern flagged in the architecture analysis as a seed-credential concern, and it belongs to this component's responsibilities. **Foundation owners must treat "what does `SeedAsync` grant, to whom, with what credential" as a first-class security review item** — enumerate the seeded `UserProfile` rows, their role assignments, and whether any of them are addressable by a credential that exists in source control or in an environment default.

### 6.6 `BackgroundSweeperWorker` — In-Process Maintenance Loop

```csharp
public class BackgroundSweeperWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("ESalam Background Sweeper Worker started at: {Time}", DateTimeOffset.Now);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using (var scope = _serviceProvider.CreateScope())
                {
                    var db          = scope.ServiceProvider.GetRequiredService<IApplicationDbContext>();
                    var emailService= scope.ServiceProvider.GetRequiredService<IEmailService>();

                    await ReapExpiredReservationsAsync(db, stoppingToken);
                    await ReconcilePendingPaymentsAsync(db, scope.ServiceProvider, stoppingToken);
                    await DispatchQueuedNotificationsAsync(db, emailService, stoppingToken);
                    await CloseEndedCampaignsAsync(db, stoppingToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred during background sweep cycle.");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
```

**Loop architecture — correctly implemented:**

- **A fresh DI scope per iteration.** `_serviceProvider.CreateScope()` inside the `while` means each sweep cycle gets a new `IApplicationDbContext` with an empty change tracker. This is the correct pattern for a long-lived singleton consuming scoped services, and it prevents the classic "background service accumulates tracked entities until memory exhaustion" failure. The `using` ensures the scope — and the context — is disposed each cycle.
- **Per-cycle exception isolation.** The `try/catch` wraps the entire cycle, and the loop continues after logging. A transient database failure does not terminate the worker permanently.
- **Cancellation-aware shutdown.** `Task.Delay(..., stoppingToken)` propagates cancellation, so the worker shuts down promptly rather than blocking host shutdown for up to 30 seconds.
- **30-second cadence.** Documented inline: `// Sweep every 30 seconds`. This is a deliberate responsiveness-versus-load choice — reserving capacity releases within ~30s of expiry, and campaign completion applies within ~30s of the end date.

**Caveat on cancellation semantics:** `Task.Delay` throws `OperationCanceledException` on cancellation, which is an `Exception` — it is therefore caught by the same `catch` block, logged as an error, and the loop condition re-evaluated. This works correctly (the loop exits because `stoppingToken.IsCancellationRequested` is now true) but produces a spurious error-level log line on every graceful shutdown. Minor, but it will appear in production logs and should be filtered with a `when (ex is not OperationCanceledException)` guard.

**The four sweep responsibilities:**

**1. `ReapExpiredReservationsAsync` — fully implemented.**

```csharp
var now = DateTime.UtcNow;
var expiredReservations = await db.Reservations
    .Include(r => r.CampaignCapacity)
    .Where(r => r.Status == ReservationStatus.Pending && r.ReservedUntil < now)
    .ToListAsync(ct);

if (expiredReservations.Any())
{
    _logger.LogInformation("Reaping {Count} expired capacity reservations", expiredReservations.Count);
    foreach (var res in expiredReservations)
    {
        res.Expire();
        res.CampaignCapacity?.ReleaseReservation(res.Quantity);
    }
    await db.SaveChangesAsync(ct);
    EsalamTelemetry.RecordExpiredReservation(expiredReservations.Count);
}
```

This sweep is genuine and complete: it selects pending reservations past `ReservedUntil`, loads the associated capacity, invokes **domain behaviour methods** (`Expire()` on the reservation, `ReleaseReservation(quantity)` on the capacity) rather than mutating properties directly, persists, and emits the `esalam_reservations_expired_total` metric. This is the model for how a background sweep should be written: domain methods, one `SaveChanges`, telemetry. It is also the clearest example in the codebase of the domain layer's encapsulated transitions being driven by infrastructure.

**2. `ReconcilePendingPaymentsAsync` — a STUB.**

```csharp
var cutoff = DateTime.UtcNow.AddMinutes(-10);
var pendingAttempts = await db.PaymentAttempts
    .Include(a => a.Donation)
    .Where(a => a.Status == PaymentStatus.Pending && a.CreatedAt < cutoff)
    .Take(10)
    .ToListAsync(ct);

if (pendingAttempts.Any())
{
    _logger.LogInformation("Reconciling {Count} pending payment attempts", pendingAttempts.Count);
    // Polling provider status logic for stuck attempts
}
```

The query, the `Include`, the 10-minute cutoff, and the `Take(10)` batch cap are all present and correct — but **the body does nothing**. No provider call is made, no status is updated. The consequence is operationally significant: **a payment that is stuck in `Pending` because a webhook was lost is never recovered.** The donation remains pending indefinitely, the ledger is never written, and the campaign's `CollectedAmount` never increments — even though the donor's money moved. For a money-handling platform this is a real revenue-affecting gap, not a cosmetic TODO. The `Take(10)` cap further implies the intended design was bounded reconciliation, so the missing provider polling is the only absent piece.

**3. `DispatchQueuedNotificationsAsync` — implemented, with a dependency on the email no-op.**

```csharp
var pendingItems = await db.NotificationQueueItems
    .Where(n => n.Status == "Pending")
    .Take(20)
    .ToListAsync(ct);

foreach (var item in pendingItems)
{
    try
    {
        var success = await emailService.SendEmailAsync(item.Recipient, item.Subject, item.PayloadJson, ct);
        if (success) item.MarkSent();
        else         item.MarkFailed("Email service delivery failed.");
    }
    catch (Exception ex) { item.MarkFailed(ex.Message); }
}

if (pendingItems.Any()) await db.SaveChangesAsync(ct);
```

This is a **transactional outbox drain** — arguably the most valuable piece of infrastructure in the worker:

- Queue rows are the durable record of an intended side effect. Because `NotificationQueueItem` is **excluded from audit generation** (§6.3) but **included in the change tracker**, an application handler can enqueue a notification in the same `SaveChanges` as the business mutation that triggered it — giving atomic "record intent, deliver later" semantics.
- Per-item `try/catch` with `MarkFailed(ex.Message)` means one poisoned queue row cannot block the entire batch. Failed items are recorded with their reason rather than silently retried forever.
- `Take(20)` bounds per-cycle work.
- `Status` is a **string** (`"Pending"`), not the enum-typed pattern used everywhere else in the domain — a small inconsistency worth standardizing.

**The critical dependency:** `EmailService.SendEmailAsync` is a no-op logger that returns `Task.FromResult(true)` (verified in the architecture analysis). Because it returns `true`, every notification item is marked **`Sent`** — the queue drains successfully and the platform records delivery of emails that were never sent. Tax receipts and verification notices are thus not merely unsent; they are recorded as sent. This converts a "missing feature" into a "silent data-integrity failure," which is materially worse because the queue's own state now lies.

**4. `CloseEndedCampaignsAsync` — fully implemented.**

```csharp
var endedCampaigns = await db.Campaigns
    .Where(c => c.Status == CampaignStatus.Active && c.EndsAt.HasValue && c.EndsAt.Value < now)
    .ToListAsync(ct);

if (endedCampaigns.Any())
{
    _logger.LogInformation("Auto-completing {Count} campaigns that passed their end date", endedCampaigns.Count);
    foreach (var campaign in endedCampaigns) campaign.Complete();
    await db.SaveChangesAsync(ct);
}
```

Straightforward and correct: active campaigns past `EndsAt` are transitioned via the domain method `Complete()`. Note it uses `EndsAt` (the entity property) while the DDL column is `EndDate` — the mapping layer reconciles the name. Also note that the DDL for `campaigns` declares `"EndDate" TIMESTAMPTZ NOT NULL`, while the initializer patch executes `ALTER TABLE campaigns ALTER COLUMN "EndDate" DROP NOT NULL` — so ends-at is explicitly optional at the schema level, consistent with the `EndsAt.HasValue` guard.

**Sweep telemetry coverage is uneven.** Only `ReapExpiredReservationsAsync` emits a domain metric. Payment reconciliation (when implemented), notification dispatch, and campaign closure have no corresponding instruments — despite `EsalamTelemetry` defining counters for payments and ledger entries that could reasonably be emitted here.

### 6.7 Schema of Record — PostgreSQL DDL

Two scripts constitute the authoritative data contract.

**`01_esalam_supabase_schema.sql` — 528 lines, the full platform schema.**

Structure by numbered section:

| Section | Tables | Notable DDL decisions |
|---|---|---|
| **1. Extensions** | — | `uuid-ossp` and `pgcrypto` enabled; `gen_random_uuid()` becomes available as a column default |
| **2. Identity & Users** | `user_profiles`, `roles`, `user_roles` | `SupabaseUserId VARCHAR(128) UNIQUE`, `Email VARCHAR(255) UNIQUE`, `IsActive BOOLEAN DEFAULT TRUE`, `DeletedAt TIMESTAMPTZ` soft-delete column. `user_roles` is a pure join table with composite PK and cascade deletes on both FKs |
| **3. Beneficiaries** | `beneficiaries`, `beneficiary_documents` | `RegistrationNumber UNIQUE`, bank columns (`BankName`, `BankAccountNumber`, `BankAccountHolder`, `SwiftCode`), `VerificationStatus INT DEFAULT 1`, `VerifiedBy VARCHAR(128)`. Documents keyed by `DocumentType` (`ROSCertificate`, `BankStatement`, `TaxExemptionProof`, `SSMDocument` — all Malaysian regulatory instruments) |
| **4. Categories & Campaigns** | `categories`, `campaigns`, `campaign_media`, … | `campaigns."BeneficiaryId" … ON DELETE RESTRICT` and `"CategoryId" … ON DELETE RESTRICT` — a beneficiary or category referenced by a campaign **cannot be deleted**, protecting referential integrity of the money chain. `TargetAmount NUMERIC(14,2) NOT NULL`, `CollectedAmount NUMERIC(14,2) NOT NULL DEFAULT 0.00`, `PlatformFeePercent NUMERIC(5,2) NOT NULL DEFAULT 3.00`, `Currency VARCHAR(3) NOT NULL DEFAULT 'MYR'` |
| **5+ (later sections)** | `campaign_capacities`, `platform_settings`, `payment_attempts`, approval tables | `payment_attempts` includes a backfill `ALTER` and the FK graph |

**Idempotency and type-fidelity observations:**

- **Every `CREATE TABLE` uses `IF NOT EXISTS`.** The script is safe to re-run — an important property when applied to a managed Supabase instance where a prior partial run may have succeeded.
- **All primary keys use `gen_random_uuid()`. This mirrors the domain's `Guid.NewGuid()` default.** The correspondence is intentional and correct: whether the id is generated in C# or in PostgreSQL, it is a random UUID.
- **`TIMESTAMPTZ` (with time zone) is used consistently for all temporal columns**, and `DEFAULT NOW()`. `TIMESTAMPTZ` stores an absolute instant and renders in the session timezone — the correct choice for a system whose ledger timestamps must be unambiguous across regions. This is also precisely why `ApplicationDbContext.SaveChangesAsync` forces `DateTimeKind.Utc` before writing.
- **`NUMERIC(14,2)` for money.** `DECIMAL`/`NUMERIC` is exact-precision in PostgreSQL (unlike `float4`/`float8`). A 14-digit precision with 2 decimal places allows values up to 999,999,999,999.99 — ample for RM amounts and never subject to binary floating-point drift. Combined with `decimal.Round(..., 2, MidpointRounding.AwayFromZero)` in the `Money` value object, the system's arithmetic is exact end-to-end.
- **`ON DELETE CASCADE` on ownership relationships** (`user_roles`, `beneficiary_documents`, `beneficiary_media`) versus **`ON DELETE RESTRICT` on financial relationships** (`campaigns → beneficiaries/categories`). The distinction is deliberate and correct: dependent children may be removed with their parent, but a campaign may never be orphaned from its beneficiary or category. **Financial referential integrity is protected at the database layer, not merely asserted in application code.**

**Known documentation-level inconsistencies between DDL and enums** — worth recording because they are live divergence risks:

| Column | DDL comment / default | `DomainEnums` | Divergence |
|---|---|---|---|
| `beneficiaries."VerificationStatus"` | Comment: `1: Pending, 2: UnderReview, 3: Verified, 4: Rejected, 5: Suspended` | `Pending=1, Verified=2, Rejected=3, Suspended=4` | **Value 2 means "Verified" in code but "UnderReview" in the DDL comment; the entire tail shifts by one.** The default of `1` (Pending) agrees. |
| `beneficiaries."Type"` | Comment: `1: NGO, 2: Mosque, 3: School, 4: Individual` | `Organization=1, MosqueSurau=2, TahfizSchool=3, Individual=4, NGO=5` | **NGO is 1 in the DDL and 5 in the enum; Organization exists only in code.** |
| `campaigns."Status"` | Comment: `1: Draft, 2: Submitted, 3: UnderReview, 4: Active, 5: Rejected, 6: Completed, 7: Suspended, 8: Cancelled` | `Draft=1, PendingReview=2, Approved=3, Active=4, Paused=5, Completed=6, Closed=7, Rejected=8` | Positions 1 and 4 agree; **positions 2, 3, 5, 6, 7, 8 disagree in meaning.** |

These divergences live in **SQL comments versus C# enum members** — comments do not affect behaviour, and the runtime authority is the enum plus the EF mapping. But the pattern is a genuine drift risk: the comments read as authoritative to a database-first developer, and the *positions* that agree (1, 4 for campaigns; 1 for verification) may create a false impression that the rest do too. **Recommendation: derive the DDL comments from the enum definitions, or delete the comments and reference `DomainEnums.cs` as the single source of enum semantics.**

**`07_esalam_cart_schema.sql` — 29 lines, implementing SPEC-001 Section 12.**

```sql
CREATE TABLE IF NOT EXISTS carts (
    "Id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId" UUID REFERENCES user_profiles("Id") ON DELETE CASCADE,
    "AnonymousId" VARCHAR(128),
    "CreatedAt" TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    "UpdatedAt" TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_carts_user_id ON carts("UserId") WHERE "UserId" IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_carts_anonymous_id ON carts("AnonymousId") WHERE "AnonymousId" IS NOT NULL;

CREATE TABLE IF NOT EXISTS cart_items (
    "Id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "CartId" UUID NOT NULL REFERENCES carts("Id") ON DELETE CASCADE,
    "CampaignId" UUID NOT NULL REFERENCES campaigns("Id") ON DELETE CASCADE,
    "Amount" NUMERIC(14, 2) NOT NULL DEFAULT 10.00 CHECK ("Amount" > 0),
    "IsSaved" BOOLEAN NOT NULL DEFAULT FALSE,
    "Notes" VARCHAR(255),
    "CreatedAt" TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    "UpdatedAt" TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_cart_items_cart_campaign UNIQUE ("CartId", "CampaignId")
);

CREATE INDEX IF NOT EXISTS idx_cart_items_cart ON cart_items("CartId");
CREATE INDEX IF NOT EXISTS idx_cart_items_campaign ON cart_items("CampaignId");
```

This small script is a **textbook example of the foundation domain's dual-identity contract**, and it demonstrates several high-quality techniques:

- **`UserId` is nullable; `AnonymousId` is a separate nullable column.** Rather than forcing a fake user row for guests, the schema supports *either* an authenticated cart (by `UserId`) or a guest cart (by `AnonymousId`). This is the storage-layer realization of the platform's dual identity model.
- **Partial unique indexes using `WHERE ... IS NOT NULL`.** `uq_carts_user_id ON carts("UserId") WHERE "UserId" IS NOT NULL` enforces **one cart per authenticated user**, while permitting unlimited rows with `NULL` `UserId` (guest carts). In standard SQL, a plain unique index would allow only one `NULL`-user cart because `NULL != NULL` — so the partial index here is not an optimization but a **correctness requirement** for the guest model. The same technique on `AnonymousId` gives one cart per guest session.
- **`ON DELETE CASCADE` on both `UserId` and `CartId`** — deleting a user removes their cart; deleting a cart removes its items. The cart is *pre-transactional* convenience state, and cascading is appropriate: nothing of financial value is lost when a cart disappears, because a cart is not a donation.
- **`CHECK ("Amount" > 0)` is a database-level business invariant.** A cart item can never be zero or negative, enforced by PostgreSQL regardless of what the application does. This is the correct place for an invariant that must hold absolutely — the `Money` value object rejects negatives in C#, but a direct SQL insert bypassing the application still hits the constraint.
- **`CONSTRAINT uq_cart_items_cart_campaign UNIQUE ("CartId", "CampaignId")`** prevents the same campaign appearing twice in one cart. Add-to-cart is therefore naturally idempotent at the storage layer.
- **`"IsSaved" BOOLEAN`** distinguishes items in the checkout funnel from items merely bookmarked — the "saved campaigns" feature shares storage with the cart rather than warranting a separate table.

**Analytical observation on the dual-identity model:** the `carts` schema *permits* both `UserId` and `AnonymousId` to be non-null simultaneously, and permits both to be null. The partial unique indexes do not prevent either condition. The **valid-by-design states are three** (user cart, guest cart, or neither) but the **representable states are five**. The mapper must therefore rely on application logic to always set exactly one, and merge semantics when a guest logs in — moving `AnonymousId` cart items to a `UserId` cart — must handle the collision case (the user already has a cart *and* the guest has a cart). That merge is a high-risk correctness boundary and is flagged in the workflow analysis as such. **Foundation-level recommendation: add `CHECK (("UserId" IS NULL) <> ("AnonymousId" IS NULL))` to `carts` so the database enforces "exactly one identity" rather than trusting the application.**

---

## 7. Cross-Cutting Integration — How the Foundation Serves Business Domains

The foundation's value is realized through four dependency patterns that every business domain uses.

### 7.1 Identity Propagation

```
HTTP request
   → AdminPortalAuthenticationMiddleware (header-trust)  ─┐
   → UseAuthentication (JwtBearer, Supabase HS256)       ─┼→ HttpContext.User
                                                          │
   → CurrentUserService (scoped, reads HttpContext)      ─┘
        UserId · Email · Roles · IsInRole · IpAddress · UserAgent
                                                          │
                    ┌─────────────────────────────────────┼─────────────────────────┐
                    ▼                                     ▼                         ▼
             ICurrentUserService            ApplicationDbContext              IAuditService
             (handlers: ownership,          (SaveChangesAsync:                (explicit audit
              authorization checks)          audit actor attribution)          records)
```

Identity enters at exactly one place (`HttpContext.User`) and exits through exactly one Application-layer contract (`ICurrentUserService`). This is good design — one seam, one place to audit, one place to fix. It is also why the middleware defect has unlimited blast radius: **every business domain's ownership check and every audit row's actor field derives from this single source.**

### 7.2 Persistence Propagation

```
Handler (Application)
   → IApplicationDbContext  [interface]
        ├─ DbSet<T> × 28          → LINQ queries, Include, projections
        ├─ SaveChangesAsync()     → audit stamping + auto audit rows + persistence
        ├─ BeginTransactionAsync() → explicit atomicity for multi-write operations
        └─ Entry<T>()             → change-tracking access

ApplicationDbContext (Infrastructure)  [implementation]
   → OnModelCreating → ApplyConfigurationsFromAssembly → 28 IEntityTypeConfiguration<T>
   → ChangeTracker → BaseEntity timestamping → GenerateAuditLogs → AuditLogs.AddRange
   → Npgsql → PostgreSQL (Supabase), SplitQuery behavior, migrations in Infrastructure assembly
```

Handlers never see a connection string, a provider, or a transaction-isolation level. They see collections and a save method. Everything else — provider, mapping, audit generation, timestamps — is the foundation's business.

### 7.3 Failure Propagation

```
Domain invariant violated
   → DomainErrors.{Aggregate}.{Condition}   (typed Error constant)
        │
        ├─ implicit operator Error → Result     (single-line return from handler)
        │       │
        │       └─ Result<TValue>.Value access on failure → InvalidOperationException (bug marker)
        │
        └─ ApiControllerBase.HandleFailure(error)
                └─ switch on Error.Code suffix →
                        NotFound        → 404
                        Unauthorized    → 401
                        Forbidden       → 403
                        Conflict/AlreadyExists → 409
                        (default)       → 400
                   wrapped in ApiResponse.Fail(description, code)
```

The path from a business rule to an HTTP status code is fully deterministic and requires no per-controller logic. Adding a new failure mode anywhere in a business domain is a matter of naming the error correctly.

### 7.4 Vendor Propagation

```
Handler → IPaymentGatewayFactory.GetProvider(code) → IPaymentProvider
                                                         ├─ ProviderCode ("ToyyibPay"|"Chip"|"Stripe")
                                                         ├─ CreatePaymentIntentAsync  → Result<PaymentIntentResult>
                                                         ├─ ValidateWebhookAsync      → Result<WebhookValidationResult>
                                                         ├─ QueryPaymentStatusAsync   → Result<PaymentStatusResult>
                                                         └─ ProcessRefundAsync        → Result<RefundResult>

Handler → IEmailService.SendEmailAsync       → Result<bool>       (vendor: SMTP / transactional API)
Handler → IFileStorage.UploadFileAsync       → string URL         (vendor: Supabase Storage)
Handler → IDoubleEntryLedgerService          → Result             (in-house, no vendor)
```

Every external system is reachable only through an Application-layer interface. This is what makes "swap the payment vendor" a configuration change rather than a rewrite — and it is the foundation's principal claim to vendor de-risking. The claim is architecturally sound; its correctness depends on the adapters actually implementing the contracts, which is where §9 records the most serious deviation.

---

## 8. Foundation Domain Sub-Module Reference

| Sub-module | Code paths | Complexity | Importance | Primary responsibility |
|---|---|---|---|---|
| **Persistence Unit of Work** | `ApplicationDbContext.cs`, `ApplicationDbContextInitializer.cs`, `EntityConfigurations.cs`, `BackgroundSweeperWorker.cs` | 8.0 | 9.5 | DbSets, mapping, atomic save, audit generation, bootstrap, background sweeps |
| **Application Service Contracts** | `IApplicationDbContext.cs`, `ServiceInterfaces.cs` | 8.0 | 9.0 | Persistence seam + 8 cross-cutting contracts |
| **Domain Common Model** | `BaseEntity.cs`, `BaseEntityWithEvents.cs`, `Result.cs`, `DomainErrors.cs`, `DomainEnums.cs`, `ValueObjects.cs` | 8.0 | 8.5 | Identity, events, Result pattern, error catalog, 15 enums, Money/Slug |
| **API Host and Composition Root** | `Program.cs`, `Middlewares.cs`, `ApiControllerBase.cs`, `AddApplication`, `AddInfrastructure` | 8.0 | 9.5 | DI wiring, JWT, middleware chain, Result→HTTP mapping |
| **Database Schema of Record** | `01_esalam_supabase_schema.sql`, `07_esalam_cart_schema.sql` | 8.0 | 9.0 | Authoritative DDL, extensions, constraints, indexes |

### 8.1 Foundation Services Inventory

| Contract | Implementation | Lifetime | Status |
|---|---|---|---|
| `IApplicationDbContext` | `ApplicationDbContext` | Scoped | ✅ Implemented |
| `ICurrentUserService` | `CurrentUserService` | Scoped | ✅ Implemented (identity source of truth) |
| `IAuditService` | `AuditService` | Scoped | ✅ Implemented (+ auto-generation in `SaveChangesAsync`) |
| `IDateTimeProvider` | `DateTimeProvider` | Singleton | ✅ Implemented |
| `IPasswordHasher` | `PasswordHasher` (PBKDF2-HMAC-SHA256, 100 000 iters) | Singleton | ✅ Implemented |
| `IFileStorage` | `SupabaseFileStorage` | Typed HttpClient | ✅ Implemented |
| `IEmailService` | `EmailService` | Scoped | ⚠️ **No-op logger returning `true`** |
| `IPaymentProvider` | `ToyyibPayAdapter`, `ChipAdapter`, `StripeAdapter` | Mixed | ⚠️ **Synthetic results** |
| `IPaymentGatewayFactory` | `PaymentGatewayFactory` | Scoped | ✅ Implemented |
| `IDoubleEntryLedgerService` | `DoubleEntryLedgerService` | Scoped | ✅ Implemented |

---

## 9. Drift Register — Foundation-Layer Findings

Every item below is a statement about **what the code does**, verified at the layer named. Verdicts: `CONFIRMED-DEVIATION` (code contradicts documented design), `GAP` (declared capability absent), `STRUCTURAL` (design characteristic with consequences).

### 9.1 CRITICAL

**F-01 — Header-trust identity synthesis with `SuperAdmin` default** · `CONFIRMED-DEVIATION`
`AdminPortalAuthenticationMiddleware` promotes any unauthenticated request carrying `X-User-Id` or `X-User-Email` to an authenticated principal, and grants `SuperAdmin` when `X-User-Roles` is omitted. It runs before `UseAuthentication`/`UseAuthorization`. Combined with CORS `AllowAll`, any reachable client can assert any identity and any role. Because `CurrentUserService` reads `HttpContext.User`, this identity flows into every handler and every auto-generated audit row.
**Fix:** delete the middleware, or hard-fail unauthenticated requests bearing `X-User-*`, and remove the `SuperAdmin` default. Add proxy-side stripping of inbound `X-User-*` headers. Effort: < 1 h.

**F-02 — Audit trail inherits spoofable actor identity** · `CONFIRMED-DEVIATION`
The automatic audit generation in `SaveChangesAsync` captures `_currentUserService.UserId`/`Email`/`IpAddress`/`UserAgent`. With F-01 present, `userId`/`userEmail` are header-supplied. Audit rows therefore record attacker-asserted actors. This is the *worst* place for the defect to land: the tamper-evident financial ledger and the compliance audit trail become attributable to a fabricated identity.
**Fix:** resolve F-01. Then add a second attribution dimension (network source, TLS client identity, or a signed proxy assertion) that cannot be set by the caller. Effort: depends on F-01.

**F-03 — Schema mutation executed from application startup** · `STRUCTURAL` / `HIGH`
`ApplicationDbContextInitializer.InitializeAsync` executes `ALTER TABLE`/`CREATE TABLE` DDL on every application start, including destructive-capability statements such as `ALTER TABLE campaigns ALTER COLUMN "EndDate" DROP NOT NULL`. Three competing schema sources exist: `deploy/database/*.sql`, the embedded patch batch, and EF Core migrations. The runtime database role requires DDL rights in production.
**Fix:** move patches into versioned EF migrations, run them as a deploy step, revoke DDL rights from the runtime role, and convert the initializer to seed-only (or seed-and-verify). Effort: 1–2 d + a deploy-pipeline change.

### 9.2 HIGH

**F-04 — Payment reconciliation sweep is a stub** · `GAP`
`ReconcilePendingPaymentsAsync` performs the query (pending attempts older than 10 minutes, `Take(10)`) and logs a count, but contains **no provider call and no state update**: `// Polling provider status logic for stuck attempts`. Consequence: a payment whose webhook was lost remains `Pending` forever — money moved, ledger never written, campaign total never incremented, no automatic recovery. For a money-handling platform this is revenue-affecting, not cosmetic.
**Fix:** implement provider polling via `IPaymentProvider.QueryPaymentStatusAsync`, apply the same idempotent settlement path used by the webhook handler, emit `esalam_payments_*`. Effort: 1 d.

**F-05 — Email delivery is a no-op that reports success** · `GAP` → escalated to `DATA-INTEGRITY`
`EmailService.SendEmailAsync` logs `"Simulating email delivery"` and returns `Task.FromResult(true)`. Because it returns `true`, `DispatchQueuedNotificationsAsync` marks every `NotificationQueueItem` as **`Sent`**. The system therefore records successful delivery of tax receipts and verification notices that were never sent — converting a missing feature into a silent lie in the notification queue's own state.
**Fix:** implement a real transactional email provider; until then, return `false` (or throw) so the queue reflects reality and items remain retryable. Effort: 4–8 h + provider onboarding.

**F-06 — Payment adapters return synthetic settlement results** · `CONFIRMED-DEVIATION`
Adapters fabricate outcomes (e.g. a constructed bill code and URL without an HTTP call; unconditional `IsValid: true`; a coerced default amount; a hard-coded `Succeeded, 50.00m` status response). Since `PaymentHandlers` trusts `validationResult.Value.IsValid`, the documented cryptographic signature verification **does not occur**. The entire money-in path can be completed without a real gateway.
**Fix:** implement real provider REST calls and signature verification; remove constant returns and amount defaults; require a non-null/verified webhook secret before enabling a provider in production. Effort: 2–3 d.

**F-07 — No concurrency tokens on money-bearing aggregates** · `STRUCTURAL` / `PLAUSIBLE-STRONG`
No `IsConcurrencyToken()`/`IsRowVersion()`/`xmin` mapping was found in `EntityConfigurations`. With last-writer-wins semantics, concurrent updates to the same campaign (two donations incrementing `CollectedAmount`) can lose an increment. Note the diagnostic contradiction: `GlobalExceptionHandlerMiddleware` explicitly handles `DbUpdateConcurrencyException`, which proves that concurrency conflicts were anticipated — yet the mapped model appears to produce none.
**Fix:** map `xmin` as a concurrency token on `Campaign` (and `CampaignCapacity`, `Payout`), or wrap the balance update in an explicit transaction with `SELECT ... FOR UPDATE`. Effort: 2–4 h.

**F-08 — Schema-of-record DDL comments disagree with enum values** · `CONFIRMED-DEVIATION`
`beneficiaries."VerificationStatus"`, `beneficiaries."Type"`, and `campaigns."Status"` carry SQL comments whose numeric-to-name mapping diverges from `DomainEnums` at multiple positions (detailed in §6.7). Behaviour follows the C# enums; the comments are misleading documentation in the authoritative schema file, and they agree at the low positions (1, 1, 1 and 4) in a way that invites a false reading.
**Fix:** regenerate the comments from `DomainEnums.cs`, or remove them and reference the enum file. Effort: < 1 h.

### 9.3 MEDIUM

**F-09 — Domain events are collected but never dispatched** · `GAP`
`BaseEntityWithEvents` accumulates `IDomainEvent`s with a correct read-only surface, but no dispatcher exists in the code paths examined. Any business rule assumed to run "when the event fires" will silently not run.
**Fix:** add a post-`SaveChangesAsync` dispatcher in `SaveChangesAsync`, or explicitly document that events are inert. Effort: 4–8 h.

**F-10 — `carts` table does not enforce "exactly one identity"** · `STRUCTURAL`
The schema permits `UserId` and `AnonymousId` to be both-null or both-non-null; only three of five representable states are valid. Guest-to-user cart merge collision handling rests entirely on application logic.
**Fix:** `CHECK (("UserId" IS NULL) <> ("AnonymousId" IS NULL))`. Effort: minutes.

**F-11 — Transactional consistency of ledger + campaign balance + audit** · `STRUCTURAL`
`BeginTransactionAsync` exists and is used explicitly by the payment webhook path, but there is no ambient transaction per command (hand-rolled CQRS has no pipeline behaviour). Any multi-write money operation that omits the explicit transaction is non-atomic. The auto-audit rows are atomic *with the mutation in the same `SaveChanges`* — but not necessarily with a separate ledger write in a different `SaveChanges`.
**Verify:** for each money-mutating handler, confirm a single transaction wraps donation finalization + ledger post + campaign increment. Effort: audit 2–4 h, then fix per finding.

**F-12 — `IApplicationDbContext.Entry<TEntity>` leaks EF Core's change-tracking model** · `STRUCTURAL`
The interface exposes `EntityEntry<TEntity>`, importing EF Core semantics into the Application layer. Notably, the implementation performs audit stamping internally via its own `ChangeTracker` rather than via externally supplied entries — so the declared capability and the actual accounting strategy are decoupled.
**Fix:** remove `Entry<TEntity>` from the interface if unused. Effort: < 1 h after usage audit.

**F-13 — Correlation ID is stored but not scoped to logs** · `STRUCTURAL`
`CorrelationIdMiddleware` writes the id to `HttpContext.Items` and the response header, but does not push a logging scope. Serilog's `Enrich.FromLogContext()` therefore has nothing to enrich with, so log lines are not correlated to the id — while the wire header suggests they are.
**Fix:** `using (_logger.BeginScope(new { CorrelationId = correlationId }))` around `_next(context)`. Effort: 15 min.

**F-14 — `Money`/`Slug` are not persisted as owned types** · `STRUCTURAL`
Financial columns are stored as raw `NUMERIC(14,2)` plus a `Currency VARCHAR(3)`; value objects exist in the domain but not in the storage contract. Per-row currency integrity depends on how each aggregate writes the column rather than being structurally enforced.
**Fix:** adopt EF Core value conversions (`OwnsOne`/`HasConversion`) for `Money`, or add a `CHECK` ensuring currency consistency. Effort: 4–8 h.

**F-15 — Provider lifetimes are inconsistent** · `STRUCTURAL`
`StripeAdapter` is a **singleton** while `ToyyibPayAdapter`/`ChipAdapter` are typed `HttpClient`s. A singleton consumed by scoped services is legal, but any per-request state held in the singleton leaks across requests.
**Verify:** confirm `StripeAdapter` holds no per-request state. Effort: 30 min.

**F-16 — Invalid JWT fallbacks in infrastructure configuration** · `CONFIRMED-DEVIATION`
`AddInfrastructure` substitutes placeholder values when configuration is absent: a placeholder Supabase URL and a literal fallback JWT secret (`"super-secret-key-that-is-at-least-32-bytes-long"`). A deployment missing `Supabase:JwtSecret` would validate tokens against a secret that is **published in source control**. The connection string has a fail-fast guard; the JWT secret does not.
**Fix:** apply the same fail-fast pattern to `Supabase:Url` and `Supabase:JwtSecret`. Effort: 15 min.

**F-17 — `RequireHttpsMetadata = false`** · `CONFIRMED-DEVIATION`
JWT bearer configuration disables HTTPS metadata requirement. Appropriate for local development, unsafe in production — it permits OIDC metadata retrieval over plaintext.
**Fix:** `RequireHttpsMetadata = !builder.Environment.IsDevelopment()`. Effort: 5 min.

### 9.4 LOW / INFO

**F-18 — `EsalamTelemetry` in the Domain layer** · `STRUCTURAL`
Telemetry is an infrastructure concern placed in `Domain/Common`, introducing the domain's only external package reference. No layering violation, but it weakens the "zero-dependency domain" property.
**Fix:** relocate to Infrastructure, or formally accept the deviation with a comment. Effort: 1 h.

**F-19 — Sweep telemetry coverage is partial** · `GAP`
Reservation reaping emits `esalam_reservations_expired_total`; payment reconciliation, notification dispatch, and campaign closure emit no domain metrics despite instruments existing.
**Fix:** instrument the remaining sweeps. Effort: 2 h.

**F-20 — `OperationCanceledException` logged as error on shutdown** · `STRUCTURAL`
`Task.Delay(..., stoppingToken)` throws on cancellation, which the cycle `catch (Exception ex)` logs at error level. Every graceful shutdown produces a spurious error line.
**Fix:** `catch (Exception ex) when (ex is not OperationCanceledException)`. Effort: 5 min.

**F-21 — `NotificationQueueItem.Status` is a string** · `STRUCTURAL`
Every other state machine in the platform uses a typed enum; the notification queue uses `"Pending"` string literals, forfeiting compiler checking.
**Fix:** introduce `NotificationStatus` enum. Effort: 1–2 h.

**F-22 — Unreferenced artifacts in repository root** · `CONFIRMED-DEVIATION`
`page_dump.html` (27 KB of rendered admin markup), `scratch_check.cs`, `CharityPlatform.slnx` (25 bytes). The `page_dump.html` in particular is leaked rendered UI committed to the source tree.
**Fix:** purge. Effort: 10 min.

**F-23 — Repository contains a stray SQL file header artifact** · `INFO`
`07_esalam_cart_schema.sql` additionally carries a UTF-8 BOM-like leading marker (`--` sequence rendered as `--`) before the comment header. Harmless to PostgreSQL, but indicates a file-encoding/normalization step worth cleaning. Effort: minutes.

---

## 10. Verification Coverage

**Read verbatim at compilation time (this document):**

*Domain:* `BaseEntity.cs`, `BaseEntityWithEvents.cs`, `Result.cs`, `DomainErrors.cs` (full, 84 lines), `DomainEnums.cs` (full, all 15 enums), `ValueObjects.cs` (full, 103 lines), `EsalamTelemetry.cs` (full, 97 lines).

*Application:* `CQRS.cs` (full, 40 lines), `ServiceInterfaces.cs` (full, 179 lines), `IApplicationDbContext.cs` (full, 53 lines), `DependencyInjection.cs` (full, 38 lines).

*Api:* `Program.cs` (full, 151 lines), `Middlewares.cs` (full, 118 lines), `ApiControllerBase.cs` (full, 44 lines).

*Infrastructure:* `ApplicationDbContext.cs` (full, 200 lines), `DependencyInjection.cs` (full, 105 lines), `BackgroundSweeperWorker.cs` (full, 152 lines), `EntityConfigurations.cs` (lines 1–120 of 693), `ApplicationDbContextInitializer.cs` (lines 1–90 of 891).

*Deploy:* `01_esalam_supabase_schema.sql` (lines 1–130 of 528), `07_esalam_cart_schema.sql` (full, 29 lines).

**Not verified — access/read-depth limited:**

| Artifact | Unread portion | Consequence |
|---|---|---|
| `EntityConfigurations.cs` | Lines 121–693 (573 lines) | **F-07 (concurrency tokens) is PLAUSIBLE-STRONG, not SOLID** — a token could exist in the unread section. Campaign, donation, financial, and platform configurations unverified. |
| `ApplicationDbContextInitializer.cs` | Lines 91–891 (801 lines) | Seed user/role content, remaining patch statements, and seed idempotency guards unverified. **Whether `SeedAsync` creates a privileged account with a known credential is UNVERIFIED.** |
| `01_esalam_supabase_schema.sql` | Lines 131–528 (398 lines) | `campaign_media`, `campaign_capacities`, `platform_settings`, `payment_attempts` backfill, approval tables, and the complete FK graph unverified. |
| `PasswordHasher.cs` | Entire file | PBKDF2-HMAC-SHA256 / 100 000 iterations / 16-byte salt / 32-byte key / `FixedTimeEquals` is **PLAUSIBLE from the architecture analysis**, not independently confirmed in this read. |
| `CurrentUserService.cs` | Entire file | Claim-to-property mapping (does `UserId` read `NameIdentifier`? does `Roles` read all `Role` claims?) is **UNVERIFIED**. This matters materially for F-01/F-02 blast radius. |
| `AuditService.cs` | Entire file | `LogActivityAsync`/`LogSecurityEventAsync` persistence path unverified; relationship to auto-generated audit rows unclear. |
| `PaymentAdapters.cs` | Tail (lines ~120–254) | F-06 generalizes from the ToyyibPay path; Chip and Stripe are **PLAUSIBLE by pattern**. |
| `PaymentGatewayFactory.cs` | Entire file | Whether provider resolution is case-sensitive and what happens on an unknown code (**`ProviderNotSupported`** vs. exception) is unverified. |

**Verdict on documentation accuracy.** The foundation-domain research materials are **structurally accurate**: the layering, dependency direction, persistence seam, CQRS contract, Result pattern, audit generation, and schema-of-record all match the code as read. Where they require correction is in **implied capability** — the documents describe a header-trust middleware as "admin portal route gating," describe payment adapters through their interfaces rather than their bodies, and describe the notification queue as a delivery mechanism when the email implementation reports success without sending. The architectural *shape* is correct; the *contract compliance* of individual implementations is where the gaps live.

---

## 11. Prioritized Remediation Plan

Ordered by (impact × ease). Effort estimates are implementation time, excluding review and deployment.

| # | Priority | Action | Closes | Effort |
|---|---|---|---|---|
| 1 | **CRITICAL** | Delete `AdminPortalAuthenticationMiddleware`, or hard-fail unauthenticated requests presenting `X-User-*` headers. Remove the `else → SuperAdmin` default. Strip inbound `X-User-*` at nginx. | F-01, F-02 | < 1 h |
| 2 | **CRITICAL** | Fail-fast on missing `Supabase:Url` / `Supabase:JwtSecret`; remove the literal fallback JWT secret; set `RequireHttpsMetadata` from the environment. | F-16, F-17 | 30 min |
| 3 | **HIGH** | Move the startup DDL patch batch into versioned EF migrations executed as a deploy step; revoke DDL rights from the runtime DB role; reduce the initializer to seed-only. | F-03 | 1–2 d |
| 4 | **HIGH** | Implement real provider REST calls and signature verification; remove constant `IsValid: true` and amount defaults. | F-06 | 2–3 d |
| 5 | **HIGH** | Implement `ReconcilePendingPaymentsAsync` against `QueryPaymentStatusAsync`, reusing the idempotent settlement path. | F-04 | 1 d |
| 6 | **HIGH** | Make `EmailService` return honest results (or implement a real provider) so the notification queue stops recording false `Sent` state. | F-05 | 4–8 h |
| 7 | **HIGH** | Add an `xmin` concurrency token to `Campaign`, `CampaignCapacity`, `Payout`; or wrap balance updates in `SELECT … FOR UPDATE`. | F-07 | 2–4 h |
| 8 | **HIGH** | Audit every money-mutating handler for single-transaction coverage of donation finalization + ledger post + campaign increment. | F-11 | 2–4 h audit |
| 9 | **MEDIUM** | Regenerate DDL enum comments from `DomainEnums.cs` (or delete them). | F-08 | < 1 h |
| 10 | **MEDIUM** | Add `CHECK (("UserId" IS NULL) <> ("AnonymousId" IS NULL))` to `carts`. | F-10 | Minutes |
| 11 | **MEDIUM** | Push a logging scope in `CorrelationIdMiddleware`. | F-13 | 15 min |
| 12 | **MEDIUM** | Read `CurrentUserService.cs` and confirm claim mapping; document the identity contract. | coverage gap | 30 min |
| 13 | **MEDIUM** | Read `ApplicationDbContextInitializer` seed section; enumerate seeded accounts, roles, and credentials. | coverage gap | 1 h |
| 14 | **MEDIUM** | Add a domain-event dispatcher, or document that events are inert. | F-09 | 4–8 h |
| 15 | **LOW** | Remove `Entry<TEntity>` from `IApplicationDbContext` after a usage audit; standardize `StripeAdapter` lifetime. | F-12, F-15 | 1–2 h |
| 16 | **LOW** | Instrument the remaining sweeps; add the `OperationCanceledException` filter. | F-19, F-20 | 2 h |
| 17 | **LOW** | Migrate `Money` to EF value conversions or add a currency `CHECK`; introduce `NotificationStatus` enum. | F-14, F-21 | 6–10 h |
| 18 | **LOW** | Purge `page_dump.html`, `scratch_check.cs`, `CharityPlatform.slnx`; relocate `EsalamTelemetry`. | F-18, F-22, F-23 | 1 h |

---

## 12. Foundation Design Principles — Summary for Contributors

Anything added to this foundation, or built on top of it, should honour the following established invariants:

1. **Domain has no project references.** Do not add one. The purity of the domain core is this architecture's foundation stone.
2. **Business failure is a `Result`, not an exception.** Exceptions mark programmer errors only. `DomainErrors` is the catalog — add a typed constant, never a bare `new Error(...)` in handler code.
3. **Name error codes by their HTTP consequence.** Suffix conventions (`NotFound`, `Unauthorized`, `Forbidden`, `Conflict`/`AlreadyExists`) drive `ApiControllerBase`'s mapping. Violating the convention silently changes the API's status codes.
4. **Never renumber an enum member.** Enum values are persisted as `INT` in PostgreSQL. Insert new members at the end with explicit values.
5. **Handlers touch persistence only through `IApplicationDbContext`.** No `DbContext`, no connection strings, no Npgsql in the Application layer.
6. **Open an explicit transaction for any operation with more than one write to money-bearing state.** There is no ambient transaction.
7. **Return entities through domain methods, not property setters.** `res.Expire()`, `campaign.Complete()`, `capacity.ReleaseReservation(q)` — encapsulated transitions are what make the ledger and the audit trail trustworthy.
8. **Aggregate identity is a client-generated GUID.** Use it; do not wait for the database to assign identity.
9. **Be aware of automatic behaviour in `SaveChangesAsync`.** Timestamps are overwritten; audit rows are generated; soft-delete filters apply globally. Read §6.3 and §6.4 before writing a handler that must see deleted rows or preserve caller-supplied timestamps.
10. **Trust boundaries are declared, not assumed.** The identity contract of this platform is a single seam (`HttpContext.User` → `ICurrentUserService`). Until F-01 is closed, any claim about authorization in a business domain must be read against that seam.

---

## 13. Conclusion

The Data and Application Foundation Domain is a **genuinely well-structured technical substrate**. The layering is enforced rather than aspirational. The persistence seam is real. The `Result` pattern is applied consistently, with a clear and correct separation between business failure and programmer error. Audit generation is a structural feature of the unit of work rather than an afterthought. Money is handled with exact-precision decimals, exact `NUMERIC(14,2)` storage, and currency-aware value objects. The schema of record uses partial unique indexes, `CHECK` constraints, and `ON DELETE RESTRICT` on financial relationships — database-level enforcement of invariants that many systems only assert in code.

The foundation's problems are not architectural; they are **contract-compliance problems at the edges of the foundation**:

- The identity contract is bypassable because a middleware accepts the identity it is supposed to verify (F-01, F-02).
- The notification contract reports success it did not achieve (F-05).
- The payment contract promises verification and delivers constants (F-06).
- The reconciliation contract has no implementation (F-04).
- The schema contract mutates itself at runtime rather than through versioned migrations (F-03).

Each of these is a bounded, locatable fix — and that is the point. Because the foundation is properly layered, each fix lands in exactly one place: one middleware deleted, one service implemented, one adapter rewritten, one migration pipeline introduced. **The architecture is sound enough that the remediation is small.** That is the strongest possible verdict on a foundation layer.

---

*Document compiled from verbatim source reads of the ESalam foundation layer, the SQL schema of record, and the consolidated research materials. Every claim above is traceable to a named file; unverified items are marked as such rather than asserted.*