# 3-Site Showcase + ClinicalAgent Comparison — Design Spec

**Issue:** casehubio/clinical#10
**Branch:** issue-10-showcase-clinicalagent
**Date:** 2026-06-18 (rev 2)

---

## Overview

Delivers the §7.4 showcase scenario from `tutorial-strategy.md`: a 3-site oncology trial
exercising all completed layers in a single narrative integration test, plus a
ClinicalAgent comparison document.

Three deliverables:

1. **Two new domain features** — eligibility screening (Site A) and protocol amendment (Site C)
2. **`ThreeSiteShowcaseTest`** — single narrative `@QuarkusTest` orchestrating all 3 sites
3. **`docs/comparison/clinicalagent.md`** — GCP/FDA gap table with class/layer references

Also: rename `ShowcaseScenarioTest` → `ClinicalLayerComplianceTest` (layer-by-layer
compliance tests, not showcase).

---

## 1. Eligibility Screening (Site A)

### Domain model additions

**`EligibilityScreeningResult` enum** (`api/model/`):
```
CRITERIA_MET, MARGINAL, EXCLUDED
```

**`EnrollmentStatus`** — no new values needed. The existing FHIR-sourced enum already has:
- `SCREENING` — used when patient is in IRB consultation after marginal result
- `ELIGIBLE` — used when all criteria are met
- `INELIGIBLE` — used when screening excludes the patient

`EligibilityScreeningService` maps results to existing statuses:
`MARGINAL → SCREENING`, `CRITERIA_MET → ELIGIBLE`, `EXCLUDED → INELIGIBLE`.

**`PatientEnrollment` new fields** (default datasource, V122):
- `screening_result VARCHAR(50)` — nullable until screened
- `screening_completed_at TIMESTAMP WITH TIME ZONE` — nullable
- `eligibility_engine_case_id UUID` — nullable; set in Phase 3 when MARGINAL engine case
  starts; used by `EligibilityScreeningListener` to route `GoalReached` events back to
  the correct enrollment

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

`CriterionResult` is a record in `api/model/`: `String id`, `boolean met`, `boolean marginal`.

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

`@ApplicationScoped`. Called synchronously from `PatientResource` — result is needed for
the REST response.

Result determination logic:
- Any `marginal=true` → `MARGINAL`
- Any `met=false` (non-marginal) → `EXCLUDED`
- All `met=true` → `CRITERIA_MET`

One `@Transactional` call covering:
1. Write `EligibilityScreeningLedgerEntry` via `EligibilityScreeningLedgerWriter.writeScreeningEntry()`
2. Update `enrollment.screeningResult`, `screeningCompletedAt`, `enrollmentStatus`
3. If `MARGINAL` → `Event<EligibilityScreeningEvent>.fireAsync(event)` (CDI async delivery
   occurs after TX commit; `EligibilityScreeningCaseService` observes it)

### `EligibilityScreeningLedgerWriter`

`@ApplicationScoped`, per ADR-0002. Owns `sequenceNumber` computation via
`findLatestBySubjectId`. Single method now; `writeResolutionEntry()` to be added when
the IRB completion listener lands (out of scope here).

**`EligibilityScreeningLedgerEntry`** (qhorus datasource, V2024):
Fields: `enrollmentId UUID`, `screeningResult VARCHAR(50)`, `criteriaCount INT`,
`marginalCount INT`.

`domainContentBytes()` → `String.join("|", enrollmentId, screeningResult,
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

The positive guard (`.enrollmentId != null`) prevents the binding from firing on the empty
context the engine presents before initial context is applied (established pattern from
`susar-oversight.yaml`).

**`EligibilityScreeningCaseHub`** extends `YamlCaseHub("clinical/eligibility-screening.yaml")`.
`@ApplicationScoped`. No `getDefinition()` override — YAML only.

**`EligibilityScreeningCaseService`** — `@ApplicationScoped`, observes
`@ObservesAsync EligibilityScreeningEvent`. Three-phase pattern (reference:
`SusarOversightCaseService`):

- **Phase 1** `@Transactional prepareAndMark(event)`: load `PatientEnrollment` by
  `enrollmentId`, idempotency guard — if `enrollmentStatus != SCREENING` return null
  (already processed), build and return initial context map
- **Phase 2**: `eligibilityScreeningCaseHub.startCase(ctx).toCompletableFuture().join()`
  outside any TX boundary
- **Phase 3** `@Transactional persistCaseId(enrollmentId, caseId)`: set
  `enrollment.eligibilityEngineCaseId = caseId`

Initial case context:
```json
{ "enrollmentId": "...", "tenantId": "...", "screeningResult": "MARGINAL",
  "criteriaResults": [...] }
```

### Compliance gap closed

Eligibility agent decisions are tamper-evident (`EligibilityScreeningLedgerEntry`) and
adaptive (marginal criteria trigger a formal IRB gate rather than an autonomous agent
rejection). ClinicalAgent has no equivalent.

---

## 2. Protocol Amendment (Site C)

### Domain model

**`ProtocolAmendmentStatus` enum** (`api/model/`):
```
PROPOSED     — created, awaiting advisor
SUPERVISED   — advisor running (idempotency guard, set in Phase 1)
APPROVED     — advisor returned PROCEED
HALTED       — advisor returned HALT
REJECTED     — reserved for human override (reachable after #86)
```

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
  supervisor_recommendation VARCHAR(50),
  engine_case_id UUID,
  proposed_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

Static finder: `ProtocolAmendment.findByEngineCaseId(UUID caseId)` — used by
`ProtocolAmendmentListener` to discriminate `GoalReached` events.

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

### REST endpoint

`POST /trials/{trialId}/amendments` → new `ProtocolAmendmentResource`.

Request: `{ "proposedChange": "Dose escalation amendment v2" }`

Returns 201 with created amendment (`status = PROPOSED`).

Resource delegates to `ProtocolAmendmentService.propose()`.

### `ProtocolAmendmentService`

`@ApplicationScoped`. Owns persist + ledger write + CDI event in one `@Transactional` call:
1. Create and persist `ProtocolAmendment` (`status = PROPOSED`)
2. Write `ProtocolAmendmentLedgerEntry` via `ProtocolAmendmentLedgerWriter.writeProposalEntry()`
3. `Event<ProtocolAmendmentProposedEvent>.fireAsync(event)`

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
Always returns `AmendmentRecommendation.PROCEED`. Javadoc:
"Stub — replace with LlmPlanningStrategy integration when casehubio/engine#101 lands
(tracked: casehubio/clinical#86)."

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

Design note: the case is advisory-only. The advisor's recommendation IS the decision.
No downstream humanTask lives in this case definition — filing and ratification are
downstream operational steps outside this case scope, to be added when #86 lands. The
goal fires when `advisorRecommendation` is set by the capability worker; the listener
reads it and routes.

Positive guard (`.amendmentId != null`) prevents the binding from firing on empty
context before initial context is applied.

**`ProtocolAmendmentCaseHub`** extends `YamlCaseHub("clinical/protocol-amendment.yaml")`.
Overrides `getDefinition()` to register Java-function worker for `protocol-amendment-advisor`
capability. The function calls `ProtocolAmendmentAdvisor.advise(context)` and returns
`{ "advisorRecommendation": recommendation.name() }`.

**`ProtocolAmendmentCaseService`** — `@ApplicationScoped`, observes
`@ObservesAsync ProtocolAmendmentProposedEvent`. Three-phase pattern:

- **Phase 1** `@Transactional prepareAndMark(event)`: load `ProtocolAmendment` by
  `amendmentId`, idempotency guard — if `status != PROPOSED` return null (already processed),
  set `amendment.status = SUPERVISED` as guard, build and return initial context map
- **Phase 2**: `protocolAmendmentCaseHub.startCase(ctx).toCompletableFuture().join()`
  outside any TX boundary
- **Phase 3** `@Transactional persistCaseId(amendmentId, caseId)`: set
  `amendment.engineCaseId = caseId`

**`ProtocolAmendmentListener`** — `@ApplicationScoped @ObservesAsync CaseLifecycleEvent`.
Accepts both `"GoalReached"` and `"CaseCompleted"` event types with idempotency guard
(established pattern from `AeEscalationListener`, per engine#393 note in CLAUDE.md).

Discrimination: `ProtocolAmendment.findByEngineCaseId(event.caseId())` — returns null for
non-amendment cases, silently skips. For amendment cases:

```java
amendment.supervisorRecommendation = (String) ctx.get("advisorRecommendation");
amendment.status = switch (amendment.supervisorRecommendation) {
    case "PROCEED"       -> ProtocolAmendmentStatus.APPROVED;
    case "HALT"          -> ProtocolAmendmentStatus.HALTED;
    case "REFER_TO_DSMB" -> ProtocolAmendmentStatus.SUPERVISED;
    default -> throw new IllegalStateException(
        "Unknown recommendation: " + amendment.supervisorRecommendation);
};
protocolAmendmentLedgerWriter.writeResolutionEntry(amendment);
```

### `ProtocolAmendmentLedgerWriter`

`@ApplicationScoped`, per ADR-0002. Two methods:
- `writeProposalEntry(ProtocolAmendment)` — called by `ProtocolAmendmentService.propose()`
- `writeResolutionEntry(ProtocolAmendment)` — called by `ProtocolAmendmentListener`

**`ProtocolAmendmentLedgerEntry`** (qhorus datasource, V2025):
Fields: `amendmentId UUID`, `trialId UUID`, `status VARCHAR(50)`,
`supervisorRecommendation VARCHAR(50)` (nullable).

`domainContentBytes()` → `String.join("|", amendmentId, trialId, status,
Objects.toString(supervisorRecommendation, "")).getBytes(UTF_8)`.

### Tracking issue

casehubio/clinical#86 — "Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy when
engine#101 lands"

---

## 3. `ThreeSiteShowcaseTest`

Single `@QuarkusTest` method `three_site_oncology_showcase()`. Class Javadoc references
§7.4 and `docs/comparison/clinicalagent.md`.

**Injected test helpers:**
```java
@Inject WorkItemQueries workItemQueries;       // test-scope: scanAll() wrapped in TX
@Inject WorkItemService workItemService;
@Inject WorkItemLifecycleAdapter lifecycleAdapter;
@Inject ProtocolAmendmentCaseService amendmentCaseService; // for direct invocation if needed
```

**Setup:** register trial (unique protocolId), add 3 sites (A, B, C), activate trial →
starts trial-level engine case.

---

**Site A — Eligibility screening:**
```
POST /trials/{t}/sites/{siteA}/patients          → enroll A-001
POST /trials/{t}/sites/{siteA}/patients/{e}/screen
    body: criteria 7 marginal, criteria 11 marginal
Assert: 200; enrollmentStatus = "SCREENING"; screeningResult = "MARGINAL"
Awaitility.await().atMost(5, SECONDS) until:
    workItemQueries.scanAll()
        .stream()
        .anyMatch(wi -> wi.getCandidateGroups().contains("irb-committee"))
Assert: matching WorkItem has expiresAt ≤ Instant.now().plus(73, HOURS)
Assert: EligibilityScreeningLedgerEntry written (subjectId = enrollmentId)
```

WorkItem lookup uses the established `WorkItemQueries.scanAll()` helper (injected
test-scope bean wrapping `WorkItemStore.scanAll()` in a transaction — same pattern as
`IrbGateLifecycleTest`).

---

**Site B — Adverse event escalation:**
```
POST /trials/{t}/sites/{siteB}/patients          → enroll B-001
POST /trials/{t}/sites/{siteB}/patients/{e}/adverse-events
    body: { grade: "GRADE_3", occurredAt: now-2h, unexpected: true }
Assert: 201; slaDeadline within 24h; workItemId = null (engine-managed)
Awaitility.await().atMost(5, SECONDS) until:
    GET /trials/{t}/sites/{siteB}/patients/{e}/adverse-events/{ae}
    .regulatorySubmissionStatus == "PENDING"
Assert: AdverseEventLedgerEntry written (subjectId = enrollmentId)
Assert: GET /trials/{t}/sites/{siteB}/patients/{e}/ledger/verify
    → { "valid": true, "merkleRoot": <non-null> }
```

`regulatorySubmissionStatus = PENDING` is set by `RegulatorySubmissionCaseService`
(`@ObservesAsync`) so Awaitility is required before asserting it.

---

**Site C — Protocol amendment:**
```
POST /trials/{t}/amendments
    body: { proposedChange: "Dose escalation amendment v2" }
Assert: 201; status = "PROPOSED"
Awaitility.await().atMost(5, SECONDS) until:
    GET /trials/{t}/amendments/{id}
    .status == "APPROVED"
Assert: amendment.supervisorRecommendation = "PROCEED"
Assert: ProtocolAmendmentLedgerEntry written (subjectId = amendmentId, two entries:
    PROPOSED + resolution)
```

Merkle proof assertion on Site B's AE ledger chain demonstrates the FDA independent
verification claim ("FDA can verify without server access").

---

## 4. `ClinicalLayerComplianceTest` (rename)

`ShowcaseScenarioTest` renamed to `ClinicalLayerComplianceTest` via IntelliJ refactor.
All 4 existing test methods kept unchanged. Class Javadoc updated: layer-by-layer
compliance verification (Layers 1–3 paths, not the full showcase narrative).

---

## 5. `docs/comparison/clinicalagent.md`

New directory `docs/comparison/` in project repo. Content:

```markdown
# casehub-clinical vs ClinicalAgent (arXiv 2404.14777)

ClinicalAgent is a peer-reviewed open-source baseline (ACM BCB '24) showing what naive
LLM trial coordination looks like. It runs a linear single-site pipeline with no
compliance infrastructure.

| GCP / FDA requirement | ClinicalAgent | casehub-clinical | Layer |
|---|---|---|---|
| Adverse event SLA — Grade 3/4 within 24h | No deadline tracking | WorkItem.claimDeadline — AdverseEventService | 2 |
| PI authorisation for protocol deviations | Agent autonomous | COMMAND commitment — ProtocolDeviationService | 3 |
| FDA tamper-evident audit | No audit trail | Merkle MMR — AdverseEventLedgerEntry | 4 |
| IRB gate for CRITICAL deviations | Not addressed | deviation-review.yaml humanTask; 72h WorkItem | 5 |
| Eligibility screening accountability | Agent decides; no record | EligibilityScreeningLedgerEntry; IRB gate if marginal | 9 |
| Protocol amendment LLM supervision | Not addressed | ProtocolAmendmentAdvisor SPI (engine#101 pending) | 9 |
| GDPR consent withdrawal (Art.17) | Not applicable | ConsentWithdrawalService + LedgerErasureService | 8 |
| Multi-site independence | Single-site linear pipeline | Trial-level CaseInstance; per-site blackboard signals | 6 |
| Trust-weighted safety routing | No trust model | ClinicalTrustRoutingPolicyProvider; EigenTrust | 7 |
| IND expedited safety reporting | Not addressed | RegulatorySubmissionCaseService; 21 CFR 312.32 | 7 |

FDA auditor verification (no server access required):
  GET /trials/{t}/sites/{s}/patients/{e}/ledger/verify
  Returns Merkle inclusion proof for the complete patient decision chain.
```

---

## 6. Migration numbers

| Migration | Datasource | Content |
|---|---|---|
| V122 | default | patient_enrollment: screening_result, screening_completed_at, eligibility_engine_case_id |
| V123 | default | protocol_amendment table |
| V2024 | qhorus | eligibility_screening_ledger_entry join table |
| V2025 | qhorus | protocol_amendment_ledger_entry join table |

---

## 7. New classes summary

| Class | Module | Notes |
|---|---|---|
| `EligibilityScreeningResult` | api/model | Enum: CRITERIA_MET, MARGINAL, EXCLUDED |
| `CriterionResult` | api/model | Record: id, met, marginal |
| `EligibilityScreeningEvent` | api | CDI event record |
| `ProtocolAmendmentProposedEvent` | api | CDI event record |
| `ProtocolAmendmentStatus` | api/model | Enum: PROPOSED, SUPERVISED, APPROVED, HALTED, REJECTED |
| `AmendmentRecommendation` | api/spi | Enum: PROCEED, REFER_TO_DSMB, HALT |
| `ProtocolAmendmentContext` | api/spi | Record: input to advisor |
| `ProtocolAmendmentAdvisor` | api/spi | SPI interface |
| `DefaultProtocolAmendmentAdvisor` | runtime/service | @DefaultBean stub |
| `EligibilityScreeningService` | runtime/service | Screen logic + ledger write + event |
| `EligibilityScreeningLedgerWriter` | runtime/service | ADR-0002 writer; writeScreeningEntry() |
| `EligibilityScreeningLedgerEntry` | runtime/ledger | LedgerEntry subclass; V2024 |
| `EligibilityScreeningCaseHub` | runtime/service | YamlCaseHub wrapper |
| `EligibilityScreeningCaseService` | runtime/service | Three-phase observer |
| `ProtocolAmendmentService` | runtime/service | persist + ledger + event |
| `ProtocolAmendmentLedgerWriter` | runtime/service | ADR-0002 writer; proposal + resolution entries |
| `ProtocolAmendmentLedgerEntry` | runtime/ledger | LedgerEntry subclass; V2025 |
| `ProtocolAmendmentCaseHub` | runtime/service | YamlCaseHub + Java-function worker |
| `ProtocolAmendmentCaseService` | runtime/service | Three-phase observer |
| `ProtocolAmendmentListener` | runtime/service | GoalReached observer; status routing |
| `ProtocolAmendmentResource` | runtime/resource | POST /trials/{t}/amendments |
| `ThreeSiteShowcaseTest` | runtime/test | §7.4 narrative integration test |

---

## Out of scope

- `EligibilityIrbCompletionListener` (and `EligibilityScreeningLedgerWriter.writeResolutionEntry()`)
  — IRB lifecycle already covered in `IrbGateLifecycleTest`; showcase asserts WorkItem created
- Per-criterion storage table — screening result summary sufficient for FDA audit
- Protocol amendment REFER_TO_DSMB and HALT paths exercised in tests — stub returns PROCEED;
  paths are code-reachable via @InjectMock but not tested until #86 lands
- Cross-site DSMB assertion — Grade 3 (Site B) doesn't meet ≥2 simultaneous Grade 4+ threshold;
  DSMB rollup covered in `DsmbRollupTest`
- `GET /trials/{t}/amendments/{id}` — needed by showcase test; a minimal GET handler
  must be added to `ProtocolAmendmentResource`
