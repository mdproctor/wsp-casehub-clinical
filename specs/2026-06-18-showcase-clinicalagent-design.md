# 3-Site Showcase + ClinicalAgent Comparison — Design Spec

**Issue:** casehubio/clinical#10
**Branch:** issue-10-showcase-clinicalagent
**Date:** 2026-06-18

---

## Overview

Delivers the §7.4 showcase scenario from `tutorial-strategy.md`: a 3-site oncology trial
exercising all 7 completed layers in a single narrative integration test, plus a
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

**`EnrollmentStatus` additions** (`api/model/`):
```
ELIGIBILITY_REVIEW  — IRB consultation pending
ELIGIBLE            — all criteria met, ready for consent
INELIGIBLE          — excluded by screening
```

**`PatientEnrollment` new fields** (default datasource, V122):
- `screening_result VARCHAR(50)` — nullable until screened
- `screening_completed_at TIMESTAMP WITH TIME ZONE` — nullable
- `eligibility_engine_case_id UUID` — nullable; set when MARGINAL engine case starts; used by listener to route `GoalReached` events back to the enrollment

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

Returns 200 with updated enrollment (screeningResult, enrollmentStatus).

### `EligibilityScreeningService`

`@ApplicationScoped`. Called from `PatientResource` (not CDI event — synchronous result
needed for REST response).

Logic:
- Any `marginal=true` → `MARGINAL`
- Any `met=false` (non-marginal) → `EXCLUDED`
- All `met=true` → `CRITERIA_MET`

In one `@Transactional` call:
1. Write `EligibilityScreeningLedgerEntry` (qhorus datasource)
2. Update `enrollment.screeningResult` + `screeningCompletedAt` + `enrollmentStatus`
3. If `MARGINAL` → fire `EligibilityScreeningEvent` CDI event (async, outside TX boundary)

### `EligibilityScreeningLedgerEntry` (qhorus datasource, V2024)

Extends `LedgerEntry`. Fields: `enrollmentId UUID`, `screeningResult VARCHAR(50)`,
`criteriaCount INT`, `marginalCount INT`.

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
          filter: ".screeningResult == \"MARGINAL\" and .irbConsultation == null"
      humanTask:
        title: "IRB eligibility consultation — marginal criteria"
        expiresIn: PT72H
        candidateGroups: [irb-committee]
        inputMapping: "{ enrollmentId: .enrollmentId, criteriaResults: .criteriaResults }"
        outputMapping: "{ irbConsultation: . }"
```

**`EligibilityScreeningCaseHub`** extends `YamlCaseHub("clinical/eligibility-screening.yaml")`.
`@ApplicationScoped`. No `getDefinition()` override — YAML only.

**`EligibilityScreeningCaseService`** — `@ApplicationScoped`, observes
`@ObservesAsync EligibilityScreeningEvent`. Three-phase pattern (same as
`TrialActivationService`): validate → `startCase().join()` outside TX → persist `engineCaseId`.

Initial case context:
```json
{ "enrollmentId": "...", "screeningResult": "MARGINAL", "criteriaResults": [...] }
```

### Compliance gap closed

Eligibility agent decisions are now tamper-evident (ledger entry) and adaptive
(marginal criteria trigger formal IRB gate rather than autonomous agent rejection).
ClinicalAgent has no equivalent.

---

## 2. Protocol Amendment (Site C)

### Domain model

**`ProtocolAmendmentStatus` enum** (`api/model/`):
```
PROPOSED, SUPERVISED, APPROVED, REJECTED, HALTED
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

### REST endpoint

`POST /trials/{trialId}/amendments` → new `ProtocolAmendmentResource`.

Request: `{ "proposedChange": "Dose escalation amendment v2" }`

Returns 201 with created amendment (status = PROPOSED).

### LLM supervisor slot

**`ProtocolAmendmentContext` record** (`api/spi/`):
```java
public record ProtocolAmendmentContext(UUID amendmentId, UUID trialId,
    String proposedChange, Map<String, Object> trialBlackboardSnapshot) {}
```

**`ProtocolAmendmentAdvisor` SPI** (`api/spi/`):
```java
public interface ProtocolAmendmentAdvisor {
    AmendmentRecommendation advise(ProtocolAmendmentContext context);
}
```

**`DefaultProtocolAmendmentAdvisor`** (`runtime/service/`, `@DefaultBean @ApplicationScoped`):
Always returns `AmendmentRecommendation.PROCEED`. Javadoc: "Stub — replace with
LlmPlanningStrategy integration when casehubio/engine#101 lands (tracked: casehubio/clinical#86)."

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
          filter: ".advisorRecommendation == null"
      capability: protocol-amendment-advisor

    - name: amendment-approved
      on:
        contextChange:
          filter: ".advisorRecommendation == \"PROCEED\""
      humanTask:
        title: "Protocol amendment approved — file with IRB"
        expiresIn: PT168H
        candidateGroups: [sponsor-representatives]
        inputMapping: "{ amendmentId: .amendmentId, proposedChange: .proposedChange }"
        outputMapping: "{ amendmentFiled: . }"
```

**`ProtocolAmendmentCaseHub`** extends `YamlCaseHub("clinical/protocol-amendment.yaml")`.
Overrides `getDefinition()` to register Java-function worker for `protocol-amendment-advisor`
capability. The function calls `ProtocolAmendmentAdvisor.advise(context)` and returns
`{ "advisorRecommendation": recommendation.name() }`.

**`ProtocolAmendmentCaseService`** — `@ApplicationScoped @ObservesAsync ProtocolAmendmentProposedEvent`.
Three-phase: persist amendment → start case → persist `engineCaseId`.

**`ProtocolAmendmentListener`** — observes `CaseLifecycleEvent("GoalReached")`.
Looks up `ProtocolAmendment.findByEngineCaseId(event.caseId())` (new static finder on entity).
Updates `amendment.status = APPROVED` and `supervisorRecommendation` from case context.

### `ProtocolAmendmentLedgerEntry` (qhorus datasource, V2025)

Fields: `amendmentId UUID`, `trialId UUID`, `proposedChange TEXT`, `status VARCHAR(50)`.

`domainContentBytes()` → `String.join("|", amendmentId, trialId, status, proposedChange).getBytes(UTF_8)`.

Written in `ProtocolAmendmentResource.propose()` when amendment is created (PROPOSED state).

### Tracking issue

casehubio/clinical#86 — "Wire ProtocolAmendmentAdvisor to LlmPlanningStrategy when engine#101 lands"

---

## 3. `ThreeSiteShowcaseTest`

Single `@QuarkusTest` method `three_site_oncology_showcase()`. Narrative comment at class
level references §7.4 and `docs/comparison/clinicalagent.md`.

**Setup:** registers trial, adds 3 sites (A, B, C), activates trial (starts trial-level
engine case).

**Site A — Eligibility screening:**
- Enroll patient A-001
- `POST .../screen` — criteria 7 and 11 marginal
- Assert: `enrollmentStatus = ELIGIBILITY_REVIEW`
- Assert: IRB consultation WorkItem exists with `expiresAt` ≤ 72h from now

**Site B — Adverse event escalation:**
- Enroll patient B-001
- `POST .../adverse-events` — GRADE_3, `unexpected=true`
- Assert: `slaDeadline` within 24h
- Assert: `regulatorySubmissionStatus = PENDING` (IND expedited report triggered)
- Assert: `AdverseEventLedgerEntry` written (`subjectId = enrollmentId`)
- Assert: `GET .../ledger/verify` → `{ "valid": true, "merkleRoot": <non-null> }`

**Site C — Protocol amendment:**
- `POST /trials/{t}/amendments` — "Dose escalation amendment v2"
- Assert: `status = PROPOSED` on creation
- `Awaitility.await().atMost(5, SECONDS)` for engine case to process advisor
- Assert: `status = APPROVED`
- Assert: `ProtocolAmendmentLedgerEntry` written

**Merkle proof assertion** demonstrates FDA independent verification claim from §7.4.
Uses existing `LedgerVerificationService` via the REST endpoint already in `PatientResource`.

---

## 4. `ClinicalLayerComplianceTest` (rename)

`ShowcaseScenarioTest` renamed to `ClinicalLayerComplianceTest` via IntelliJ refactor.
All 4 existing test methods kept unchanged. Class Javadoc updated to reflect purpose:
layer-by-layer compliance verification (Layers 1–3 paths, not full showcase narrative).

---

## 5. `docs/comparison/clinicalagent.md`

Markdown doc in project repo `docs/comparison/` (new directory). Content:

```
# casehub-clinical vs ClinicalAgent (arXiv 2404.14777)

| GCP / FDA requirement | ClinicalAgent | casehub-clinical | Layer |
|---|---|---|---|
| Adverse event SLA — Grade 3/4 within 24h | No deadline tracking | WorkItem.claimDeadline — AdverseEventService | 2 |
| Eligibility screening accountability | Agent decides; no record | EligibilityScreeningLedgerEntry; IRB gate if marginal | 5 |
| PI authorisation for deviations | Agent autonomous | COMMAND commitment — ProtocolDeviationService | 3 |
| IRB gate for CRITICAL deviations | Not addressed | deviation-review.yaml humanTask; 72h WorkItem | 5 |
| Protocol amendment LLM supervision | Not addressed | ProtocolAmendmentAdvisor SPI (engine#101 pending) | — |
| GDPR consent withdrawal (Art.17) | Not applicable | ConsentWithdrawalService + LedgerErasureService | 8 |
| Multi-site independence | Single-site linear pipeline | Trial-level CaseInstance; per-site blackboard signals | 6 |
| FDA tamper-evident audit | No audit trail | Merkle MMR; GET .../ledger/verify | 4 |
| Trust-weighted safety routing | No trust model | ClinicalTrustRoutingPolicyProvider; EigenTrust | 7 |
| IND expedited safety reporting | Not addressed | RegulatorySubmissionCaseService; 21 CFR 312.32 | 7 |

FDA auditor verification (no server access required):
GET /trials/{t}/sites/{s}/patients/{e}/ledger/verify
```

---

## 6. Migration numbers

| Migration | Datasource | Content |
|---|---|---|
| V122 | default | patient_enrollment screening fields |
| V123 | default | protocol_amendment table |
| V2024 | qhorus | eligibility_screening_ledger_entry join table |
| V2025 | qhorus | protocol_amendment_ledger_entry join table |

---

## Out of scope

- Full `EligibilityIrbCompletionListener` — IRB lifecycle already covered in `IrbGateLifecycleTest`; showcase asserts WorkItem created, not IRB response
- Per-criterion storage table — screening result summary sufficient for FDA audit
- Protocol amendment REFER_TO_DSMB path — stub always returns PROCEED; fuller routing tested when #86 lands
- Cross-site DSMB assertion — Grade 3 (Site B) doesn't meet the ≥2 simultaneous Grade 4+ threshold; DSMB rollup is covered in `DsmbRollupTest`
