# CBR Phase 1 — Wire CbrCaseMemoryStore for AE + Deviation + Amendment Precedents

**Issue:** casehubio/clinical#116 (parent epic: #115)
**Date:** 2026-07-07
**Status:** Approved

## Goal

Wire casehub-neocortex's `CbrCaseMemoryStore` into clinical to store and retrieve
case-based reasoning precedents for adverse events, protocol deviations, and protocol
amendments. Demonstrates all three CBR case types (FeatureVector, Plan, Textual) with
domain-appropriate data shapes.

## CBR Case Types

Three case types, each in `io.casehub.clinical.cbr`:

### 1. ClinicalAeCbrCase — FeatureVectorCbrCase

Stored on `AeEscalationCompletedEvent`. `caseType = "clinical-ae"`.

**Problem features** (what the AE looked like):

| Feature | Type | Notes |
|---|---|---|
| `grade` | Numeric(1, 5) | CTCAE grade as integer |
| `eventType` | Categorical | e.g. "NAUSEA", "HEPATOTOXICITY" — from `AdverseEvent.eventType` |
| `trialPhase` | Categorical | EARLY_PHASE_I through PHASE_IV — from `ClinicalTrial.phase` via site→trial traversal (see data access below) |
| `unexpected` | Categorical | boolean — from `AdverseEvent.unexpected` |
| `suspected` | Categorical | boolean — from `AdverseEvent.suspected` |

**Outcome features** (how it was resolved):

| Feature | Type | Notes |
|---|---|---|
| `safetyReviewOutcome` | Categorical | From `AeEscalationCompletedEvent.safetyReviewOutcome` |
| `dsmbEscalated` | Categorical | boolean — from `AeEscalationCompletedEvent.dsmbEscalated` |
| `indReportFiled` | Categorical | boolean — `AdverseEvent.regulatorySubmissionStatus != NONE` |
| `susarOversight` | Categorical | boolean — `AdverseEvent.susarOversightStatus != NONE` |

**Required text fields:**
- `problem` = structured narrative: `"Grade {grade} {eventType} in {trialPhase} trial, unexpected={unexpected}, suspected={suspected}"`
- `solution` = resolution summary: `"Safety review: {safetyReviewOutcome}. DSMB escalated: {dsmbEscalated}. IND report: {indReportFiled}. SUSAR oversight: {susarOversight}."`
- `outcome` = `"COMPLETED"` (all escalation cases that fire `AeEscalationCompletedEvent` completed successfully)

**Data access — `trialPhase`:** The writer loads `TrialSite.findById(event.siteId())` → `ClinicalTrial.findById(site.trialId)` → `trial.phase`. Two-hop traversal from event data.

**Feature schema divergence from issue #116:** Issue #116 listed `siteProfile` (categorical).
Dropped because `siteProfile` was undefined — no field or enum exists. If a concrete site
characterization emerges (e.g., enrollment volume tier), it can be added in a later phase.
`unexpected` and `suspected` were added as material regulatory facts on the AE entity that
determine SUSAR eligibility and IND reporting — directly relevant to AE outcome similarity.

Only Grade 3+ AEs get engine cases (and therefore `AeEscalationCompletedEvent`).
Grade 1-2 AEs follow a fixed path with no meaningful resolution variation — no CBR
case stored. This means cross-grade pattern detection (e.g., "this Grade 1 AE resembles
previous Grade 3 AEs") is not possible in Phase 1. Phase 6 (AE trajectory monitoring,
issue #119) addresses this with temporal trajectory CBR.

### 2. ClinicalDeviationCbrCase — PlanCbrCase

Stored on `ProtocolDeviationResolvedEvent`. `caseType = "clinical-deviation"`.

**Problem features:**

| Feature | Type | Notes |
|---|---|---|
| `deviationType` | Categorical | e.g. "CONSENT_VIOLATION" |
| `severity` | Categorical | MINOR, MAJOR, CRITICAL |
| `escalationRequirement` | Categorical | NONE, SPONSOR_NOTIFICATION, IRB_REVIEW — from `EscalationRequirement` enum |

**Outcome features:**

| Feature | Type | Notes |
|---|---|---|
| `piDecision` | Categorical | APPROVED, REJECTED, EXPIRED — from `ProtocolDeviationResolvedEvent.terminalStatus` mapped to `PiApprovalStatus` |
| `irbDecision` | Categorical | APPROVED, REJECTED, DEFERRED, EXPIRED, or null — from `IrbApproval.find("deviationId", deviationId)` → `irb.decision` |

**Required text fields:**
- `problem` = structured narrative: `"{severity} {deviationType} at site, escalation={escalationRequirement}"`
- `solution` = resolution summary: `"PI decision: {piDecision}. IRB decision: {irbDecision or N/A}."`
- `outcome` = `terminalStatus.name()` (APPROVED/REJECTED/EXPIRED/ESCALATED)

**Plan trace** — from the deviation-review engine case bindings:
- `pi-oversight` binding → step outcome (DONE/DECLINE/EXPIRED)
- `irb-committee` binding → step outcome (if CRITICAL path fired)

**Data access — PlanTrace:** The writer loads `ProtocolDeviation.findById(deviationId)` →
`deviation.engineCaseId`. If non-null, queries the engine runtime API:
`caseRuntime.getCase(engineCaseId)` → iterate `case.planItems()`, map each completed
binding to a `PlanTrace(bindingName, capabilityName, stepOutcome)`. The engine case
context exposes binding completion states via `PlanItem.status()`.

**Data access — `irbDecision`:** `IrbApproval.find("deviationId", deviationId)` returns
the IRB committee decision if an IRB gate fired. Null for MINOR/MAJOR deviations
that don't trigger IRB review.

**Feature schema divergence from issue #116:** Issue #116 listed `siteEnrollmentSize`
(numeric) and `piResponseHistory` (categorical). `siteEnrollmentSize` was dropped —
not on the entity, and site size is not a strong predictor of deviation resolution
outcome. `piResponseHistory` would require cross-deviation aggregation per PI —
complex data access with unclear similarity value at this phase. `escalationRequirement`
was added because it's directly on the entity and determines the resolution path
(NONE/SPONSOR_NOTIFICATION/IRB_REVIEW).

If engine case ID is null (deviation expired before case started), stored with
empty plan trace.

### 3. ClinicalAmendmentCbrCase — TextualCbrCase

Stored on `ProtocolAmendmentResolvedEvent` (new event). `caseType = "clinical-amendment"`.

No structured features. Pure text:
- `problem` = the `proposedChange` text from `ProtocolAmendment.proposedChange`
- `solution` = advisor recommendation name: `amendment.supervisorRecommendation.name()` (PROCEED/HALT/REFER_TO_DSMB)
- `outcome` = terminal status: `amendment.status.name()` (APPROVED/HALTED/SUPERVISED)

## CBR Writers

Three CDI event observers in `io.casehub.clinical.cbr`:

### AeResolutionCbrWriter

`@ObservesAsync AeEscalationCompletedEvent`. Loads `AdverseEvent` entity (for
eventType, suspected, regulatorySubmissionStatus, susarOversightStatus, tenantId)
and `TrialSite`/`ClinicalTrial` (for trialPhase). Builds the feature map
(problem + outcome features) and stores via `ClinicalCbrService`.

**Idempotency:** calls `eraseEntity(aeId.toString(), ae.tenantId)` before `store()`.

**`store()` parameters:**
- `caseType` = `"clinical-ae"`
- `entityId` = `aeId.toString()`
- `domain` = `ClinicalCbrDomains.AE`
- `tenantId` = `ae.tenantId`
- `caseId` = `ae.engineCaseId != null ? ae.engineCaseId.toString() : null`

### DeviationResolutionCbrWriter

`@ObservesAsync ProtocolDeviationResolvedEvent`. Loads `ProtocolDeviation` entity
for engineCaseId, `IrbApproval` for IRB decision. Queries engine case context for
binding completion states to construct `PlanTrace` records. If engineCaseId is
null, empty trace.

**Idempotency:** calls `eraseEntity(deviationId.toString(), event.tenantId())` before `store()`.

**`store()` parameters:**
- `caseType` = `"clinical-deviation"`
- `entityId` = `deviationId.toString()`
- `domain` = `ClinicalCbrDomains.DEVIATION`
- `tenantId` = `event.tenantId()`
- `caseId` = `deviation.engineCaseId != null ? deviation.engineCaseId.toString() : null`

### AmendmentResolutionCbrWriter

`@ObservesAsync ProtocolAmendmentResolvedEvent`. Loads `ProtocolAmendment` entity
for proposedChange, supervisorRecommendation, and status. Stores a `TextualCbrCase`.

**Idempotency:** calls `eraseEntity(amendmentId.toString(), amendment.tenantId)` before `store()`.

**`store()` parameters:**
- `caseType` = `"clinical-amendment"`
- `entityId` = `amendmentId.toString()`
- `domain` = `ClinicalCbrDomains.AMENDMENT`
- `tenantId` = `amendment.tenantId`
- `caseId` = `amendment.engineCaseId != null ? amendment.engineCaseId.toString() : null`

**New event required:** `ProtocolAmendmentResolvedEvent` — fired by
`ProtocolAmendmentStatusUpdater` when status reaches any terminal state (APPROVED,
HALTED, or SUPERVISED). One `fireAsync()` call added to each of the three terminal
branches in `applyRecommendation()`, consistent with AE and deviation event patterns.

**Transaction safety:** `applyRecommendation()` is `@Transactional(REQUIRES_NEW)`.
The `fireAsync()` call dispatches the event asynchronously before the transaction
commits. The CBR writer should use `@ObservesAsync` (matching existing codebase
convention) and tolerate the rare case of receiving an event for a rolled-back
transaction — the idempotent erase-before-store pattern handles this gracefully.

## Shared Infrastructure

### ClinicalCbrService

Thin wrapper around `CbrCaseMemoryStore`. Injected by all three writers for `store()`
and by the REST endpoints for `retrieveSimilar()`. Handles tenant scoping.

### ClinicalCbrSchemaInitializer

`@ApplicationScoped @Startup`. Registers three `CbrFeatureSchema` instances (one per
case type) at boot. The amendment schema is empty (TextualCbrCase has no features).

### ClinicalCbrDomains

Constants class in `api` module with three `MemoryDomain` instances:
- `AE = MemoryDomain("clinical-ae")`
- `DEVIATION = MemoryDomain("clinical-deviation")`
- `AMENDMENT = MemoryDomain("clinical-amendment")`

**Divergence from `ClinicalMemoryDomains`:** The existing `ClinicalMemoryDomains`
(runtime) organizes recall memory by *scope* (PATIENT, SITE, DRUG, IRB) — each
domain aggregates contextual knowledge for agents within that scope. CBR domains
organize by *case type* — each domain contains structurally homogeneous cases for
similarity matching. The two organizing principles serve different purposes and are
intentionally distinct.

`ClinicalCbrDomains` lives in `api` (not `runtime`) because the REST response records
in `api` reference these domain constants.

## REST Endpoints

### GET /trials/{trialId}/adverse-events/{aeId}/precedents

Builds `CbrQuery` from current AE features. Problem features weighted 1.0, outcome
features weighted 0.0 (find AEs that LOOKED like this one). `topK = 10`,
`minSimilarity = 0.3`, `vectorWeight = 0.0` (Phase 1 — no embeddings; Phase 3 will
revisit when semantic text embeddings arrive). Returns `List<AePrecedentResponse>`.

**Routing:** Trial-level endpoint on `TrialDashboardResource` (matching existing
`/trials/{trialId}/adverse-events` list pattern). Precedent search operates across
all sites in the trial — site/patient context is unnecessary for similarity queries.

### GET /trials/{trialId}/deviations/{devId}/precedents

Builds query from current deviation features. Same weighting (match on problem).
`vectorWeight = 0.0`. Returns `List<DeviationPrecedentResponse>` including
`List<PlanStepResponse>`.

**Routing:** Trial-level endpoint on `TrialDashboardResource`. Deviation precedent
search spans all sites — the existing `DeviationResource` at
`/trials/{trialId}/sites/{siteId}/deviations/{deviationId}` is site-scoped,
but precedent queries need trial-wide visibility.

### GET /trials/{trialId}/amendments/{amendmentId}/precedents

Builds query with `problem = amendment.proposedChange` for text matching.
`vectorWeight = 0.0`. Returns `List<AmendmentPrecedentResponse>`.

All three endpoints `@RolesAllowed` consistent with parent resource. Scoped by
`tenantId` from `CurrentPrincipal`.

### Response Records (api module)

```java
record AePrecedentResponse(double score, String grade, String eventType,
    String safetyReviewOutcome, boolean dsmbEscalated, String problem, String outcome)

record DeviationPrecedentResponse(double score, String deviationType, String severity,
    String piDecision, String irbDecision, List<PlanStepResponse> steps,
    String problem, String outcome)

record PlanStepResponse(String bindingName, String capabilityName, String stepOutcome)

record AmendmentPrecedentResponse(double score, String proposedChange,
    String advisorOutcome, String outcome)
```

## Explore Page Panels

"Past Similar Cases" section on AE, deviation, and amendment explore pages.
Collapsible sub-table triggered by row selection, showing precedent rows sorted
by score. Uses `casehub-pages` DSL — `lookup()` against the precedent endpoint.

**Pre-implementation check:** verify latest blocks-ui and pages versions before
building explore panels. Check for any DSL API changes since the last explore
page work (issue #101).

## Maven Dependencies

Add to `runtime/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-neocortex-memory-cbr-inmem</artifactId>
</dependency>
```

`casehub-neocortex-memory-api` is already a transitive dependency. No Qdrant
dependency — Phase 1 runs against InMemoryCbrCaseMemoryStore. Qdrant is a
config-only swap when neocortex#69 ships.

## CDI Wiring

**Selected alternatives** — add to both production and test `application.properties`:
```properties
quarkus.arc.selected-alternatives=\
  ...,\
  io.casehub.neocortex.memory.cbr.inmem.InMemoryCbrCaseMemoryStore
```

**Jandex index** — add if `memory-cbr-inmem` doesn't ship one:
```properties
quarkus.index-dependency.neocortex-cbr-inmem.group-id=io.casehub
quarkus.index-dependency.neocortex-cbr-inmem.artifact-id=casehub-neocortex-memory-cbr-inmem
```

## SNAPSHOT Migration Notes

**Ledger API migration** (if touching ledger code during this work):
- `io.casehub.ledger.runtime.model.*` → `io.casehub.ledger.api.model.*`
- `io.casehub.ledger.runtime.model.supplement.*` → `io.casehub.ledger.api.model.supplement.*`
- `io.casehub.ledger.runtime.repository.LedgerEntryRepository` → `io.casehub.ledger.api.spi.LedgerEntryRepository`
- Runtime-specific classes (config, services, concrete repositories) unchanged.

**casehub-work API** — `WorkItemCreateRequest.category()` removed; use `.types(List.of(...))`.

## Testing Strategy

| Level | What | How |
|---|---|---|
| Unit | Feature extraction logic | Plain JUnit — given entity fields, assert features map correct |
| Unit | Plan trace construction | Plain JUnit — given case context, assert PlanTrace list correct |
| Unit | Schema registration | Mock store, verify `registerSchema` calls |
| Integration | Writer → store round-trip | `@QuarkusTest` — fire event, assert case in in-memory store |
| Integration | REST precedent endpoints | `@QuarkusTest` — pre-populate store, GET, assert scored results |
| Integration | End-to-end | Extend `ThreeSiteShowcaseTest` — after AE escalation, call `/precedents` |

Writers called directly in integration tests (same pattern as
`SafetyOfficerNotificationIntegrationTest`), avoiding Awaitility for CDI event delivery.

**CDI wiring smoke test:** One `@QuarkusTest` per writer that fires the actual CDI
event via `Event.fireAsync()` and asserts the CBR case appears in the in-memory store.
This catches misconfigured `@ObservesAsync` annotations, wrong event types, or bean
scope issues that direct-call tests would miss.

## New Files Summary

| Module | File | Purpose |
|---|---|---|
| api | `ClinicalCbrDomains` | Three `MemoryDomain` constants |
| api | `AePrecedentResponse` | REST response record |
| api | `DeviationPrecedentResponse` + `PlanStepResponse` | REST response records |
| api | `AmendmentPrecedentResponse` | REST response record |
| api | `ProtocolAmendmentResolvedEvent` | New CDI event |
| runtime | `ClinicalCbrSchemaInitializer` | `@Startup` schema registration |
| runtime | `ClinicalCbrService` | Thin wrapper — store + retrieveSimilar |
| runtime | `AeResolutionCbrWriter` | `@ObservesAsync` → FeatureVectorCbrCase |
| runtime | `DeviationResolutionCbrWriter` | `@ObservesAsync` → PlanCbrCase |
| runtime | `AmendmentResolutionCbrWriter` | `@ObservesAsync` → TextualCbrCase |
| runtime | Precedent methods on existing resources | 3 GET endpoints |
| webui | Explore page panels | "Past Similar Cases" sub-tables |
| test | Unit + integration tests | Per testing strategy above |

## Out of Scope

- Qdrant/vector backend (neocortex#69)
- LLM-powered similarity (semantic text embeddings)
- CBR-informed agent routing (clinical#78)
- Plan trace from engine cases (upgrade to PlanCbrCase for AE — future phase)
