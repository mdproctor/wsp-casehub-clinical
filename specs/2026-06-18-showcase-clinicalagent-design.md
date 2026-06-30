# 3-Site Showcase + ClinicalAgent Comparison — Design Spec

**Issue:** casehubio/clinical#10
**Branch:** issue-10-showcase-clinicalagent
**Date:** 2026-06-18 (rev 7)

---

## Overview

Delivers the §7.4 showcase scenario from `tutorial-strategy.md`: a 3-site oncology trial
exercising all completed layers in a single narrative integration test, plus a
ClinicalAgent comparison document.

Three deliverables:

1. **Two new domain features** — eligibility screening (Site A) and protocol amendment (Site C)
2. **`ThreeSiteShowcaseTest`** — single narrative `@QuarkusTest` orchestrating all 3 sites
3. **`docs/comparison/clinicalagent.md`** — GCP/FDA gap table with class/layer references

Also: rename `ShowcaseScenarioTest` → `ClinicalLayerComplianceTest`.

---

## Layer 9 — Showcase

Layers 1–8 each integrate a new CaseHub foundation module. Layer 9 is different: it adds
new domain application features (eligibility screening, protocol amendment) that exercise
existing foundation modules (Layers 4+5) without a new foundation dependency. The value
demonstrated is not a new module integration but the structural completeness of the
compliance story.

ARC42STORIES.MD will be updated to document Layer 9 at implementation close.

---

## 1. Eligibility Screening (Site A)

### Domain model additions

**`EligibilityScreeningResult` enum** (`api/model/`):
```
CRITERIA_MET, MARGINAL, EXCLUDED
```

**`EligibilityScreeningCaseStatus` enum** (`api/model/`):
```
NONE, REQUESTED, COMPLETED, FAILED
```
Same shape as `SusarOversightStatus` / `AeEscalationStatus`. Used as the idempotency
guard on the engine case lifecycle, independent of the domain `enrollmentStatus`.
COMPLETED is set by the future IRB completion listener (out of scope here) when the
engine case goal fires.

**`EnrollmentStatus`** — no new values. The existing enum already contains:
- `SCREENING` — used when IRB consultation is pending after marginal result
- `ELIGIBLE` — used when all criteria are met
- `INELIGIBLE` — used when patient is excluded

**`PatientEnrollment` new fields** (default datasource, V122):
- `screening_result VARCHAR(50)` — nullable until screened
- `screening_completed_at TIMESTAMP WITH TIME ZONE` — nullable
- `eligibility_engine_case_id UUID` — nullable; set in Phase 3
- `eligibility_screening_case_status VARCHAR(50) NOT NULL DEFAULT 'NONE'` — engine
  case lifecycle status; idempotency guard for `EligibilityScreeningCaseService`

### CDI event

**`EligibilityScreeningEvent` record** (`api/`):
```java
public record EligibilityScreeningEvent(
    UUID enrollmentId,
    String tenantId,
    EligibilityScreeningResult screeningResult,
    List<CriterionResult> criteriaResults
) {}
```

**`CriterionResult` record** (`api/model/`): `String id`, `boolean met`, `boolean marginal`.

### REST endpoint

`POST /trials/{trialId}/sites/{siteId}/patients/{enrollmentId}/screen`
on `PatientResource`.

Request body:
```json
{
  "criteria": [
    { "id": "criterion-7", "met": false, "marginal": true },
    { "id": "criterion-11", "met": false, "marginal": true }
  ]
}
```

Returns 200 with updated enrollment (`screeningResult`, `enrollmentStatus`).

### `EligibilityScreeningService`

`@ApplicationScoped`. Called synchronously from `PatientResource`.

Result logic:
- Any `marginal=true` → `MARGINAL`
- Any `met=false` (non-marginal) → `EXCLUDED`
- All `met=true` → `CRITERIA_MET`

One `@Transactional` call:
1. Call `eligibilityScreeningLedgerWriter.writeScreeningEntry()` (qhorus datasource)
2. Update `enrollment.screeningResult`, `screeningCompletedAt`, `enrollmentStatus`
3. If `MARGINAL` → fire `EligibilityScreeningEvent` via `Event.fireAsync()` (CDI delivers
   after TX commit; `EligibilityScreeningCaseService` observes it)

XA required: default datasource (entity) + qhorus datasource (ledger) in same TX.

### `EligibilityScreeningLedgerWriter`

`@ApplicationScoped`, per ADR-0002. Owns `sequenceNumber` computation via
`findLatestBySubjectId`. Single method now (`writeScreeningEntry()`); `writeResolutionEntry()`
added when IRB completion listener lands (out of scope here).

**`EligibilityScreeningLedgerEntry`** (qhorus datasource, V2024):
Fields: `enrollmentId UUID`, `screeningResult VARCHAR(50)`, `criteriaCount INT`,
`marginalCount INT`.

`domainContentBytes()`: `String.join("|", enrollmentId, screeningResult,
String.valueOf(criteriaCount), String.valueOf(marginalCount)).getBytes(UTF_8)`.

### Engine case

**`eligibility-screening.yaml`**:
```yaml
dsl: "0.1"
version: "1.0.0"
name: eligibility-screening
namespace: clinical

spec:
  goals:
    - name: irb-consultation-complete
      kind: success
      condition: ".irbConsultation != null"

  completion:
    success:
      allOf: [irb-consultation-complete]

  bindings:
    - name: irb-consultation
      on:
        contextChange:
          filter: ".enrollmentId != null and .screeningResult == \"MARGINAL\" and .irbConsultation == null"
      humanTask:
        title: "IRB eligibility consultation — marginal criteria"
        expiresIn: PT72H
        candidateGroups: [irb-committee]
        inputMapping: "{ enrollmentId: .enrollmentId, criteriaResults: .criteriaResults }"
        outputMapping: "{ irbConsultation: . }"
```

Positive guard (`.enrollmentId != null`) prevents firing on empty context before initial
context is applied (pattern from `susar-oversight.yaml`).

**`EligibilityScreeningCaseHub`** extends `YamlCaseHub("clinical/eligibility-screening.yaml")`.
`@ApplicationScoped`. No `getDefinition()` override.

**`EligibilityScreeningCaseService`** — `@ApplicationScoped`, observes
`@ObservesAsync EligibilityScreeningEvent`. Four-phase pattern (reference:
`SusarOversightCaseService`):

- **Phase 1** `@Transactional prepareAndMark(event)`: load `PatientEnrollment` by
  `PatientEnrollment.findById(enrollmentId)` (base Panache — not `findByIdForTenant`;
  `@ObservesAsync` runs off-request with no `@RequestScoped CurrentPrincipal`);
  idempotency guard — if `enrollment.eligibilityScreeningCaseStatus != NONE` return null;
  set `enrollment.eligibilityScreeningCaseStatus = REQUESTED`; build and return initial
  context map
- **Phase 2**: `eligibilityScreeningCaseHub.startCase(ctx).toCompletableFuture().join()`
  outside any TX boundary
- **Phase 3** `@Transactional persistCaseId(enrollmentId, caseId)`: set
  `enrollment.eligibilityEngineCaseId = caseId`
- **Phase 4** `@Transactional markFailed(enrollmentId)` — called in catch block wrapping
  Phases 2–3: sets `enrollment.eligibilityScreeningCaseStatus = FAILED`

Initial case context (all values serialized as strings or primitives — no domain objects):
```json
{
  "enrollmentId": "<uuid-string>",
  "tenantId": "<string>",
  "screeningResult": "MARGINAL",
  "criteriaResults": [
    { "id": "criterion-7", "met": false, "marginal": true }
  ]
}
```

Each `CriterionResult` is serialized to `Map.of("id", r.id(), "met", r.met(), "marginal",
r.marginal())` before insertion. The engine context is `Map<String, Object>`; the JQ
evaluator requires JSON-compatible types. Do not insert `CriterionResult` records directly.

---

## 2. Protocol Amendment (Site C)

### Domain model

**`ProtocolAmendmentStatus` enum** (`api/model/`):
```
PROPOSED     — created, awaiting advisor
SUPERVISED   — terminal state: advisor recommended DSMB review (pending #86)
APPROVED     — advisor returned PROCEED
HALTED       — advisor returned HALT
```

Note: SUPERVISED is a *terminal business state* for the DSMB referral path, not an
idempotency guard. The Phase 1 idempotency guard is `AmendmentCaseStatus.REQUESTED`.

**`AmendmentCaseStatus` enum** (`api/model/`):
```
NONE, REQUESTED, COMPLETED, FAILED
```
Same shape as `AeEscalationStatus` / `SusarOversightStatus`. Separate from
`ProtocolAmendmentStatus` to keep infrastructure lifecycle concerns off the business state.
COMPLETED is set by `ProtocolAmendmentListener` after successfully routing the recommendation.

**`AmendmentRecommendation` enum** (`api/spi/`):
```
PROCEED, REFER_TO_DSMB, HALT
```

**`ProtocolAmendment` entity** (default datasource, V123):
```sql
CREATE TABLE protocol_amendment (
  id UUID PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  trial_id UUID NOT NULL,
  proposed_change TEXT NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'PROPOSED',
  amendment_case_status VARCHAR(50) NOT NULL DEFAULT 'NONE',
  supervisor_recommendation VARCHAR(50),
  engine_case_id UUID,
  proposed_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

No `findByEngineCaseId` static finder needed — `ProtocolAmendmentListener` discriminates
via the case context (see §2.6 below).

### CDI event

**`ProtocolAmendmentProposedEvent` record** (`api/`):
```java
public record ProtocolAmendmentProposedEvent(
    UUID amendmentId,
    UUID trialId,
    String proposedChange,
    String tenantId
) {}
```

### REST endpoints

`ProtocolAmendmentResource` exposes two endpoints:

**`POST /trials/{trialId}/amendments`** — delegates to `ProtocolAmendmentService.propose()`.
Request: `{ "proposedChange": "Dose escalation amendment v2" }`.
Returns 201 with created amendment (`status = PROPOSED`).

**`GET /trials/{trialId}/amendments/{amendmentId}`** — ownership check: load amendment,
verify `amendment.trialId == trialId`. Returns:
```json
{
  "id": "...",
  "trialId": "...",
  "proposedChange": "...",
  "status": "APPROVED",
  "supervisorRecommendation": "PROCEED",
  "proposedAt": "..."
}
```

### `ProtocolAmendmentService`

`@ApplicationScoped`. `@Transactional propose()`:
1. Create and persist `ProtocolAmendment` (`status = PROPOSED`, `amendmentCaseStatus = NONE`)
2. Call `protocolAmendmentLedgerWriter.writeProposalEntry(amendment)`
3. Fire `ProtocolAmendmentProposedEvent` via `Event.fireAsync()`

XA required: default datasource (entity) + qhorus datasource (ledger) in same TX.

### LLM supervisor slot

**`ProtocolAmendmentContext` record** (`api/spi/`):
```java
public record ProtocolAmendmentContext(
    UUID amendmentId,
    UUID trialId,
    String proposedChange,
    Map<String, Object> trialBlackboardSnapshot
) {}
```

**`ProtocolAmendmentAdvisor` SPI** (`api/spi/`):
```java
public interface ProtocolAmendmentAdvisor {
    AmendmentRecommendation advise(ProtocolAmendmentContext context);
}
```

**`DefaultProtocolAmendmentAdvisor`** (`runtime/service/`, `@DefaultBean @ApplicationScoped`):
Always returns `AmendmentRecommendation.PROCEED`. Javadoc: "Stub — replace with
LlmPlanningStrategy integration when casehubio/engine#101 lands (tracked: clinical#86)."

### Engine case

**`protocol-amendment.yaml`**:
```yaml
dsl: "0.1"
version: "1.0.0"
name: protocol-amendment
namespace: clinical

spec:
  capabilities:
    - name: protocol-amendment-advisor
      inputSchema: "{ amendmentId: .amendmentId, trialId: .trialId, proposedChange: .proposedChange }"

  goals:
    - name: amendment-resolved
      kind: success
      condition: ".advisorRecommendation != null"

  completion:
    success:
      allOf: [amendment-resolved]

  bindings:
    - name: advisor-consultation
      on:
        contextChange:
          filter: ".amendmentId != null and .advisorRecommendation == null"
      capability: protocol-amendment-advisor
```

Advisory-only case: the advisor's recommendation is the decision. No downstream humanTask.
Positive guard (`.amendmentId != null`) prevents empty-context firing.

**`ProtocolAmendmentCaseHub`** extends `YamlCaseHub("clinical/protocol-amendment.yaml")`.
Overrides `getDefinition()` to register a Java-function worker for `protocol-amendment-advisor`.
Reference implementation: `ClinicalSusarOversightCaseHub` (same pattern, same builder shape).

```java
@ApplicationScoped
public class ProtocolAmendmentCaseHub extends YamlCaseHub {

    @Inject ProtocolAmendmentAdvisor advisor;
    private volatile CaseDefinition augmentedDefinition;  // double-checked locking for thread safety

    public ProtocolAmendmentCaseHub() { super("clinical/protocol-amendment.yaml"); }

    @Override
    public CaseDefinition getDefinition() {
        if (augmentedDefinition == null) {
            synchronized (this) {
                if (augmentedDefinition == null) {
                    CaseDefinition def = super.getDefinition();
                    def.getWorkers().add(Worker.builder()
                        .name("protocol-amendment-advisor-worker")
                        .capabilities(List.of(Capability.builder()
                            .name("protocol-amendment-advisor")
                            .inputSchema("{ amendmentId: .amendmentId, trialId: .trialId, proposedChange: .proposedChange }")
                            .outputSchema(".")   // merges { advisorRecommendation } back into case context
                            .build()))
                        .function(ctx -> {
                            ProtocolAmendmentContext pac = new ProtocolAmendmentContext(
                                UUID.fromString((String) ctx.get("amendmentId")),
                                UUID.fromString((String) ctx.get("trialId")),
                                (String) ctx.get("proposedChange"),
                                Map.of()  // blackboard snapshot added when #86 lands
                            );
                            // WorkerFunction.Sync expects Function<Map<String,Object>, WorkerResult>
                            // WorkerResult.of() is the factory for a success result with no planned action
                            return WorkerResult.of(Map.of("advisorRecommendation", advisor.advise(pac).name()));
                        })
                        .build());
                    augmentedDefinition = def;
                }
            }
        }
        return augmentedDefinition;
    }
}
```

`.outputSchema(".")` is required — it merges the worker's return map into the case context.
Without it, `.advisorRecommendation` is never set and the goal `.advisorRecommendation != null`
never fires, hanging the case permanently.

**`ProtocolAmendmentCaseService`** — `@ApplicationScoped`, observes
`@ObservesAsync ProtocolAmendmentProposedEvent`. Four-phase pattern:

- **Phase 1** `@Transactional prepareAndMark(event)`: load `ProtocolAmendment` by
  `ProtocolAmendment.findById(amendmentId)` (base Panache — not tenant-scoped; same
  reasoning as `AeEscalationCaseService.prepareAndMarkRequested()` line 60);
  idempotency guard — if `amendment.amendmentCaseStatus != NONE` return null;
  set `amendment.amendmentCaseStatus = REQUESTED`; build and return initial context map
- **Phase 2**: `protocolAmendmentCaseHub.startCase(ctx).toCompletableFuture().join()`
  outside any TX boundary
- **Phase 3** `@Transactional persistCaseId(amendmentId, caseId)`: set
  `amendment.engineCaseId = caseId`
- **Phase 4** `@Transactional markFailed(amendmentId)` — called in catch block wrapping
  Phases 2–3: sets `amendment.amendmentCaseStatus = FAILED`

Initial case context (all strings/primitives — no domain objects):
```json
{
  "amendmentId": "<uuid-string>",
  "trialId": "<uuid-string>",
  "proposedChange": "<string>",
  "tenantId": "<string>"
}
```

`amendmentId` must be in the initial context — `ProtocolAmendmentListener` depends on it
for discrimination.

### `ProtocolAmendmentListener`

`@ApplicationScoped`. `@Transactional`. Observes `@ObservesAsync CaseLifecycleEvent`.

Accepts `"GoalReached"` and `"CaseCompleted"` event types (engine#393: `GoalReached` fires
first and reliably; accept both with idempotency guard).

Context access follows the `AeEscalationListener` pattern exactly:

```java
if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;

var instance = caseInstanceRepository
    .findByUuid(event.caseId(), event.tenancyId())
    .await().atMost(Duration.ofSeconds(5));
if (instance == null) return;

// Discriminate: not a protocol amendment case if amendmentId absent
Object amendmentIdObj = instance.getCaseContext().getPath("amendmentId");
if (amendmentIdObj == null) return;

UUID amendmentId = UUID.fromString(amendmentIdObj.toString());
ProtocolAmendment amendment = ProtocolAmendment.findById(amendmentId);
if (amendment == null) return;

// Idempotency guard: GoalReached fires multiple times per case (engine#393).
// supervisorRecommendation is null until the listener first sets it; non-null on
// any re-delivery regardless of branch taken (including REFER_TO_DSMB which keeps
// status=SUPERVISED — a status-based guard would re-enter on re-delivery).
if (amendment.supervisorRecommendation != null) return;

String rec = (String) instance.getCaseContext().getPath("advisorRecommendation");
amendment.supervisorRecommendation = rec;
amendment.status = switch (rec) {
    case "PROCEED"       -> ProtocolAmendmentStatus.APPROVED;
    case "HALT"          -> ProtocolAmendmentStatus.HALTED;
    case "REFER_TO_DSMB" -> ProtocolAmendmentStatus.SUPERVISED;
    default -> throw new IllegalStateException("Unknown recommendation: " + rec);
};
amendment.amendmentCaseStatus = AmendmentCaseStatus.COMPLETED;
protocolAmendmentLedgerWriter.writeResolutionEntry(amendment);
```

**`@Transactional` and XA:** the listener updates `amendment.status` (default datasource)
and calls `writeResolutionEntry()` (qhorus datasource) in the same method. XA is required
on both datasources.

**Observer fallback (PP-20260530-49856c):** explicitly not applicable here. The protocol
addresses the pattern where a `REQUIRES_NEW` sub-transaction commits a guard before the
ledger write, creating a potential FDA gap if a downstream `fireAsync()` then throws.
`ProtocolAmendmentListener` has no `REQUIRES_NEW` split and no `fireAsync()` after the
ledger write. Status update and ledger write are in the same XA transaction — both commit
or neither does. The double-write scenario cannot occur.

### `ProtocolAmendmentLedgerWriter`

`@ApplicationScoped`, per ADR-0002. Owns `sequenceNumber` computation via
`findLatestBySubjectId(amendment.id, tenancyId)` — both `writeProposalEntry` and
`writeResolutionEntry` write to the same subject chain; sequence ownership must be
centralised here, not spread across callers. Two methods:
- `writeProposalEntry(ProtocolAmendment)` — called by `ProtocolAmendmentService`
- `writeResolutionEntry(ProtocolAmendment)` — called by `ProtocolAmendmentListener`

**`ProtocolAmendmentLedgerEntry`** (qhorus datasource, V2025):
Fields: `amendmentId UUID`, `trialId UUID`, `proposedChange TEXT`, `status VARCHAR(50)`,
`supervisorRecommendation VARCHAR(50)` (nullable).

`domainContentBytes()`: `String.join("|", amendmentId, trialId, status,
proposedChange, Objects.toString(supervisorRecommendation, "")).getBytes(UTF_8)`.

`proposedChange` is included because the FDA audit trail must prove not just that a
proposal event occurred but what was proposed. Same reasoning as `AdverseEventLedgerEntry`
including `grade`.

### Tracking issue

casehubio/clinical#86 — "Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy when
engine#101 lands"

---

## 3. `ThreeSiteShowcaseTest`

Single `@QuarkusTest` method `three_site_oncology_showcase()`. Class Javadoc references
§7.4 and `docs/comparison/clinicalagent.md`.

**Injected test helpers:**
```java
@Inject WorkItemQueries workItemQueries;   // scanAll() wrapped in TX
@Inject LedgerEntryRepository ledgerRepo;  // for amendment ledger count assertion
```

**Setup:** register trial (unique `protocolId`), add 3 sites (A, B, C), activate trial.

---

**Site A — Eligibility screening:**
```
POST /trials/{t}/sites/{siteA}/patients          → enroll A-001
POST /trials/{t}/sites/{siteA}/patients/{e}/screen
    body: criterion-7 marginal, criterion-11 marginal
Assert HTTP 200: enrollmentStatus = "SCREENING", screeningResult = "MARGINAL"

Awaitility.await().atMost(5, SECONDS) until:
    workItemQueries.scanAll()
        .stream()
        .anyMatch(wi -> wi.getCandidateGroups().contains("irb-committee"))

Assert: matching WorkItem.expiresAt ≤ Instant.now().plus(73, HOURS)

// Ledger assertion via patient audit chain (same mechanism as Site B)
Assert: GET /trials/{t}/sites/{siteA}/patients/{e}/ledger/verify
    → { "valid": true, "merkleRoot": <non-null> }
```

WorkItem lookup uses `WorkItemQueries.scanAll()` (established pattern: `IrbGateLifecycleTest`).
Patient ledger verify covers the `EligibilityScreeningLedgerEntry` since its
`subjectId = enrollmentId`.

---

**Site B — Adverse event escalation:**
```
POST /trials/{t}/sites/{siteB}/patients          → enroll B-001
POST /trials/{t}/sites/{siteB}/patients/{e}/adverse-events
    body: { grade: "GRADE_3", occurredAt: now-2h, unexpected: true }
Assert HTTP 201: slaDeadline within 24h; workItemId = null (engine-managed)

Awaitility.await().atMost(5, SECONDS) until:
    GET /trials/{t}/sites/{siteB}/patients/{e}/adverse-events/{ae}
    .regulatorySubmissionStatus == "PENDING"

// FDA independent verification — Merkle proof for complete patient chain
Assert: GET /trials/{t}/sites/{siteB}/patients/{e}/ledger/verify
    → { "valid": true, "merkleRoot": <non-null> }
```

`regulatorySubmissionStatus = PENDING` is set by `RegulatorySubmissionCaseService`
(`@ObservesAsync`) — Awaitility is required.

---

**Site C — Protocol amendment:**
```
POST /trials/{t}/amendments
    body: { proposedChange: "Dose escalation amendment v2" }
Assert HTTP 201: status = "PROPOSED"

Awaitility.await().atMost(5, SECONDS) until:
    GET /trials/{t}/amendments/{id}
    .status == "APPROVED"

Assert: amendment.supervisorRecommendation = "PROCEED"

// Ledger: proposal + resolution entries written (no verify endpoint; direct repo query)
Assert: ledgerRepo.findBySubjectId(amendmentId, "default")
    .stream()
    .filter(e -> e instanceof ProtocolAmendmentLedgerEntry)
    .count() == 2
```

Amendment ledger entries are in the qhorus datasource. `LedgerEntryRepository.findBySubjectId`
is the actual API (confirmed from decompiled interface). The instanceof filter guards against
spurious entries from background observers writing to the same subjectId — same fragility
noted in the 2026-06-03 blog entry regarding `AdverseEventServiceTest`. Two typed entries
expected: proposal (from `ProtocolAmendmentService`) + resolution (from
`ProtocolAmendmentListener`). A REST verify endpoint for amendments is not in scope here.

---

## 4. Test Plan

Per CLAUDE.md: unit tests for pure logic, integration tests for Panache/CDI/REST.

### Unit tests (no Quarkus, Mockito for dependencies)

**`EligibilityScreeningServiceTest`**
- `marginal_criterion_takes_priority_over_excluded` — one `marginal=true` AND one `met=false`
  non-marginal → `MARGINAL` (not `EXCLUDED`; documents the precedence rule)
- `all_non_marginal_failed_criterion_results_in_EXCLUDED`
- `all_criteria_met_results_in_CRITERIA_MET`

**`EligibilityScreeningCaseServiceTest`**
- `phase1_idempotency_guard_skips_when_status_is_REQUESTED`
- `phase1_sets_REQUESTED_on_NONE_status`
- `phase4_sets_FAILED_on_startCase_exception`
- `context_serializes_criterion_results_as_maps_not_records` — asserts each entry in
  `criteriaResults` is `Map<String, Object>`, not a `CriterionResult` instance

**`ProtocolAmendmentCaseServiceTest`**
- `phase1_idempotency_guard_skips_when_not_NONE`
- `phase1_sets_REQUESTED`
- `phase4_sets_FAILED_on_startCase_exception`
- `initial_context_contains_amendmentId_as_string` — listener discrimination depends on it


**`EligibilityScreeningLedgerWriterTest`** (Mockito-mocked `LedgerEntryRepository`)
- `writeScreeningEntry_sets_criteriaCount_to_total_list_size` — two criteria in, `criteriaCount = 2`
- `writeScreeningEntry_sets_marginalCount_to_count_of_marginal_true` — one `marginal=true`, `marginalCount = 1`
- `writeScreeningEntry_uses_correct_screeningResult` — MARGINAL result written to entry

**`ProtocolAmendmentLedgerWriterTest`** (Mockito-mocked `LedgerEntryRepository`)
- `writeProposalEntry_includes_proposedChange`
- `writeResolutionEntry_includes_supervisorRecommendation`
- `sequenceNumber_increments_between_proposal_and_resolution_entries`

**`DefaultProtocolAmendmentAdvisorTest`**
- `always_returns_PROCEED`

### Integration tests (`@QuarkusTest` with H2)

**`EligibilityScreeningIntegrationTest`** — focuses on the service + ledger path isolated
from the engine case:
- `screen_MARGINAL_sets_SCREENING_status_and_writes_ledger_entry`
- `screen_CRITERIA_MET_sets_ELIGIBLE_status`
- `screen_EXCLUDED_sets_INELIGIBLE_status`

**`ProtocolAmendmentListenerTest`** (`@QuarkusTest` — listener calls `ProtocolAmendment.findById()`,
a Panache static; cannot be mocked without Quarkus context. Reference: `AeEscalationListenerMemoryTest`
pattern — `@InjectMock` for CDI deps, real entity in `@BeforeEach @Transactional setup()`).
- `proceed_sets_APPROVED_and_COMPLETED_and_non_null_recommendation`
- `halt_sets_HALTED_and_COMPLETED`
- `refer_to_dsmb_sets_SUPERVISED_and_COMPLETED`
- `redelivery_skipped_when_supervisorRecommendation_already_set`
- `non_amendment_case_skipped_when_amendmentId_absent_from_context`
- `writes_resolution_ledger_entry_exactly_once`

**`ProtocolAmendmentIntegrationTest`** — exercises the full REST + observer path:
- `propose_creates_amendment_PROPOSED_and_writes_proposal_ledger_entry`
- `propose_then_await_APPROVED_writes_resolution_ledger_entry`
  (Awaitility polling GET /amendments/{id})

### End-to-end

**`ThreeSiteShowcaseTest`** — the narrative showcase described in §3. Tests only the
PROCEED happy path. HALT and REFER_TO_DSMB branches are covered in
`ProtocolAmendmentListenerTest` via `@InjectMock`.

---

## 5. `ClinicalLayerComplianceTest` (rename)

`ShowcaseScenarioTest` renamed to `ClinicalLayerComplianceTest` via IntelliJ refactor.
All 4 existing test methods kept unchanged. Class Javadoc updated.

---

## 6. `docs/comparison/clinicalagent.md`

```markdown
# casehub-clinical vs ClinicalAgent (arXiv 2404.14777)

ClinicalAgent (ACM BCB '24) runs a linear single-site pipeline with no compliance
infrastructure.

| GCP / FDA requirement | ClinicalAgent | casehub-clinical | Layer |
|---|---|---|---|
| Adverse event SLA — Grade 3/4 within 24h | No deadline tracking | WorkItem.claimDeadline — AdverseEventService | 2 |
| PI authorisation for protocol deviations | Agent autonomous | COMMAND commitment — ProtocolDeviationService | 3 |
| FDA tamper-evident audit | No audit trail | Merkle MMR — AdverseEventLedgerEntry | 4 |
| IRB gate for CRITICAL deviations | Not addressed | deviation-review.yaml humanTask; 72h WorkItem | 5 |
| GDPR consent withdrawal (Art.17) | Not applicable | ConsentWithdrawalService + LedgerErasureService | 8 |
| Multi-site independence | Single-site linear pipeline | Trial-level CaseInstance; blackboard signals | 6 |
| Trust-weighted safety routing | No trust model | ClinicalTrustRoutingPolicyProvider; EigenTrust | 7 |
| IND expedited safety reporting | Not addressed | RegulatorySubmissionCaseService; 21 CFR 312.32 | 7 |
| Eligibility screening accountability | Agent decides; no record | EligibilityScreeningLedgerEntry; IRB gate if marginal | 9 |
| Protocol amendment LLM supervision | Not addressed | ProtocolAmendmentAdvisor SPI (clinical#86 / engine#101) | 9 |

Layer 9 = Showcase — new domain features exercising existing foundation layers without a
new foundation module integration.

FDA auditor verification (no server access required):
  GET /trials/{t}/sites/{s}/patients/{e}/ledger/verify
```

---

## 7. Migration numbers

| Migration | Datasource | Content |
|---|---|---|
| V122 | default | patient_enrollment: screening_result, screening_completed_at, eligibility_engine_case_id, eligibility_screening_case_status |
| V123 | default | protocol_amendment table (inc. amendment_case_status) |
| V2024 | qhorus | eligibility_screening_ledger_entry join table |
| V2025 | qhorus | protocol_amendment_ledger_entry join table |

---

## 8. New classes summary

| Class | Module | Notes |
|---|---|---|
| `EligibilityScreeningResult` | api/model | Enum: CRITERIA_MET, MARGINAL, EXCLUDED |
| `EligibilityScreeningCaseStatus` | api/model | Enum: NONE, REQUESTED, COMPLETED, FAILED |
| `CriterionResult` | api/model | Record: id, met, marginal |
| `EligibilityScreeningEvent` | api | CDI event record |
| `ProtocolAmendmentProposedEvent` | api | CDI event record |
| `ProtocolAmendmentStatus` | api/model | Enum: PROPOSED, SUPERVISED, APPROVED, HALTED |
| `AmendmentCaseStatus` | api/model | Enum: NONE, REQUESTED, COMPLETED, FAILED |
| `AmendmentRecommendation` | api/spi | Enum: PROCEED, REFER_TO_DSMB, HALT |
| `ProtocolAmendmentContext` | api/spi | Record: input to advisor |
| `ProtocolAmendmentAdvisor` | api/spi | SPI interface |
| `DefaultProtocolAmendmentAdvisor` | runtime/service | @DefaultBean stub → PROCEED |
| `EligibilityScreeningService` | runtime/service | Screen logic + ledger write + event |
| `EligibilityScreeningLedgerWriter` | runtime/service | ADR-0002; writeScreeningEntry() |
| `EligibilityScreeningLedgerEntry` | runtime/ledger | LedgerEntry subclass; V2024 |
| `EligibilityScreeningCaseHub` | runtime/service | YamlCaseHub wrapper |
| `EligibilityScreeningCaseService` | runtime/service | Four-phase observer |
| `ProtocolAmendmentService` | runtime/service | @Transactional persist + ledger + event |
| `ProtocolAmendmentLedgerWriter` | runtime/service | ADR-0002; proposal + resolution |
| `ProtocolAmendmentLedgerEntry` | runtime/ledger | LedgerEntry subclass; V2025 |
| `ProtocolAmendmentCaseHub` | runtime/service | YamlCaseHub + Java-function worker |
| `ProtocolAmendmentCaseService` | runtime/service | Four-phase observer |
| `ProtocolAmendmentListener` | runtime/service | GoalReached observer; CaseInstanceRepository |
| `ProtocolAmendmentResource` | runtime/resource | POST + GET /trials/{t}/amendments |
| `ThreeSiteShowcaseTest` | runtime/test | §7.4 narrative integration test |

---

## Out of scope

- `EligibilityScreeningLedgerWriter.writeResolutionEntry()` and IRB completion listener —
  IRB lifecycle covered in `IrbGateLifecycleTest`; showcase asserts WorkItem created only
- Per-criterion storage table — summary sufficient for FDA audit
- Protocol amendment verify endpoint (`GET .../amendments/{id}/ledger/verify`) — amendment
  ledger asserted via injected `LedgerEntryRepository` in the showcase test
- Protocol amendment REFER_TO_DSMB and HALT paths exercised in tests — reachable via
  `@InjectMock`; not tested in showcase (stub always returns PROCEED)
- Cross-site DSMB assertion — Grade 3 (Site B) doesn't meet ≥2 simultaneous Grade 4+;
  DSMB rollup covered in `DsmbRollupTest`