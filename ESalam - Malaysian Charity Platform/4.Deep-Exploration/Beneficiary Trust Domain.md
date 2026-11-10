# Beneficiary Trust Domain — Technical Documentation

**Project:** ESalam Charity & Donation Management Platform
**Domain Classification:** Core Business Domain
**Importance Rating:** 9.0 / 10
**Complexity:** 6.5 / 10
**Code Slice:** `src/CharityPlatform.Domain/Entities/Beneficiaries/*`, `Application/Beneficiaries/*`, `Api/Controllers/Beneficiaries/*`, `Contracts/Beneficiaries/*`
**Document Status:** Verified against source research materials

---

## 1. Executive Summary

The **Beneficiary Trust Domain** is the ESalam platform's gatekeeper for legitimate charitable organizations. It models every entity that can *receive* donated funds — a charitable organization, NGO, mosque, orphanage, or individual relief recipient — as a first-class **aggregate root** with legal identity, banking details, KYC media, and a **KYB-style (Know-Your-Business) verification state machine**.

In a donation platform, this domain answers a single uncompromising question:

> *"Is the entity that will receive these funds verified, legal, and accountable?"*

The answer is enforced *structurally*, not by convention. A beneficiary cannot become a campaign recipient unless it has explicitly passed through the `Verify()` transition. The verification status has **no public setter** anywhere in the codebase — every state change must flow through an intention-revealing aggregate method (`Verify`, `Reject`, `Suspend`) that stamps the acting user, the timestamp, and the reason directly into the entity. This makes the domain the deepest upstream dependency in the platform's money chain:

```
Beneficiary Verification → Campaign Creation → Donation Checkout → Payout Disbursement
        (this domain)         (gated by KYB)      (trusts campaigns)   (pays verified orgs)
```

If this domain is compromised, every downstream control becomes cosmetic: unverified recipients get attached to campaigns, collect donations, and receive payouts through an otherwise-correct approval workflow.

---

## 2. Business Context & Value Proposition

### 2.1 The Problem It Solves

Malaysian charitable fundraising suffers from a trust fragmentation problem: donors cannot easily distinguish a legitimate religious foundation from a fraudulent collection drive. Regulatory and reputational risk concentrates at the point where money leaves the platform. ESalam's architectural answer is to make trust a **prerequisite, not a post-hoc check** — the platform refuses to attach funds-destination metadata to any entity that has not passed verification.

### 2.2 Strategic Responsibilities

| Responsibility | Business Meaning |
|---|---|
| **Legal Identity Custody** | Stores name, slug, registration number, and beneficiary classification — the entity's legal fingerprint on the platform |
| **KYC Evidence Store** | Holds organizational documents and media submitted for review (`BeneficiaryMedia`) |
| **Banking Settlement Routing** | Holds `BankName`, `BankAccountNumber`, `BankAccountHolder` — the destination for payout disbursements |
| **Verification Adjudication** | Runs the KYB state machine that converts a *pending* organization into an *eligible* funds recipient |
| **Historical Integrity** | Soft-deletion preserves campaign associations and settlement records even after an organization is retired |
| **Public Discoverability** | Serves SEO-friendly slug-or-id resolution for public NGO detail pages |

### 2.3 Position in the Domain Graph

From the domain-relations analysis, the Beneficiary Trust Domain participates in exactly one inbound dependency — but it is a load-bearing one:

| From | Relation | Strength | Meaning |
|---|---|---|---|
| Fundraising Campaign Domain | Data Dependency | **8.5 / 10** | A campaign designates a verified beneficiary; only beneficiaries that passed the state machine may be attached |
| Operations Console Domain | Service Call | 8.0 / 10 | Admin console reads beneficiary records and applies verification decisions through the handlers |

The domain's **outbound** dependency is minimal (persistence via `IApplicationDbContext`, media resolution via `IFileStorage`), which is architecturally desirable: the KYB gate is intentionally *not* entangled with payment or approval logic. It is a pure trust boundary — the only thing it trusts is its own state machine.

---

## 3. Domain Model

### 3.1 Aggregate Structure

```
Beneficiary (Aggregate Root)
├── BaseEntity           → Id, CreatedAt, UpdatedAt, audit timestamps
├── ISoftDeletable       → IsDeleted, DeletedAt (query-filtered by EF Core)
│
├── Legal Identity Block
│   ├── Name                    (private set)
│   ├── Slug                    (private set — SEO identifier)
│   ├── RegistrationNumber      (private set — legal registration)
│   └── BeneficiaryType         (private set — classification)
│
├── Contact Block
│   ├── ContactPersonName
│   ├── Email
│   ├── PhoneNumber
│   ├── Address
│   ├── State
│   └── Postcode
│
├── Banking Settlement Block
│   ├── BankName
│   ├── BankAccountNumber
│   └── BankAccountHolder       ← payout destination
│
├── Verification Block
│   ├── BeneficiaryVerificationStatus   (private set — NO public setter)
│   ├── VerifiedAt
│   ├── RejectionReason
│   └── acting-user stamps on transitions
│
├── Media Collection (1:N)
│   └── BeneficiaryMedia[]
│       ├── MediaUrl
│       ├── MediaType           (default "Image", alternative "Video")
│       ├── FileName / FileFormat
│       ├── DisplayOrder
│       └── IsCover
│
└── Behavior Methods (only sanctioned mutations)
    ├── Verify(actingUser)
    ├── Reject(actingUser, reason)
    └── Suspend(actingUser, reason)
```

### 3.2 Encapsulation Discipline

The domain implements **strict invariant protection** through three mechanisms:

**1. Private setters everywhere.** Every property listed above carries a `private set`. EF Core materializes the entity through a private parameterless constructor (a standard ORM accommodation), but *call sites outside the aggregate have no write access*. This is not a stylistic preference — it is the mechanism that makes the KYB gate enforceable.

**2. Intention-revealing transition methods.** The three public behavior methods are the entire mutation surface for verification:

| Method | Effect | Stamps |
|---|---|---|
| `Verify(actingUser)` | Status → **Verified** | `VerifiedAt`, acting identity |
| `Reject(actingUser, reason)` | Status → **Rejected** | `RejectionReason`, acting identity |
| `Suspend(actingUser, reason)` | Status → **Suspended** | reason, acting identity |

Because the status property is unreachable, a controller-level bug, a misconfigured interface, or a future handler that "accidentally" assigns a status cannot bypass the state machine. This is a textbook example of making an invariant **structurally impossible to violate** rather than **procedurally prohibited**.

**3. Soft deletion over hard deletion.** `ISoftDeletable` means a retired organization never orphans its historical campaign associations or settlement records. EF Core query filters hide it from normal reads while the audit trail and financial history remain intact — a hard deletion here would break ledger reconciliation for every past payout.

### 3.3 The Child Entity — `BeneficiaryMedia`

`BeneficiaryMedia` extends `BaseEntity` and holds the one-to-many relationship (`BeneficiaryId` + navigation property). Its mutation surface is deliberately narrow:

| Method | Purpose |
|---|---|
| `UpdateCaption(string?)` | Set or clear the caption |
| `SetDisplayOrder(int order)` | Reposition within the gallery |
| `SetCover(bool isCover)` | Designate as the gallery cover image |

Critically, **media URLs are resolved through `IFileStorage`, not stored as domain concerns.** The aggregate knows a media item *exists* and how it is *presented*; it does not know which storage provider hosts it. Swapping Supabase Storage for S3 or a CDN requires zero changes to the beneficiary aggregate or its handlers — the abstraction holds at the application/API boundary.

*(Note: media items serve dual purpose — KYC evidence documents consumed by compliance reviewers, and organizational imagery displayed on public NGO pages.)*

---

## 4. Verification State Machine (The KYB Gate)

### 4.1 States and Transitions

```
                    ┌─────────────────────┐
                    │  PENDING (creation) │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
         Verify()         Reject()        (no action)
              │                │
              ▼                ▼
    ┌─────────────────┐  ┌─────────────┐
    │    VERIFIED     │  │  REJECTED   │
    │  (eligible)     │  │ (ineligible)│
    └────────┬────────┘  └─────────────┘
             │                 ▲
             │ Suspend()       │ (creator may resubmit)
             ▼                 │
    ┌─────────────────┐        │
    │   SUSPENDED     │────────┘
    │ (de-eligible)   │
    └─────────────────┘
```

### 4.2 State Semantics

| State | Attachable to Campaign? | Receives Payouts? | Business Meaning |
|---|---|---|---|
| **Pending** | ❌ No | ❌ No | Freshly registered, awaiting compliance review |
| **Verified** | ✅ **Yes** | ✅ **Yes** | Passed KYB; legal identity and banking details adjudicated |
| **Rejected** | ❌ No | ❌ No | Failed review; creator notified, may resubmit |
| **Suspended** | ❌ No | ❌ No | Previously verified; removed from eligibility (regulatory action, misconduct, or organizational change) |

### 4.3 Enforcement Chain

The invariant *"only verified beneficiaries can receive campaign funds"* is enforced across three layers with no single point of bypass:

```
Layer 1 — Domain:      No public setter for BeneficiaryVerificationStatus
                       All transitions flow through Verify/Reject/Suspend
                       Actor + reason + timestamp stamped at the entity

Layer 2 — Application: UpdateBeneficiaryStatusCommand → Handler
                       Current-user context passed into the transition
                       Persistence via IApplicationDbContext

Layer 3 — Consumer:    Campaign association queries beneficiary by
                       id / slug-or-id and must observe
                       VerificationStatus == Verified before attaching
```

Layers 1 and 2 are verified in the source materials. Layer 3's *precise* implementation (whether campaign creation hard-rejects non-verified beneficiaries at the handler level, at the DB constraint level, or only in UI selection) is **PLAUSIBLE but unconfirmed** from the available read depth — see §8.

---

## 5. Component Architecture

### 5.1 Layer Allocation

| Layer | Artifact | Responsibility |
|---|---|---|
| **Domain** | `Beneficiary.cs` | Aggregate root; identity/banking/contact state; state machine |
| **Domain** | `BeneficiaryMedia.cs` | Child entity; gallery ordering and cover designation |
| **Application** | `BeneficiaryHandlers.cs` | CQRS query/command handlers; filtering, pagination, statistics, DTO mapping |
| **Contracts** | `BeneficiaryContracts.cs` | `BeneficiaryDto`, `UpdateBeneficiaryProfilePhotoRequest`, `UpdateBeneficiaryCoverImageRequest` |
| **Api** | `BeneficiariesController.cs` | 17 HTTP actions; validation branches; uniform result mapping |
| **Infrastructure** | `IApplicationDbContext`, `IFileStorage` | Abstraction boundaries consumed by handlers |

### 5.2 CQRS Surface

**Queries**

| Query | Key | Purpose |
|---|---|---|
| `GetBeneficiariesQuery` | filters + pagination | List view with aggregated `ActiveCount` / `TotalRaised` statistics |
| `GetBeneficiaryByIdQuery` | `Guid` | Internal lookup by primary key |
| `GetBeneficiaryBySlugOrIdQuery` | `string` | Public/SEO-friendly route resolution |

**Commands**

| Command | Handler Behavior |
|---|---|
| `CreateBeneficiaryCommand` | Constructs `Beneficiary` in pending state; persists via `IApplicationDbContext` |
| `UpdateBeneficiaryCommand` | Mutates identity, contact, banking, profile visuals, and media gallery |
| `UpdateBeneficiaryStatusCommand` | Drives `Verify` / `Reject` / `Suspend` with current-user + reason |

All handlers project entities through a shared `MapToDto` helper — a **single mapping source of truth**, which eliminates the classic DTO-drift class of defect where list and detail views diverge.

### 5.3 API Surface — Functionally Grouped

The `BeneficiariesController` exposes 17 actions grouped into six functional families:

| Family | Operations | Notes |
|---|---|---|
| **Onboarding** | Create beneficiary | `CreateBeneficiaryCommand` payload |
| **Profile Maintenance** | Update identity / contact / banking | `UpdateBeneficiaryCommand` |
| **Compliance Verification** | Set verification decision | Maps to `UpdateBeneficiaryStatusCommand` → aggregate transitions |
| **Profile Visuals** | Update profile photo / cover image | Typed by narrow request records; validation branches return `ApiResponse.Fail` |
| **Media Gallery** | Add / list / update / remove | Over `BeneficiaryMedia`, with upload validation |
| **Bank Settlement** | Read / update banking details | The payout routing destination |
| **Read Surface** | List (paginated) / by-id / by-slug-or-id | Public + internal consumption |

Every action funnels through a recurring `HandleResult` helper that adapts handler `Result` objects into MVC action results, and every response is shaped through `CharityPlatform.Contracts.Common` envelopes. **The wire format is uniform across success, failure, and validation outcomes** — clients never need to branch on response shape.

---

## 6. Interaction Model

### 6.1 Onboarding Sequence

```
Compliance Admin → POST create beneficiary (legal identity, contact, banking)
  → BeneficiariesController
  → CreateBeneficiaryCommand
  → Handler.Handle(command, currentUser)
  → new Beneficiary(...)  [pending state]
  → IApplicationDbContext.Add + SaveChanges
  → BeneficiaryDto
  → HandleResult(...) → 200 ApiResponse<BeneficiaryDto>
```

### 6.2 Verification Sequence (the critical path)

```
Compliance Admin → PUT verification decision (verify / reject / suspend)
  → BeneficiariesController
  → UpdateBeneficiaryStatusCommand
  → Handler.Handle(command, currentUser, reason)
  → Beneficiary.Verify(actor)
     | Beneficiary.Reject(actor, reason)
     | Beneficiary.Suspend(actor, reason)
  → [state machine enforces transition; stamps VerifiedAt / RejectionReason / actor]
  → IApplicationDbContext.SaveChanges
  → BeneficiaryDto → 200 ApiResponse
```

The verification decision is **never** a property assignment. It is always a method invocation on the aggregate, carrying the acting identity and the rationale into the entity itself — which is what makes the compliance trail auditable at the point of mutation rather than reconstructed from peripheral logs.

### 6.3 Media & Profile Asset Sequence

```
Compliance Admin → upload profile photo / cover image / gallery media
  ├── invalid file or media type → 400 ApiResponse.Fail  [guard runs BEFORE dispatch]
  └── valid
      → UpdateBeneficiaryCommand / media command
      → Handler.Handle(...)
      → BeneficiaryMedia.UpdateCaption / SetDisplayOrder / SetCover
      → IFileStorage: store blob, resolve MediaUrl
      → IApplicationDbContext.SaveChanges
      → BeneficiaryDto (with media) → 200 ApiResponse
```

Validation guards execute **before** command dispatch — invalid uploads never reach the domain or the storage provider, which prevents orphaned blobs and partial state.

### 6.4 Campaign Association (Cross-Domain Read Path)

```
Campaign Domain → GET beneficiary by id / slug-or-id
  → GetBeneficiaryByIdQuery / GetBeneficiaryBySlugOrIdQuery
  → Handler.Handle(query)
  → IApplicationDbContext (filters + pagination)
  → Beneficiary (+ media, statistics)
  → MapToDto → BeneficiaryDto
  → 200 ApiResponse<BeneficiaryDto>
  → Campaign Domain: attach only if VerificationStatus == Verified
```

### 6.5 Soft Deletion

```
Compliance Admin → DELETE beneficiary
  → soft-delete command
  → Handler.Handle(...)
  → mark ISoftDeletable (EF Core query filters hide it)
  → 200 ApiResponse
```

Historical campaign associations and settlement records remain intact and reconcilable.

---

## 7. Architectural Assessment

### 7.1 Strengths

**Structural invariant enforcement.** The absence of a public status setter is the single most valuable design decision in this domain. It converts "developers must remember to call `Verify()`" into "the compiler prevents any other option." This is the difference between a policy and a mechanism.

**Exceptionally clean layering.** The domain has zero outbound dependencies beyond `BaseEntity` and `ISoftDeletable`. It does not know about EF Core, storage providers, HTTP, or CQRS. Media URL resolution is pushed to the application/API boundary via `IFileStorage`. This makes the KYB state machine trivially unit-testable and immune to infrastructure churn.

**Dual-purpose media entity.** `BeneficiaryMedia` serves both KYC evidence (compliance consumption) and public organizational imagery (donor-facing consumption) through one entity with ordering and cover semantics. This avoids a parallel document table while keeping presentation concerns (order, cover) as explicit domain operations.

**Single mapping source of truth.** The shared `MapToDto` helper means list, detail, and slug-resolved views cannot drift. Combined with the uniform `HandleResult` envelope, client integration is predictable.

**Soft deletion preserves audit continuity.** For a compliance-sensitive domain, the ability to retire an organization without destroying its financial history is not a convenience — it is a regulatory requirement.

**Statistics aggregation at the read boundary.** `ActiveCount` and `TotalRaised` are computed server-side during list queries rather than requiring client-side joins, keeping the admin console's beneficiary management view efficient.

### 7.2 Risk Observations

| # | Observation | Impact | Evidence Status |
|---|---|---|---|
| **B-01** | **Verification authorization is not evidenced at the controller level.** The research materials describe the status command flowing to the aggregate, but do not confirm that a role-policy attribute (e.g. `RequireReviewer` / `RequireSuperAdmin`) gates the verification endpoint specifically. Combined with the platform-wide **header-trust admin authentication (D-01/DRIFT)** — where `AdminPortalAuthenticationMiddleware` promotes unauthenticated requests carrying `X-User-Id`/`X-User-Email` and defaults to `SuperAdmin` when `X-User-Roles` is absent — an attacker who can reach the API directly may be able to invoke `Verify()` on an arbitrary beneficiary. | **CRITICAL** if confirmed | **UNVERIFIABLE** — controller attribute usage unread; drift finding is CONFIRMED system-wide |
| **B-02** | **Banking detail mutation shares the update command with identity/contact fields.** `UpdateBeneficiaryCommand` handles identity, contact, banking, visuals, and media. A payout-stopping attacker only needs to reach this one command — there is no separate, more-heavily-gated path for `BankAccountNumber`. | **HIGH** | CONFIRMED (single command handles all update families) |
| **B-03** | **Post-verification mutation is not evidenced as re-gated.** Once `Verified`, can the banking block be changed without triggering re-review? The domain has no `RequireReverification()` transition and no evident "material change invalidates verification" rule. | **HIGH** — classic verified-then-swap attack: pass KYB with a legitimate account, then redirect settlement to an attacker-controlled account | **UNVERIFIABLE** from available read depth |
| **B-04** | **Slug uniqueness and slug-change policy unverified.** `GetBeneficiaryBySlugOrIdQuery` supports public routing; if slugs are mutable without constraint, an organization could assume another's public identity, or a squatted slug could redirect donor traffic. | MEDIUM | **UNVERIFIABLE** — EF index declarations in `EntityConfigurations.cs` unread |
| **B-05** | **`Suspend()` semantics for re-instatement unclear.** No `Reinstate()` transition is documented. If a suspended beneficiary cannot be restored, the only recovery path is creating a duplicate organization record — which would fragment settlement history. | MEDIUM | CONFIRMED (only Verify/Reject/Suspend exist) |
| **B-06** | **Media upload validation depth unverified.** The controller returns `ApiResponse.Fail` for invalid uploads, but the *criteria* (MIME allowlist, size cap, extension/content mismatch checks, path traversal in `FileName`) are not evidenced. Media upload is a recurring RCE and stored-XSS vector. | MEDIUM–HIGH | **UNVERIFIABLE** — validation branches referenced but rules unread |
| **B-07** | **Rejection reason is free-text with no structured taxonomy.** `RejectionReason` is a plain string. Over time this degrades compliance reporting (no aggregate "rejected for X% missing registration documents"). | LOW | CONFIRMED |
| **B-08** | **No documented re-verification cadence.** KYB is treated as a one-time gate. Real-world charity regulation typically requires periodic re-attestation (annual filings, officer changes, license renewal). The domain has no `Reverify()` or review-due timestamp. | MEDIUM | CONFIRMED (no such state or field exists) |

### 7.3 The Consolidated Attack Chain (if B-01 through B-03 hold)

```
Step 1 — Reach API directly (bypass nginx UI, if network path exists)
Step 2 — Send request with X-User-Id / X-User-Email and no X-User-Roles
         → AdminPortalAuthenticationMiddleware synthesizes SuperAdmin principal
              [D-01, CONFIRMED system-wide]
Step 3 — PUT verification decision → Beneficiary.Verify(attacker)
         → organization becomes Verified, VerifiedAt + actor stamped
Step 4 — PUT update → overwrite BankName / BankAccountNumber / BankAccountHolder
         → settlement routed to attacker account
              [B-02 + B-03, plausible]
Step 5 — Campaign created against newly verified beneficiary
Step 6 — Donations collected; payout approved by a legitimate multi-step chain
         → funds disbursed to attacker's account
         → ledger entries balanced, hashed, and auditable — the evidence trail
           is pristine and points at the attacker's own headers
```

**The severity here is architectural, not incidental.** Every downstream control — the double-entry ledger, the chained SHA-256 audit trail, the multi-step approval workflow — works exactly as designed. They simply execute faithfully on top of a falsified trust root. This is why the Beneficiary Trust Domain's `importance: 9.0` rating is accurate: it is the cheapest place in the entire money chain to inject an illegitimate funds recipient.

---

## 8. Verification Coverage & Open Gaps

The following are explicitly **not confirmed** from the available research materials. Each is stated as a gap rather than assumed correct, per evidence standards:

**Not verified — access-limited:**
- **Role-policy attributes on `BeneficiariesController` actions** — the materials describe 17 actions, a `HandleResult` helper, and validation branches, but do not itemize `[Authorize(Policy = ...)]` usage per action. Whether verification and banking endpoints are policy-gated is **UNVERIFIABLE**.
- **Campaign-side enforcement of the verified-only invariant** — the relation "only beneficiaries that passed verification may be attached" is described as an architectural intent. Whether it is enforced by a domain check in `CampaignHandlers`, a database foreign-key/constraint, a filtered query, or only by UI selection is **UNVERIFIABLE** from the current read depth (`CampaignHandlers.cs` is 40 KB; only headers examined).
- **`EntityConfigurations.cs` (29 KB)** — unique indexes on slug and registration number, and any concurrency tokens, are unread. Slug/registration uniqueness is therefore **UNVERIFIABLE**.
- **Media validation rules** — MIME allowlist, size limits, and filename sanitization are referenced by response branches but the rule bodies are **UNVERIFIABLE**.
- **`ApplicationDbContextInitializer.cs` (53 KB)** — seeded beneficiary records and default verification states are unread.
- **Whether `Suspend()` is reachable from `Verified` only, or also from `Pending`/`Rejected`** — the transition graph in §4.1 is derived from the state semantics; the precise guard clauses inside the aggregate methods are **PLAUSIBLE but unconfirmed**.

**Verdict on documentation accuracy for this domain:** The research materials describe the domain's *structure* with high fidelity — aggregate composition, encapsulation approach, CQRS surface, and API families all match the described implementation. The materials are accurate about what the domain **is**. They are silent on the two questions that determine its **real-world security value**: (a) who is authorized to invoke the verification transition, and (b) whether post-verification banking mutation is re-gated. Those gaps are the domain's material risk surface.

---

## 9. Recommendations

| # | Priority | Action | Closes | Effort |
|---|---|---|---|---|
| 1 | **CRITICAL** | Apply an explicit role policy (`RequireReviewer` or a new `RequireKybOfficer`) to the verification-status action and every banking-detail mutation. Verify the attribute is present on the action, not merely on the controller. | B-01 | 1–2 h |
| 2 | **CRITICAL** | Fix the platform-wide header-trust authentication drift (`AdminPortalAuthenticationMiddleware` SuperAdmin default; `AdminApiClient` hard-coded context). Until closed, no controller-level policy is trustworthy. | B-01 (root cause) | < 1 h per D-01/D-02 remediation |
| 3 | **HIGH** | Split `BankAccountNumber`, `BankAccountHolder`, and `BankName` out of `UpdateBeneficiaryCommand` into a dedicated, more heavily gated `UpdateBeneficiarySettlementDetailsCommand` with its own authorization and audit event. | B-02 | 4–8 h |
| 4 | **HIGH** | Introduce a **verification-invalidating change rule**: any mutation of legal identity or banking details sets status back to `Pending` and requires re-approval. Add a `RequireReverification()` aggregate method so the rule lives in the domain, not in handler glue. | B-03 | 1–2 d |
| 5 | **HIGH** | Harden media upload validation: MIME allowlist enforced by content sniffing (not `Content-Type` header), size cap, filename sanitization, and rejection of executable/script extensions. | B-06 | 4–8 h |
| 6 | **MEDIUM** | Add unique constraints on `Slug` and `RegistrationNumber` at the EF Core configuration and database level; document the slug-immutability policy (recommended: immutable after first publication, with explicit redirect handling on change). | B-04 | 2–4 h |
| 7 | **MEDIUM** | Add a `Reinstate(actingUser, reason)` transition so a suspended organization can be restored without creating a duplicate record and fragmenting settlement history. | B-05 | 2–4 h |
| 8 | **MEDIUM** | Add `LastVerifiedAt` and `NextReviewDueAt` fields plus a background sweep that flags expired verifications into a compliance queue. Pair with the existing `BackgroundSweeperWorker` pattern. | B-08 | 1 d |
| 9 | **LOW** | Replace free-text `RejectionReason` with a structured reason code (plus optional free-text) to enable compliance aggregate reporting. | B-07 | 4 h |
| 10 | **LOW** | Add an explicit domain-level guard assertion: `Campaign.AssignBeneficiary()` refuses any beneficiary whose `VerificationStatus != Verified`, throwing a domain error rather than relying on upstream query discipline. | Hardens §4.3 Layer 3 | 2–4 h |

---

## 10. Summary

The Beneficiary Trust Domain is a **small, well-encapsulated, architecturally clean module** that carries outsized business weight. Its design correctly identifies trust as a *structural* property: the verification state machine cannot be bypassed by property assignment, the domain is free of infrastructure entanglement, and soft deletion preserves the audit continuity that a compliance-sensitive domain requires.

Its material risk is not in its internal design but at its **boundary**: the domain correctly refuses to grant eligibility without `Verify()`, but the research materials do not evidence *who is permitted to call* `Verify()`, nor whether a verified organization's banking details can subsequently be redirected without re-review. Given the platform-wide authentication drift confirmed elsewhere in the architecture (header-trust admin middleware defaulting to `SuperAdmin`), the KYB gate's real-world strength currently rests on assumptions that should be explicitly verified and, where found wanting, closed.

> **Domain verdict:** Sound construction, correct invariant philosophy, unverified perimeter. The state machine is the strongest component; the authorization and post-verification mutation policy are the weakest, and they are the two things that determine whether this gate actually holds.

---

*Document compiled from source-verified research materials. Items marked UNVERIFIABLE are stated as gaps, not assumed correct. Evidence references: `Beneficiary.cs`, `BeneficiaryMedia.cs`, `BeneficiaryHandlers.cs`, `BeneficiariesController.cs`, `BeneficiaryContracts.cs`, cross-referenced against the ESalam system architecture drift analysis.*