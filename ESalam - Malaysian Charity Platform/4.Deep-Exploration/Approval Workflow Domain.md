I have full evidence coverage of the Approval Workflow Domain. Compiling the documentation now.

---

# ESalam — Approval Workflow Domain Documentation

**Document classification:** Domain Module Specification
**Domain type:** Core Business Domain
**Complexity rating:** 8.0 / 10
**Importance rating:** 9.5 / 10
**Evidence base:** Verbatim source read of the domain entities, application use cases, contracts, and API controller
**Generation time (UTC):** 2026-11-10 03:05:25

---

## 1. Executive Summary

The Approval Workflow Domain is ESalam's **governance engine** — the single mechanism that decides whether a fundraising campaign may become publicly donatable and whether collected funds may leave the platform. It is a reusable, template-driven, multi-step, cyclic, and escalation-capable workflow engine that governs **two distinct business objects** through **one state machine**: campaign moderation and payout disbursement.

The architectural narrative is deliberately narrow: **a template defines who must approve, an instance records that they did.** `PositionMaster` is the reusable template carrying applicability rules (monetary bounds, category scope, beneficiary scope). `PositionDefinition` binds a template to a real organizational role or a named individual, with validity windows. `ApprovalPosition` is the runtime instance — the state machine — and it stores reviewer identity, timestamps, comments, escalation lineage, and cycle number **inside the domain entity itself**, not in a peripheral audit table.

This is the strongest design property of the domain: **the audit trail is the entity.** If an approval happened, the evidence is on the object that had to change for the approval to take effect. There is no window in which a decision is recorded in one place and the effect in another. Combined with the automatic `AuditLog` generation in `ApplicationDbContext.SaveChangesAsync`, every transition produces two independent, correlated records — one semantic, one mechanical.

The domain's blast radius is the reason it carries a 9.5 importance rating. It is the gate on **both** governance boundaries simultaneously. A single defect in the state machine does not compromise one boundary — it compromises campaign admission *and* fund release at the same time.

---

## 2. Domain Position Within the Platform

### 2.1 Layering

The domain obeys the strict inward dependency direction of the Clean Architecture:

```
Domain  ←  Application  ←  Infrastructure / Api
   ↑            ↑
   └── Contracts (shared DTO boundary, consumed by Api and both front ends)
```

Concretely:

| Layer | Artifact | Responsibility |
|---|---|---|
| **Domain** | `Entities/Campaigns/ApprovalPosition.cs`, `PositionMaster.cs`, `PositionDefinition.cs` | Invariants, state transitions, applicability predicates, authorization predicates |
| **Application** | `Campaigns/ApprovalPositionUseCases.cs`, `Financial/PayoutApprovalUseCases.cs` | CQRS use cases, step generation, authorization orchestration, terminal side effects |
| **Contracts** | `Campaigns/ApprovalWorkflowContracts.cs` | Request/response shapes crossing the API boundary |
| **Api** | `Controllers/Campaigns/ApprovalPositionsController.cs` | Attribute-routed HTTP surface, uniform result mapping |

The Domain layer references **no other project** — it holds only business rules. The Application layer holds no EF Core query *shapes* beyond `IApplicationDbContext`; the Api layer holds no business rules at all.

### 2.2 Cross-Domain Relationships

| Counterparty domain | Relationship | Strength | Mechanism |
|---|---|---|---|
| Fundraising Campaign | Data Dependency (inbound) | 8.0 | `ApprovalPosition.CampaignId` gates campaign publication; approval of all steps drives `Campaign.Approve()` |
| Financial Governance | Service Call (inbound) | 9.5 | `ApprovalPosition.PayoutId` gates disbursement; terminal approval invokes `IDoubleEntryLedgerService.PostPayoutDisbursementAsync` |
| Identity & Access | Service Call (outbound) | 9.0 | `ICurrentUserService` supplies `UserId` and `Roles`; `PositionDefinition.CanUserFulfill` resolves authority |
| Operations Console | Service Call (inbound) | 8.5 | AdminWeb approval dialogs dispatch step commands and render step chains |
| Data & Application Foundation | Data Dependency | 9.0 | Persistence via `IApplicationDbContext`; `Result` pattern for control flow; `BaseEntity` for identity/timestamps |

The dual association is the defining structural feature. `ApprovalPosition` carries **nullable** `CampaignId` and **nullable** `PayoutId` foreign keys. One entity, two governed object types, one transition authority.

---

## 3. Domain Model

### 3.1 Template Layer — `PositionMaster`

`PositionMaster` is the reusable definition of an approval position. It is fully encapsulated: a private parameterless constructor for EF Core materialization, and a static `Create(...)` factory as the only legitimate construction path. All mutation flows through `Update(...)` or `SetActive(...)`.

| Property | Type | Purpose |
|---|---|---|
| `Name` | `string` | Human-readable position label |
| `Code` | `string` | Unique identifier, normalized to `UPPER` and trimmed |
| `Description` | `string?` | Optional administrative notes |
| `WorkflowType` | `PositionWorkflowType` | **Discriminator**: `Campaign` or `Payout` |
| `ParentPositionId` | `Guid?` | Self-referencing hierarchy link |
| `HierarchyOrder` | `int` | Ordering key; **tier 1 is the universal baseline** |
| `MinTargetAmount` | `decimal` | Activation threshold for tiers above 1 |
| `MaxTargetAmount` | `decimal?` | Declared upper bound |
| `IsActive` | `bool` | Soft participation switch |
| `CategoryId` | `Guid?` | Optional campaign-category scoping |
| `BeneficiaryId` | `Guid?` | Optional beneficiary scoping |

**Applicability predicates.** Two methods encode whether a template participates in a chain.

```csharp
public bool IsApplicable(decimal targetAmount, Guid? campaignCategoryId, Guid? campaignBeneficiaryId)
{
    if (!IsActive) return false;
    if (WorkflowType != PositionWorkflowType.Campaign) return false;

    if (HierarchyOrder > 1 && targetAmount < MinTargetAmount)
        return false;

    if (CategoryId.HasValue && CategoryId.Value != campaignCategoryId)
        return false;

    if (BeneficiaryId.HasValue && BeneficiaryId.Value != campaignBeneficiaryId)
        return false;

    return true;
}
```

`IsApplicableForPayout(decimal payoutAmount, Guid? beneficiaryId)` is the payout counterpart, identical in structure but omits category matching (payouts carry no campaign category dimension).

The `HierarchyOrder > 1` guard is important and easy to misread. It means **tier 1 activates unconditionally regardless of amount**, while every higher tier requires `targetAmount >= MinTargetAmount`. The template set therefore always yields at least one baseline review step — a deliberate anti-bypass property, reinforced in the generation helpers by an explicit fallback (see §4.2).

### 3.2 Assignment Layer — `PositionDefinition`

`PositionDefinition` answers the question: *who, specifically, may fulfil this position, and when.* Two static factories enforce the assignment invariant:

| Factory | Semantics |
|---|---|
| `CreateForRole(positionMasterId, roleName, validFrom, validTo, isActive)` | `AssignmentType = Role`, `UserId = null` |
| `CreateForUser(positionMasterId, userId, validFrom, validTo, isActive)` | `AssignmentType = User`, `RoleName = null` |

Temporal validity and authority resolution are encapsulated:

```csharp
public bool IsCurrentlyActive(DateTime? referenceDateUtc = null)
{
    if (!IsActive) return false;
    var date = referenceDateUtc ?? DateTime.UtcNow;
    if (date < ValidFrom) return false;
    if (ValidTo.HasValue && date > ValidTo.Value) return false;
    return true;
}

public bool CanUserFulfill(string userId, IEnumerable<string> userRoles, DateTime? referenceDateUtc = null)
{
    if (!IsCurrentlyActive(referenceDateUtc)) return false;

    if (AssignmentType == PositionAssignmentType.User)
        return string.Equals(UserId, userId, StringComparison.OrdinalIgnoreCase);

    if (AssignmentType == PositionAssignmentType.Role && !string.IsNullOrWhiteSpace(RoleName))
        return userRoles.Any(r => string.Equals(r, RoleName, StringComparison.OrdinalIgnoreCase));

    return false;
}
```

Two properties matter operationally. First, **validity is evaluated at decision time, not at assignment time** — a definition that has expired cannot authorize a transition even if a step instance was bound to it. Second, both `AssignmentType == Role` with a null/whitespace `RoleName` and any unrecognized assignment shape **fail closed** (`return false`). There is no permissive default branch.

Third — and this is the security-relevant structural fact — `CanUserFulfill` is the **only** authority check in the domain. Every approve, reject, and escalate handler funnels through it, which is why the design can claim a single transition authority across both governed object types.

### 3.3 Instance Layer — `ApprovalPosition`

`ApprovalPosition` is the runtime state-machine instance. It links an *approvable* (polymorphic: `CampaignId` or `PayoutId`), a `PositionMasterId`, and carries the workflow coordinates.

| Property | Type | Purpose |
|---|---|---|
| `CampaignId` / `Campaign` | `Guid?` / `Campaign?` | Campaign-moderation target |
| `PayoutId` / `Payout` | `Guid?` / `Payout?` | Payout-disbursement target |
| `PositionMasterId` / `PositionMaster` | `Guid` / `PositionMaster` | Template binding |
| `StepOrder` | `int` | Position in the ordered chain (sequential gate) |
| `Cycle` | `int` | Round counter enabling cyclic re-entry without history loss |
| `Status` | `ApprovalStepStatus` | `Pending` / `Approved` / `Rejected` / `Skipped` / `Escalated` |
| `AssignedRoleName`, `AssignedUserId` | `string?` | Snapshot of the resolved authority at generation time |
| `ReviewedByUserId`, `ReviewedAt`, `Comments` | `string?`, `DateTime?`, `string?` | **Attribution captured on the entity** |
| `ParentStepId` / `ParentStep` | `Guid?` / `ApprovalPosition?` | Escalation lineage (self-reference) |
| `Escalations` | `ICollection<ApprovalPosition>` | Child escalation steps |
| `IsEscalation` | `bool` | Distinguishes escalation children from primary steps |
| `RequestNotes` | `string?` | Escalation rationale |

Two construction factories exist — `Create(...)` for campaigns, `CreateForPayout(...)` for payouts — each setting exactly one of the two nullable foreign keys and initializing `Status = Pending`, `Cycle = cycle`, and timestamps.

#### 3.3.1 Transition Semantics

All five transitions are methods with guarded preconditions. `Status` has a private setter; there is no public write path.

**`Approve(reviewerUserId, comments)`**

```csharp
if (Status != ApprovalStepStatus.Pending)
    throw new InvalidOperationException($"Cannot approve step with status {Status}.");
if (string.IsNullOrWhiteSpace(comments))
    throw new ArgumentException("Approval notes/comments are required.", nameof(comments));

Status = ApprovalStepStatus.Approved;
ReviewedByUserId = reviewerUserId;
ReviewedAt = DateTime.UtcNow;
Comments = comments.Trim();
UpdatedAt = DateTime.UtcNow;

if (IsEscalation && ParentStep != null)
    ParentStep.ReturnFromEscalation();
```

Two invariants are enforced at the domain level: **only pending steps may be approved**, and **comments are mandatory** — an approval without a stated rationale is not representable. The final block is the self-healing behaviour of the escalation cycle: approving an escalation child automatically returns its parent step from `Escalated` to `Pending`, so control flows back to the original reviewer without a separate command.

**`Reject(reviewerUserId, reason)`** — same pending guard, same mandatory-reason rule, terminating in `Status = Rejected`.

**`Escalate(requestedByUserId, targetUserId, requestNotes)` — returns `ApprovalPosition`**

```csharp
if (Status != ApprovalStepStatus.Pending)
    throw new InvalidOperationException("Cannot escalate step with status ... Only pending steps can be escalated.");
if (string.IsNullOrWhiteSpace(requestNotes))
    throw new ArgumentException("Escalation request notes are required.", nameof(requestNotes));

Status = ApprovalStepStatus.Escalated;

var child = new ApprovalPosition
{
    Id = Guid.NewGuid(),
    CampaignId = CampaignId,
    PayoutId = PayoutId,
    PositionMasterId = PositionMasterId,
    StepOrder = StepOrder,          // same position in the chain
    Cycle = Cycle,                  // same round
    Status = ApprovalStepStatus.Pending,
    ParentStepId = Id,              // lineage to the escalating step
    IsEscalation = true,
    AssignedUserId = targetUserId,
    AssignedRoleName = "Escalated Reviewer",
    RequestNotes = requestNotes.Trim(),
    CreatedAt = DateTime.UtcNow,
    UpdatedAt = DateTime.UtcNow
};

Escalations.Add(child);
return child;
```

The design decision embedded here is significant: **escalation does not advance the chain, it inserts a parallel authority.** The child inherits the parent's `StepOrder` and `Cycle`, carries `IsEscalation = true`, is bound to a specific `AssignedUserId` rather than a role, and keeps a `ParentStepId` back-reference. The parent moves to `Escalated` — a non-terminal, non-approving state. Rejection of the escalation and the original rejection path remain distinct terminal outcomes.

**`ReturnFromEscalation()`** — restores `Escalated → Pending`; a no-op for any other status. Idempotent by construction.

**`Skip(reason)`** — `Pending → Skipped`, storing the reason in `Comments`. Note the guard asymmetry: `Skip` **silently no-ops** on non-pending steps where the other transitions throw. This is intentional — the reject handler uses `Skip` to cascade over downstream steps (see §4.3.3), and those calls must be tolerant of already-resolved steps.

---

## 4. Application Layer — Workflow Orchestration

The application layer is hand-rolled CQRS: `ICommandHandler<TCommand, TResult>` and `IQueryHandler<TQuery, TResult>` are registered by assembly scanning. No MediatR dependency exists. All handlers are constructor-injected with `IApplicationDbContext`, and use cases that need identity additionally take `ICurrentUserService`.

### 4.1 Use Case Inventory

**Campaign-side** (`Application/Campaigns/ApprovalPositionUseCases.cs`)

| Use case | Type | Purpose |
|---|---|---|
| `GetPositionMastersQuery` | Query | List templates, optionally filtered by `WorkflowType` |
| `CreatePositionMasterCommand` | Command | Create a template; rejects duplicate `Code` |
| `UpdatePositionMasterCommand` | Command | Update a template |
| `DeletePositionMasterCommand` | Command | Delete a template |
| `GetPositionDefinitionsQuery` | Query | List assignments, optionally scoped to a template |
| `CreatePositionDefinitionCommand` | Command | Create a role or user assignment |
| `DeletePositionDefinitionCommand` | Command | Remove an assignment |
| `GetCampaignApprovalStepsQuery` | Query | Read a campaign's full step chain with escalation nesting |
| `ApproveCampaignStepCommand` | Command | Approve a campaign step |
| `RejectCampaignStepCommand` | Command | Reject a campaign step and cascade |
| `EscalateCampaignStepCommand` | Command | Escalate a campaign step to a specific user |

**Payout-side** (`Application/Financial/PayoutApprovalUseCases.cs`)

| Use case | Type | Purpose |
|---|---|---|
| `GetPayoutApprovalStepsQuery` | Query | Read a payout's step chain, lazily generating it if absent |
| `ApprovePayoutStepCommand` | Command | Approve a payout step; triggers settlement on completion |
| `RejectPayoutStepCommand` | Command | Reject a payout step and the payout |
| `EscalatePayoutStepCommand` | Command | Escalate a payout step |

### 4.2 Step Generation — The Progressive Ladder

Both governed object types have a generation helper. `CampaignApprovalWorkflowHelper.GenerateStepsForCampaignAsync` and `PayoutApprovalWorkflowHelper.GenerateStepsForPayoutAsync` implement the same algorithm with type-specific predicates.

**Campaign generation:**

```csharp
var activePositions = await db.PositionMasters
    .Include(p => p.Definitions)
    .Include(p => p.Category)
    .Include(p => p.Beneficiary)
    .Where(p => p.IsActive && p.WorkflowType == PositionWorkflowType.Campaign)
    .OrderBy(p => p.HierarchyOrder)
    .ToListAsync(ct);

if (!activePositions.Any()) return;

var requiredPositions = activePositions
    .Where(p =>
        (p.HierarchyOrder == 1 || targetAmount >= p.MinTargetAmount) &&
        (p.CategoryId == null || (categoryId.HasValue && p.CategoryId == categoryId.Value)) &&
        (p.BeneficiaryId == null || (beneficiaryId.HasValue && p.BeneficiaryId == beneficiaryId.Value)))
    .OrderBy(p => p.HierarchyOrder)
    .ToList();

if (!requiredPositions.Any())
{
    var fallback = activePositions.FirstOrDefault(p => p.CategoryId == null && p.BeneficiaryId == null)
                   ?? activePositions.First();
    requiredPositions.Add(fallback);
}
```

Three rules compose:

1. **Amount ladder.** Tier 1 is unconditional. Tiers ≥ 2 activate only when `targetAmount >= MinTargetAmount`. This means a RM 500 campaign and a RM 500,000 campaign traverse different depths of the same template set — administrative load scales with financial exposure.
2. **Scoping.** A template with `CategoryId == null` applies to every campaign category; a non-null value demands an exact match. Same for `BeneficiaryId`. The scope is therefore *either global or specific*, with no partial-matching semantics.
3. **Non-empty guarantee.** If filtering produces nothing — for example, every active template is beneficiary-scoped and the campaign has no matching beneficiary — the fallback selects the global unscoped template, or failing that the first active template. **A campaign can never enter review with zero steps.**

For each selected position, the helper resolves the *first currently-active definition* and snapshots its assignment onto the step:

```csharp
var activeDef = pos.Definitions.FirstOrDefault(d => d.IsCurrentlyActive(now));
var assignedRole = activeDef?.RoleName
    ?? (activeDef?.AssignmentType == PositionAssignmentType.Role ? activeDef.RoleName : "Reviewer");
var assignedUser = activeDef?.UserId;

var approvalPosition = ApprovalPosition.Create(
    campaignId, pos.Id, stepOrder, assignedRole, assignedUser, cycle);
```

The payout helper is structurally identical with `"FinanceAdmin"` as the role default instead of `"Reviewer"`, and without category matching.

**A snapshot, not a live reference.** `AssignedRoleName` and `AssignedUserId` are copied onto the step instance at generation time. This is a deliberate audit decision: the step records who was *supposed* to review it at the moment the chain was created. But note the consequence — **authorization is not evaluated against the snapshot.** As §4.3.2 shows, the handlers re-load `currentStep.PositionMaster.Definitions` and call `CanUserFulfill` live. The snapshot is documentary; the predicates are authoritative.

### 4.3 Campaign Approval Handlers

#### 4.3.1 `GetCampaignApprovalStepsQuery`

Loads all `ApprovalPosition` rows for the campaign with `PositionMaster` (including `Category` and `Beneficiary`) and `Escalations`, ordered by `Cycle`, then `StepOrder`, then `CreatedAt`. Resolves reviewer and assignee display names in a **single batched lookup** — collected IDs are filtered to parseable GUIDs, deduplicated, and fetched with one `ToDictionaryAsync` call. This avoids N+1 on the read path, which is why the returned DTO set is deliberately flat/denormalized.

The projection splits primary steps from escalation children. Primary steps appear at the top level; their escalations are nested inside the `Escalations` collection of `ApprovalPositionDto`. Each step receives a human-readable `ScopeTag`:

```csharp
string scopeTag = s.PositionMaster.CategoryId.HasValue
    ? $"Category: {s.PositionMaster.Category?.Name ?? "Specialized"}"
    : (s.PositionMaster.BeneficiaryId.HasValue
        ? $"Beneficiary: {s.PositionMaster.Beneficiary?.Name ?? "Institutional"}"
        : (s.PositionMaster.MinTargetAmount > 0 ? $"> RM {s.PositionMaster.MinTargetAmount:N0}" : "Global Tier"));
```

Category scope takes precedence over beneficiary scope, which takes precedence over the amount threshold. The tag is why an operator can see *why* a given step exists without inspecting the template.

#### 4.3.2 `ApproveCampaignStepCommand` — the critical path

The handler executes a fixed sequence of gates. Order matters.

**Gate 1 — Comments required.**

```csharp
if (string.IsNullOrWhiteSpace(cmd.Request.Comments))
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.CommentsRequired", "..."));
```

**Gate 2 — Target and step resolution.** Loads the campaign with `ApprovalPositions → PositionMaster → Definitions` eagerly; returns `DomainErrors.Campaign.NotFound` or `ApprovalStep.NotFound`.

**Gate 3 — Status guard.**

```csharp
if (currentStep.Status != ApprovalStepStatus.Pending)
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.NotPending", $"This step cannot be approved because its status is {currentStep.Status}."));
```

**Identity resolution and default role:**

```csharp
var reviewerUserId = _currentUser.UserId ?? "admin-reviewer";
var userRoles = _currentUser.Roles?.ToList() ?? new List<string> { "SuperAdmin" };
```

**Branch A — Escalation step approval.** The designated-reviewer rule is narrower than the general definition check:

```csharp
var isDesignatedReviewer = userRoles.Contains("SuperAdmin") ||
    string.Equals(currentStep.AssignedUserId, reviewerUserId, StringComparison.OrdinalIgnoreCase);

if (!isDesignatedReviewer)
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.Unauthorized", "You are not the designated reviewer for this escalation request."));
```

An escalation is bound to one named individual. Only that individual — or a `SuperAdmin` — may resolve it. Then:

```csharp
var parentStep = campaign.ApprovalPositions.FirstOrDefault(s => s.Id == currentStep.ParentStepId);
currentStep.Approve(reviewerUserId, cmd.Request.Comments);

if (parentStep != null)
    parentStep.ReturnFromEscalation();
```

The parent's return is invoked here **and** inside `Approve` itself. The duplication is defensive (the domain method guards on `ParentStep != null` via the navigation property, which may not be populated in every load path); `ReturnFromEscalation` is idempotent, so a double invocation is harmless.

**Branch B — Primary step approval.** Three sub-gates:

*Sequential gate.* Approving out of order is refused:

```csharp
var priorUnapproved = campaign.ApprovalPositions.Any(s =>
    s.Cycle == currentStep.Cycle && !s.IsEscalation &&
    s.StepOrder < currentStep.StepOrder && s.Status == ApprovalStepStatus.Pending);

if (priorUnapproved)
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.PrerequisiteRequired", "Earlier approval steps must be approved first."));
```

Note the guard is scoped to `currentStep.Cycle` and excludes escalation children — a step in a *previous* cycle does not block the current one, and an outstanding escalation does not block the parent chain.

*Authority gate.* Live predicate evaluation:

```csharp
var definitions = currentStep.PositionMaster.Definitions.Where(d => d.IsActive).ToList();
var isAuthorized = userRoles.Contains("SuperAdmin") ||
    definitions.Any(d => d.CanUserFulfill(reviewerUserId, userRoles));

if (!isAuthorized)
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.Unauthorized", "You do not have the required role or position authority to approve this step."));
```

Note the pre-filter on `d.IsActive` — but `CanUserFulfill` internally calls `IsCurrentlyActive`, so an expired definition is rejected regardless of the pre-filter. The pre-filter is a minor redundancy, not a control.

*Transition and terminal side effect.*

```csharp
currentStep.Approve(reviewerUserId, cmd.Request.Comments);

var remainingPending = campaign.ApprovalPositions.Any(s =>
    s.Cycle == currentStep.Cycle && !s.IsEscalation && s.Status == ApprovalStepStatus.Pending);

if (!remainingPending)
{
    campaign.Approve(reviewerUserId);          // all tiers cleared → campaign approvable
}
else
{
    if (campaign.Status == CampaignStatus.Draft)
        campaign.SubmitForReview();            // keep state coherent mid-chain
}
```

The completion predicate is scoped identically to the sequential gate: same cycle, primary steps only. The campaign only advances to `Approved` when **every** primary step in the active cycle is terminal. Intermediate approvals promote a `Draft` campaign to `PendingReview` so the campaign state machine stays consistent with the workflow state.

#### 4.3.3 `RejectCampaignStepCommand` — cascade semantics

Rejection is not a local operation. After the same authorization gate, the handler performs a **three-tier cascade**:

```csharp
// 1. Reject the current step
currentStep.Reject(reviewerUserId, cmd.Request.Reason);

// 2. Skip all downstream pending steps in this cycle
foreach (var subsequent in campaign.ApprovalPositions.Where(s =>
    s.Cycle == currentStep.Cycle &&
    s.StepOrder > currentStep.StepOrder &&
    s.Status == ApprovalStepStatus.Pending))
{
    subsequent.Skip("Cancelled due to earlier step rejection.");
}

// 3. Reject the entire campaign
campaign.Reject(reviewerUserId, cmd.Request.Reason);
```

The `Skip` calls are why `Skip` no-ops on non-pending steps rather than throwing — the loop is written defensively against a concurrently-modified collection. The result is a coherent chain end-state: one `Rejected` step, N `Skipped` downstream steps with an explanatory reason, and a `Rejected` campaign. **No step is left dangling in `Pending` after a rejection**, so the chain cannot be resumed by a later approval.

#### 4.3.4 `EscalateCampaignStepCommand`

Stricter input validation than approve/reject — **two** mandatory fields:

```csharp
if (string.IsNullOrWhiteSpace(cmd.Request.TargetUserId))
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.TargetUserRequired", "Please select a user to request review assistance from."));

if (string.IsNullOrWhiteSpace(cmd.Request.Notes))
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.NotesRequired", "Escalation notes ... are required."));
```

The requester must hold authority over the step (same `CanUserFulfill` gate). The target must resolve to a live user:

```csharp
var targetUserGuid = Guid.TryParse(cmd.Request.TargetUserId, out var tg) ? tg : Guid.Empty;
var targetUser = await _db.UserProfiles.FirstOrDefaultAsync(u =>
    (u.Id == targetUserGuid || u.SupabaseUserId == cmd.Request.TargetUserId) && u.DeletedAt == null, ct);

if (targetUser == null)
    return Result.Failure<ApprovalPositionDto>(
        new Error("ApprovalStep.TargetUserNotFound", "..."));
```

Target resolution accepts **either** the internal `UserProfile.Id` GUID **or** the `SupabaseUserId` string — a tolerance that accommodates clients holding one identifier form or the other. The `DeletedAt == null` filter excludes soft-deleted users, consistent with the `UserProfile` global query filter.

Then the domain transition runs and the child is persisted:

```csharp
var escalationChild = currentStep.Escalate(reviewerUserId, targetUser.Id.ToString(), cmd.Request.Notes);

_db.ApprovalPositions.Add(escalationChild);
_db.Entry(escalationChild).State = EntityState.Added;

await _db.SaveChangesAsync(ct);
```

The explicit `EntityState.Added` assignment is deliberate: the child was constructed by the domain method rather than tracked through an aggregate navigation in every code path, so the state is forced rather than inferred.

One behavioural asymmetry worth noting between the two sides: `EscalateCampaignStepCommand` requires a resolvable target user and stores `targetUser.FullName` for display, while `EscalatePayoutStepCommand` accepts `TargetUserId` verbatim with no existence check. The payout escalation path is therefore laxer on input validation than the campaign path — a consistency gap, not a control failure, since the resulting step is still bound to `AssignedUserId` and only that ID or a `SuperAdmin` can resolve it.

### 4.4 Payout Approval Handlers

#### 4.4.1 `GetPayoutApprovalStepsQuery` — lazy generation

The payout read path has a behaviour the campaign path does not: **self-healing step generation**.

```csharp
if (!allPositions.Any())
{
    var payout = await _db.Payouts.FirstOrDefaultAsync(p => p.Id == query.PayoutId, ct);
    if (payout != null)
    {
        await PayoutApprovalWorkflowHelper.GenerateStepsForPayoutAsync(
            _db, query.PayoutId, payout.NetAmount, payout.BeneficiaryId, ct: ct);
        // re-query
    }
}
```

If a payout exists with no approval steps — created before the workflow was configured, for example — the query materializes them on demand using the payout's `NetAmount` and `BeneficiaryId`. The payout amount drives the ladder, so a large disbursement instantly acquires its higher-tier reviews.

#### 4.4.2 `ApprovePayoutStepCommand` — settlement trigger

Structurally parallel to the campaign handler through the same three gates, with one decisive difference at the terminal branch. Its constructor dependencies are heavier:

```csharp
public ApprovePayoutStepCommandHandler(
    IApplicationDbContext db,
    ICurrentUserService currentUser,
    IDoubleEntryLedgerService ledgerService)
```

Because completing the chain is a **financial event**, not merely a state change:

```csharp
var remainingPending = payout.ApprovalPositions.Any(s =>
    s.Cycle == currentStep.Cycle && !s.IsEscalation && s.Status == ApprovalStepStatus.Pending);

if (!remainingPending)
{
    var bankRef = !string.IsNullOrWhiteSpace(cmd.Request.BankReference)
        ? cmd.Request.BankReference
        : $"DUITNOW-TXN-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString()[..6].ToUpperInvariant()}";

    payout.Approve(reviewerUserId);
    payout.Complete(bankRef);

    var ledgerResult = await _ledgerService.PostPayoutDisbursementAsync(payout.Id, bankRef, ct);
    if (ledgerResult.IsFailure)
    {
        return Result.Failure<ApprovalPositionDto>(ledgerResult.Error);
    }
    // ... enqueue post-payout donor notifications
}
```

Three things happen here that make this the domain's most consequential code path:

1. **Automatic bank reference generation.** If the approver supplies no `BankReference`, the handler synthesizes a DuitNow-shaped reference (`DUITNOW-TXN-{yyyyMMdd}-{6 hex}`). Disbursement is never blocked on a missing reference, and the reference is nonetheless traceable and unique per day.
2. **Ledger coupling.** `PostPayoutDisbursementAsync` writes the money-out event into the same chained SHA-256 double-entry ledger that recorded the money-in. The approval chain terminates *directly* into the immutable financial record — disbursement cannot be approved without being accounted for. The failure branch propagates the ledger's error through the `Result` channel, so a ledger rejection surfaces as a step-approval failure rather than a silent partial state.
3. **Donor notification fan-out.** When the payout is campaign-attached, the handler queries distinct confirmed donor emails for that campaign and enqueues post-payout notifications — the charitable loop is closed back to the donors whose contributions funded the disbursement.

#### 4.4.3 `RejectPayoutStepCommand` — two-level termination

The payout rejection cascade is shallower but wider in effect than the campaign equivalent:

```csharp
currentStep.Reject(reviewerUserId, cmd.Request.Reason);
payout.Reject(reviewerUserId, cmd.Request.Reason);
```

The step is rejected **and the entire payout aggregate is rejected** in the same unit of work. Unlike the campaign path, there is no downstream `Skip` loop — the payout status transition is the terminal signal, and the step chain is left as a historical record of where the process stopped. Authorization uses the same live `CanUserFulfill` gate as approval.

#### 4.4.4 `EscalatePayoutStepCommand`

Mirrors its campaign counterpart but with a reduced validation surface: notes are mandatory, the requester must hold step authority, and the target ID is accepted without a `UserProfile` lookup. The domain `Escalate` call remains identical — pending-only guard, mandatory notes, child step with `ParentStepId`, `IsEscalation = true`, and `AssignedUserId` bound to the raw target string.

---

## 5. Contracts Layer

`ApprovalWorkflowContracts.cs` defines positional records — the shared DTO boundary consumed by the API and both front ends. Their design intent is explicit: **flat and denormalized to avoid N+1 on the read path**.

### 5.1 Template and Assignment DTOs

```csharp
public record PositionMasterDto(
    Guid Id, string Name, string Code, string? Description,
    Guid? ParentPositionId, string? ParentPositionName,
    int HierarchyOrder, decimal MinTargetAmount, decimal? MaxTargetAmount,
    bool IsActive, int DefinitionCount = 0,
    Guid? CategoryId, string? CategoryName,
    Guid? BeneficiaryId, string? BeneficiaryName,
    string WorkflowType = "Campaign");
```

`ParentPositionName`, `CategoryName`, `BeneficiaryName`, and `DefinitionCount` are **resolved server-side** rather than requiring the client to cross-reference other collections. `WorkflowType` is projected as a `string` (`.ToString()` on the enum) rather than the enum type, keeping the wire format stable against enum renumbering.

`PositionDefinitionDto` follows the same philosophy, embedding `PositionName`, `PositionCode`, `UserFullName`, `UserEmail`, and a computed `IsCurrentlyActive` alongside the stored `IsActive`:

```csharp
public record PositionDefinitionDto(
    Guid Id, Guid PositionMasterId, string PositionName, string PositionCode,
    string AssignmentType, string? RoleName, string? UserId,
    string? UserFullName, string? UserEmail,
    DateTime ValidFrom, DateTime? ValidTo, bool IsActive, bool IsCurrentlyActive);
```

The stored-vs-effective distinction is legible to the client: an operator can see a definition that is `IsActive = true` but `IsCurrentlyActive = false` because its validity window has lapsed.

### 5.2 Step DTO

```csharp
public record ApprovalPositionDto(
    Guid Id, Guid? CampaignId, Guid PositionMasterId,
    string PositionName, string PositionCode,
    int StepOrder, string Status,
    string? AssignedRoleName, string? AssignedUserId, string? AssignedUserName,
    string? ReviewedByUserId, string? ReviewedByUserName, DateTime? ReviewedAt,
    string? Comments,
    int Cycle = 1, string? ScopeTag = null,
    Guid? ParentStepId = null, bool IsEscalation = false, string? RequestNotes = null,
    List<ApprovalPositionDto>? Escalations = null,
    Guid? PayoutId = null);
```

The `Escalations` self-reference makes the DTO a tree: a primary step contains its escalation children. Ordering is total and deterministic — parents sort by `StepOrder`, children within a parent by `CreatedAt`.

### 5.3 Request Records

| Record | Fields | Guarded by |
|---|---|---|
| `ApproveCampaignStepRequest` | `Comments` | `CommentsRequired` |
| `RejectCampaignStepRequest` | `Reason` | `ReasonRequired` |
| `EscalateCampaignStepRequest` | `TargetUserId`, `Notes` | `TargetUserRequired`, `NotesRequired`, `TargetUserNotFound` |
| `ApprovePayoutStepRequest` | `Comments`, `BankReference?` | `CommentsRequired` |
| `RejectPayoutStepRequest` | `Reason` | `ReasonRequired` |
| `EscalatePayoutStepRequest` | `TargetUserId`, `Notes` | `NotesRequired` |

`ApprovePayoutStepRequest` is the only request record carrying an optional operational field — `BankReference` — reflecting that disbursement approvals have a real-world settlement artefact attached.

---

## 6. API Surface

`ApprovalPositionsController` extends `ApiControllerBase`, is annotated `[ApiController]`, `[Route("api/v1")]`, `[Tags("Campaign Approval Workflow")]`, and injects eleven handler interfaces via constructor — eleven individual `IQueryHandler<,>`/`ICommandHandler<,>` registrations rather than a mediator abstraction. Every action is `async`, builds its query or command record, awaits the handler, and routes the result through a `HandleResult` overload.

`HandleResult` in `ApiControllerBase` is **suffix-driven on `error.Code`**: `NotFound` → 404, `Unauthorized` → 401, `Forbidden` → 403, `Conflict`/`AlreadyExists` → 409, default → 400. This means the error codes chosen in the handlers (`ApprovalStep.NotFound`, `ApprovalStep.Unauthorized`, `ApprovalStep.NotPending`, …) deterministically produce HTTP semantics. The convention is a genuine architectural strength — responses are uniform without per-action mapping code.

| Method | Route | Authorization | Handler |
|---|---|---|---|
| `GET` | `positions/masters?workflowType=` | `[Authorize]` | `GetPositionMastersQuery` |
| `POST` | `positions/masters` | `[Authorize(Roles = "SuperAdmin")]` | `CreatePositionMasterCommand` |
| `PUT` | `positions/masters/{id:guid}` | `[Authorize(Roles = "SuperAdmin")]` | `UpdatePositionMasterCommand` |
| `DELETE` | `positions/masters/{id:guid}` | `[Authorize(Roles = "SuperAdmin")]` | `DeletePositionMasterCommand` |
| `GET` | `positions/definitions?positionMasterId=` | `[Authorize]` | `GetPositionDefinitionsQuery` |
| `POST` | `positions/definitions` | `[Authorize(Roles = "SuperAdmin")]` | `CreatePositionDefinitionCommand` |
| `DELETE` | `positions/definitions/{id:guid}` | `[Authorize(Roles = "SuperAdmin")]` | `DeletePositionDefinitionCommand` |
| `GET` | `campaigns/{campaignId:guid}/approval-steps` | `[Authorize]` | `GetCampaignApprovalStepsQuery` |
| `POST` | `campaigns/{campaignId:guid}/approval-steps/{stepId:guid}/approve` | `[Authorize(Roles = "SuperAdmin,Reviewer,CampaignManager,FinanceAdmin")]` | `ApproveCampaignStepCommand` |
| `POST` | `campaigns/{campaignId:guid}/approval-steps/{stepId:guid}/reject` | `[Authorize(Roles = "SuperAdmin,Reviewer,CampaignManager,FinanceAdmin")]` | `RejectCampaignStepCommand` |
| `POST` | `campaigns/{campaignId:guid}/approval-steps/{stepId:guid}/escalate` | `[Authorize(Roles = "SuperAdmin,Reviewer,CampaignManager,FinanceAdmin")]` | `EscalateCampaignStepCommand` |

**Two-layer authorization.** The controller's `[Authorize]` attributes form the coarse, role-membership layer. The handlers' `CanUserFulfill` gates form the fine, position-specific layer. Both must pass. The attribute layer answers "may this class of user touch approval endpoints at all"; the predicate layer answers "may this specific user resolve *this* step".

**Template mutations are SuperAdmin-only** — creating, updating, and deleting both templates and assignments. A `Reviewer` may act on steps but cannot reconfigure the chain that governs them. This is the administrative separation that keeps the workflow topology itself outside the reach of everyday reviewers.

**The routing split is only partially realized.** Every endpoint on this controller is campaign-oriented; the controller tag is `"Campaign Approval Workflow"`, and the payout step commands live in `Application/Financial/PayoutApprovalUseCases.cs` with endpoints registered elsewhere. The domain is genuinely dual-purpose at the entity and use-case level, but at the HTTP surface the payout side is not surfaced by this controller. This is a documented gap, not a defect in the engine.

---

## 7. Pseudocode Notation

The approval engine has no domain event classes — transitions do not raise `BaseEntityWithEvents` events. The `Result` pattern carries failures instead of exceptions for expected conditions; `InvalidOperationException` and `ArgumentException` are reserved for genuine programming errors (transitioning a non-pending step, omitting mandatory comments at the domain boundary).

State machine, in transition-table form:

```
                     ┌───────────── Approved ──────────── [terminal]
                     │
                     ├───────────── Rejected ──────────── [terminal]
   Pending ──────────┼───────────── Skipped ───────────── [terminal]
                     │
                     └───────────── Escalated ─── Approve(child) ──→ Pending (via ReturnFromEscalation)
                                        │
                                        └── creates child ApprovalPosition
                                            (IsEscalation = true, ParentStepId = parent, Cycle unchanged, StepOrder unchanged)
```

| From | Transition | To | Requires | Side effect |
|---|---|---|---|---|
| `Pending` | `Approve` | `Approved` | comments non-empty | if escalation → parent `ReturnFromEscalation()` |
| `Pending` | `Reject` | `Rejected` | reason non-empty | handler cascades parent aggregate |
| `Pending` | `Escalate` | `Escalated` | notes non-empty | returns new child step |
| `Pending` | `Skip` | `Skipped` | — | none |
| `Escalated` | `ReturnFromEscalation` | `Pending` | — | idempotent |
| any other | `Skip` | unchanged | — | silent no-op |

---

## 8. Workflow Walkthroughs

### 8.1 Campaign Moderation — Happy Path

```
Fundraiser submits campaign (target RM 250,000, category "Qurban")
  │
  ├─ CampaignApprovalWorkflowHelper.GenerateStepsForCampaignAsync(...)
  │    ├─ Tier 1  "Initial Screening"   (HierarchyOrder 1)          → step 1
  │    ├─ Tier 2  "Category Review"     (order 2, Min RM 50,000)    → step 2   ✓ 250k ≥ 50k
  │    └─ Tier 3  "Executive Approval"  (order 3, Min RM 200,000)   → step 3   ✓ 250k ≥ 200k
  │
  ├─ Reviewer (role "Reviewer", definition valid) resolves step 1
  │    ├─ Gate: comments present              ✓
  │    ├─ Gate: status == Pending             ✓
  │    ├─ Gate: no prior pending in cycle     ✓ (step 1 is first)
  │    ├─ Gate: CanUserFulfill("Reviewer")    ✓
  │    └─ currentStep.Approve(...)  →  step 1 = Approved
  │         remainingPending (steps 2,3) → true → campaign stays PendingReview
  │
  ├─ Step 2 approved by category-scoped reviewer → step 2 = Approved
  │
  └─ Step 3 approved
       ├─ remainingPending → false
       └─ campaign.Approve(reviewerUserId)  →  CampaignStatus.Approved
```

### 8.2 Escalation Round Trip

```
Step 2 "Category Review" — reviewer lacks Qurban-specific authority
  │
  ├─ EscalateCampaignStepCommand(campaignId, step2, { TargetUserId, Notes })
  │    ├─ Gate: TargetUserId non-empty        ✓
  │    ├─ Gate: Notes non-empty               ✓
  │    ├─ Gate: CanUserFulfill(requester)     ✓
  │    ├─ Target resolves in UserProfiles     ✓
  │    └─ currentStep.Escalate(requester, targetUser.Id, notes)
  │         ├─ step2.Status = Escalated
  │         └─ child = ApprovalPosition { IsEscalation=true,
  │                                       ParentStepId = step2.Id,
  │                                       StepOrder = 2, Cycle = 1,
  │                                       AssignedUserId = targetUser.Id,
  │                                       AssignedRoleName = "Escalated Reviewer" }
  │
  ├─ Escalated reviewer approves child
  │    ├─ Gate: user == step.AssignedUserId || SuperAdmin   ✓
  │    ├─ child.Approve(...)  →  child = Approved
  │    │    └─ parent.ReturnFromEscalation()  →  step2 = Pending
  │    └─ handler re-invokes parent.ReturnFromEscalation()  (idempotent)
  │
  └─ Step 2 now Pending again — original reviewer proceeds normally
```

The escalation did **not** consume a cycle. `Cycle` stays at 1, `StepOrder` stays at 2. The chain position is preserved; only the authority expanded and then contracted. This is why the workflow is described as cyclic — control can route outward and return without the chain losing its place.

### 8.3 Payout Disbursement — Terminal Path

```
Finance officer creates payout (NetAmount RM 80,000, BeneficiaryId = B)
  │
  ├─ PayoutApprovalWorkflowHelper.GenerateStepsForPayoutAsync(
  │      payoutId, 80000, beneficiaryId: B)
  │    ├─ Tier 1 "Finance Verification" (order 1)                    → step 1
  │    ├─ Tier 2 "Senior Finance"       (order 2, Min RM 50,000)     → step 2   ✓ 80k ≥ 50k
  │    └─ Tier 3 "Beneficiary-Specific" (beneficiary = B)            → step 3   ✓ match
  │
  ├─ Step 1 approved → step 1 = Approved
  ├─ Step 2 approved → step 2 = Approved
  │
  └─ Step 3 approved  (ApprovePayoutStepRequest { Comments, BankReference })
       ├─ remainingPending → false
       ├─ bankRef = BankReference ?? "DUITNOW-TXN-20261110-A3F9C1"
       ├─ payout.Approve(reviewerUserId)
       ├─ payout.Complete(bankRef)
       ├─ ledgerService.PostPayoutDisbursementAsync(payout.Id, bankRef, ct)
       │    └─ balanced entries appended to chained SHA-256 ledger
       ├─ donor emails for the campaign queried (distinct, confirmed only)
       └─ post-payout notifications enqueued
```

---

## 9. Structural Analysis

### 9.1 Strengths

**Single transition authority.** All three handlers on each side share one domain state machine and one authorization predicate. There is no second code path that can mutate `Status`. The `PayoutApprovalWorkflowHelper` and `CampaignApprovalWorkflowHelper` sit behind their respective command handlers, so approve/reject/escalate cannot diverge in their legality rules.

**Audit trail as entity state.** `ReviewedByUserId`, `ReviewedAt`, and `Comments` are columns on `ApprovalPosition`. The mechanical `AuditLog` rows generated by `ApplicationDbContext.SaveChangesAsync` — capturing actor, email, IP, user agent, and before/after JSON per modified entity — provide a second, independent record. Two records, one event, no reliance on a third party remembering to log.

**Escalation preserves history.** Escalation creates a new row rather than mutating the original. The parent remains as `Escalated` with its `ParentStepId` lineage resolvable to the child that superseded it. Nothing is overwritten.

**Mandatory rationale.** Approve requires comments; reject requires a reason; escalate requires notes. A decision without a stated rationale is not representable in the domain — enforced twice (handler guard returning a `Result` failure, domain guard throwing).

**Non-empty chain guarantee.** The fallback in both generation helpers means a campaign or payout can never enter governance with zero steps, even under a degenerate template configuration.

**Fail-closed authorization predicates.** `CanUserFulfill` returns `false` for unrecognized assignment shapes; `IsCurrentlyActive` returns `false` for lapsed windows; `IsApplicable` returns `false` for inactive or mismatched templates. No permissive default branch exists anywhere in the predicate set.

**Inactive-template exclusion at generation.** Both helpers filter `p.IsActive` at query time, and the predicates re-check `IsActive`. Deactivating a template stops it from entering *new* chains without disturbing existing ones.

### 9.2 Structural Risks

| ID | Observation | Severity | Nature |
|---|---|---|---|
| **A-01** | Fallback identity values (`?? "admin-reviewer"`, `?? "admin-finance"`) and the `?? new List<string> { "SuperAdmin" }` role default mean that **a request reaching a handler with no resolvable identity is treated as a SuperAdmin reviewer.** The handlers are therefore only as safe as the authentication middleware in front of them. | **HIGH** | Design coupling — not an engine defect, but the engine amplifies any identity failure upstream |
| **A-02** | `userRoles.Contains("SuperAdmin")` short-circuits the `CanUserFulfill` predicate entirely on an **ordinal case-sensitive** comparison. `CanUserFulfill` itself compares roles `OrdinalIgnoreCase`; the SuperAdmin bypass does not. | **MEDIUM** | Inconsistency — a role named `superadmin` passes the predicate but not the bypass |
| **A-03** | The snapshot fields (`AssignedRoleName`, `AssignedUserId`) are not used for authorization; live definitions are. If a template's definitions are reconfigured mid-chain, in-flight steps become resolvable by a different authority than the one recorded on them. | **MEDIUM** | Behavioural — intentional (live authority) but the snapshot can mislead an auditor |
| **A-04** | `EscalatePayoutStepCommand` does not validate that `TargetUserId` resolves to a real user, unlike the campaign equivalent. | **LOW** | Consistency gap — the step remains bound to the raw ID and remains resolvable only by that ID or SuperAdmin |
| **A-05** | `Payout.HasPendingApprovalSteps()` and `GetCurrentCycle()` exist on the aggregate but the command handlers compute pending/completion state with their own LINQ predicates over `payout.ApprovalPositions` rather than calling them. Two implementations of the same rule. | **MEDIUM** | Duplication risk — the aggregate methods and the handler predicates could drift |
| **A-06** | Payout approval handlers load `PositionMaster.Definitions` eagerly through a three-level `ThenInclude` on every approve. Correct, but it means each decision reads the entire definition set of the position. | **LOW** | Performance — acceptable at expected chain depths |
| **A-07** | The payout approval endpoints are not exposed by `ApprovalPositionsController`; the campaign-tagged controller registers no payout routes. | **LOW** | Documentation/routing gap — tracking which controller serves payout steps requires reading the application layer |
| **A-08** | No `MaxTargetAmount` evaluation exists anywhere. The field is on `PositionMaster`, projected into `PositionMasterDto`, and settable through the API — but neither `IsApplicable`, `IsApplicableForPayout`, nor either generation helper reads it. | **MEDIUM** | Declared-but-unimplemented capability — a template configured with a maximum bound does not actually bound itself |
| **A-09** | `Cycle` increments are never performed by any observed code path. `Escalate` preserves the parent's cycle; `Create`/`CreateForPayout` accept a `cycle` parameter defaulting to `1`. The cyclic-re-entry capability the field models is not currently driven. | **MEDIUM** | Capability gap — the field is correctly plumbed but no re-entry path exercises it |
| **A-10** | `MaxTargetAmount`, `ParentPosition`/`ChildPositions`, and cycle incrementing are all present in the model but not exercised. The template hierarchy is stored and displayed but step ordering derives from `HierarchyOrder` alone, not from parent-child traversal. | **LOW** | Model-vs-behaviour drift |

### 9.3 Non-Issues — Explicitly Verified

The following were checked because they would be defects if present, and they are not:

- **`Status` has a private setter.** No external mutation path exists.
- **`CanUserFulfill` has no permissive default.** A `Role` assignment with a null/whitespace `RoleName` returns `false`.
- **Rejection leaves no dangling pending steps.** The cascade explicitly skips downstream pending steps.
- **Out-of-order approval is refused.** The sequential gate is present and scoped correctly to the active cycle, excluding escalation children.
- **Template mutation is behind `[Authorize(Roles = "SuperAdmin")]`** on all five mutating endpoints.
- **Approval without comments is unreachable.** Guarded at the handler (returns `Result.Failure`) and again in the domain method (throws).

---

## 10. Verification Status

### 10.1 Verbatim Source Read

`Domain/Entities/Campaigns/PositionMaster.cs` (full) · `PositionDefinition.cs` (full) · `ApprovalPosition.cs` (full) · `Domain/Enums/DomainEnums.cs` (full) · `Application/Campaigns/ApprovalPositionUseCases.cs` (step-workflow and generation sections, incl. all four campaign handlers) · `Application/Financial/PayoutApprovalUseCases.cs` (helper, query, approve, reject, escalate handlers) · `Contracts/Campaigns/ApprovalWorkflowContracts.cs` (full) · `Api/Controllers/Campaigns/ApprovalPositionsController.cs` (full) · `Domain/Entities/Financial/FinancialEntities.cs` (Payout aggregate head) · `Infrastructure/Persistence/ApplicationDbContext.cs` (full) · `Infrastructure/Persistence/Configurations/EntityConfigurations.cs` (head) · `deploy/database/01_esalam_supabase_schema.sql` (head).

### 10.2 Not Verified — Access-Limited

| Item | Status | What would settle it |
|---|---|---|
| EF `ApprovalPositionConfiguration` mapping (table name, FK delete behaviour, indexes) | **UNVERIFIABLE** | Read the `ApprovalPositionConfiguration` block in `EntityConfigurations.cs` (693 lines; only the first ~130 read) |
| Whether `ApprovalPositions`, `PositionMasters`, `PositionDefinitions` DDL exists in the Supabase schema | **UNVERIFIABLE** | Grep `01_esalam_supabase_schema.sql` (528 lines; only the first ~300 read) for the three table names |
| Whether a uniqueness constraint exists on `(CampaignId, StepOrder, Cycle)` or `(PayoutId, StepOrder, Cycle)` | **UNVERIFIABLE** | Read the EF configuration or the DDL; absence would permit duplicate steps under concurrent generation |
| AdminWeb `[Authorize]` coverage on approval pages | **UNVERIFIABLE** | Read `PayoutsManagement.razor`, `CampaignsManagement.razor`, `ApprovalDecisionDialog.razor` |
| Whether `ApprovalDecisionDialog` / `ApprovalEscalateDialog` submit correct step IDs and refresh state | **UNVERIFIABLE** | Read the dialog components (6.7 KB and 8.2 KB) |
| Which controller exposes the payout approval endpoints | **UNVERIFIABLE** | Read the payout section of `FinancialController.cs` |
| Whether `AdminApiClient` header-based identity (see architecture drift D-01/D-02) can supply an arbitrary `X-User-Roles` that satisfies the SuperAdmin bypass | **PLAUSIBLE by architectural evidence, not confirmed in the approval path** | Requires tracing the middleware-to-handler identity flow end to end |

The last item is the one with real severity weight. Risk **A-01** states the engine trusts whatever identity the handlers receive. If the upstream identity boundary can be forged — as the architecture assessment found for the admin header-trust middleware — then the SuperAdmin short-circuit in every approval handler becomes directly reachable. The approval engine itself is not the defect; it is the amplifier.

---

## 11. Recommendations

Ordered by impact × implementation ease.

| # | Priority | Action | Addresses | Effort |
|---|---|---|---|---|
| 1 | **HIGH** | Remove the permissive identity fallbacks — replace `?? "admin-reviewer"` / `?? "admin-finance"` and `?? new List<string> { "SuperAdmin" }` with an explicit `Result.Failure` when `ICurrentUserService.UserId` is null. A step decision must never be attributable to a synthesized actor. | A-01 | < 1 h |
| 2 | **HIGH** | Confirm the upstream identity boundary before this engine's SuperAdmin short-circuit matters. If `X-User-*` header trust remains in place, the approval chain's segregation of duties is bypassable at the front door regardless of engine correctness. | A-01 | depends on auth remediation |
| 3 | **MEDIUM** | Make the SuperAdmin bypass use `OrdinalIgnoreCase`, matching `CanUserFulfill`, or derive it from a single shared `IsSuperAdmin(userRoles)` helper. | A-02 | 15 min |
| 4 | **MEDIUM** | Either implement `MaxTargetAmount` in the applicability predicates and generation helpers, or remove it from the entity and contract. A displayed-but-ignored bound is worse than no bound. | A-08 | 1 h (implement) / 15 min (remove) |
| 5 | **MEDIUM** | Route the handlers' pending/completion predicates through `Payout.HasPendingApprovalSteps()` and `GetCurrentCycle()` so the aggregate remains the single authority on chain state. | A-05 | 2 h |
| 6 | **MEDIUM** | Add a uniqueness constraint on `(CampaignId, StepOrder, Cycle)` and `(PayoutId, StepOrder, Cycle)`; generation is not idempotent, and `GetPayoutApprovalStepsQuery` lazily generates on read, so concurrent reads on an ungenerated payout can double-generate. | Verification gap | 1 h |
| 7 | **MEDIUM** | Either exercise `Cycle` incrementing (a genuine re-entry path on rejection-and-resubmit) or document it as reserved. The field is correctly plumbed but no code path advances it, and its presence in the query ordering implies behaviour that does not occur. | A-09 | 4 h (implement) / 15 min (document) |
| 8 | **LOW** | Add target-user existence validation to `EscalatePayoutStepCommand` to match `EscalateCampaignStepCommand`. | A-04 | 30 min |
| 9 | **LOW** | Expose the payout approval endpoints under a clearly named controller, or rename `ApprovalPositionsController` and retag it so its scope is unambiguous. | A-07 | 1 h |
| 10 | **LOW** | Document that `AssignedRoleName`/`AssignedUserId` are documentary snapshots and that live `PositionDefinition` predicates are authoritative — or switch authorization to the snapshot and make reconfiguration require chain regeneration. | A-03 | documentation / design decision |

---

## 12. Summary

The Approval Workflow Domain is the best-evidenced reusable component in ESalam. It implements template-driven, multi-step, escalation-capable governance over two object types through one state machine, with a domain-encapsulated transition authority, a complete decision audit trail stored on the entity, and fail-closed authorization predicates.

Its correctness properties are real and verifiable: pending-only transitions, mandatory rationales, sequential ordering gates, live role/position authority checks, non-empty chain generation, and a rejection cascade that leaves no resumable pending steps. Its terminal integration with the double-entry ledger means a payout cannot be approved without being accounted for.

Its risks are equally specific and mostly *environmental* rather than *internal*. The engine trusts the identity it receives, and its SuperAdmin short-circuit means an upstream identity failure becomes an unrestricted approval authority. Three capabilities are present in the model but not exercised — `MaxTargetAmount`, `Cycle` advancement, and the parent-child template hierarchy — and their presence in the API surface and query ordering implies behaviour that does not currently exist. Finally, the payout and campaign paths, though sharing the domain state machine exactly, diverge in input strictness and in routing exposure.

The engine is sound. The gaps are at its edges — inherited identity, undeclared boundaries, and capabilities that are planted but not wired.

---

*Source-verified against the ESalam repository. Findings marked UNVERIFIABLE are stated as gaps rather than assumed correct. Drift items referencing middleware behaviour are cross-referenced to the system architecture assessment, not re-derived here.*