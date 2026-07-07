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
| `eventType` | Categorical | e.g. "NAUSEA", "HEPATOTOXICITY" |
| `trialPhase` | Categorical | PHASE_I through PHASE_IV |
| `unexpected` | Categorical | boolean |
| `suspected` | Categorical | boolean |

**Outcome features** (how it was resolved):

| Feature | Type | Notes |
|---|---|---|
| `escalationOutcome` | Categorical | From AeEscalationStatus: COMPLETED, FAILED |
| `safetyReviewOutcome` | Categorical | Extracted from case context |
| `dsmbEscalated` | Categorical | boolean |
| `indReportFiled` | Categorical | boolean — was IND expedited report triggered |
| `susarOversight` | Categorical | boolean — was SUSAR oversight case triggered |

Only Grade 3+ AEs get engine cases (and therefore `AeEscalationCompletedEvent`).
Grade 1-2 AEs follow a fixed path with no meaningful resolution variation — no CBR
case stored.

### 2. ClinicalDeviationCbrCase — PlanCbrCase

Stored on `ProtocolDeviationResolvedEvent`. `caseType = "clinical-deviation"`.

**Problem features:**

| Feature | Type | Notes |
|---|---|---|
| `deviationType` | Categorical | e.g. "CONSENT_VIOLATION" |
| `severity` | Categorical | MINOR, MAJOR, CRITICAL |
| `escalationRequirement` | Categorical | NONE, IRB, SPONSOR, BOTH |

**Outcome features:**

| Feature | Type | Notes |
|---|---|---|
| `piDecision` | Categorical | APPROVED, REJECTED, EXPIRED |
| `irbDecision` | Categorical | APPROVED, REJECTED, DEFERRED, EXPIRED, or null |

**Plan trace** — from the deviation-review engine case bindings:
- `pi-oversight` binding → step outcome (DONE/DECLINE/EXPIRED)
- `irb-committee` binding → step outcome (if CRITICAL path fired)

If engine case ID is null (deviation expired before case started), stored with
empty plan trace.

### 3. ClinicalAmendmentCbrCase — TextualCbrCase

Stored on `ProtocolAmendmentResolvedEvent` (new event). `caseType = "clinical-amendment"`.

No structured features. Pure text:
- `problem` = the `proposedChange` text from the amendment
- `solution` = advisor recommendation (PROCEED/REJECT + rationale if available)
- `outcome` = terminal status (APPROVED/REJECTED)

## CBR Writers

Three CDI event observers in `io.casehub.clinical.cbr`:

### AeResolutionCbrWriter

`@ObservesAsync AeEscalationCompletedEvent`. Loads the `AdverseEvent` entity to
get eventType, suspected, regulatorySubmissionStatus, susarOversightStatus. Builds
the feature map (problem + outcome features) and stores via `ClinicalCbrService`.

`entityId = aeId.toString()`, `domain = MemoryDomain("clinical-ae")`.

### DeviationResolutionCbrWriter

`@ObservesAsync ProtocolDeviationResolvedEvent`. Loads `ProtocolDeviation` entity
for engineCaseId. Queries engine case context for binding completion states to
construct `PlanTrace` records. If engineCaseId is null, empty trace.

`entityId = deviationId.toString()`, `domain = MemoryDomain("clinical-deviation")`.

### AmendmentResolutionCbrWriter

`@ObservesAsync ProtocolAmendmentResolvedEvent`. Reads proposedChange and terminal
status from the event. Stores a `TextualCbrCase`.

`entityId = amendmentId.toString()`, `domain = MemoryDomain("clinical-amendment")`.

**New event required:** `ProtocolAmendmentResolvedEvent` — fired by
`ProtocolAmendmentStatusUpdater` when status reaches APPROVED or REJECTED. One-line
`fireAsync()` addition, consistent with AE and deviation patterns.

## Shared Infrastructure

### ClinicalCbrService

Thin wrapper around `CbrCaseMemoryStore`. Injected by all three writers for `store()`
and by the REST endpoints for `retrieveSimilar()`. Handles tenant scoping.

### ClinicalCbrSchemaInitializer

`@ApplicationScoped @Startup`. Registers three `CbrFeatureSchema` instances (one per
case type) at boot. The amendment schema is empty (TextualCbrCase has no features).

### ClinicalCbrDomains

Constants class with three `MemoryDomain` instances:
- `AE = MemoryDomain("clinical-ae")`
- `DEVIATION = MemoryDomain("clinical-deviation")`
- `AMENDMENT = MemoryDomain("clinical-amendment")`

## REST Endpoints

### GET /trials/{trialId}/adverse-events/{aeId}/precedents

Builds `CbrQuery` from current AE features. Problem features weighted 1.0, outcome
features weighted 0.0 (find AEs that LOOKED like this one). `topK = 10`,
`minSimilarity = 0.3`. Returns `List<AePrecedentResponse>`.

### GET /trials/{trialId}/deviations/{devId}/precedents

Builds query from current deviation features. Same weighting (match on problem).
Returns `List<DeviationPrecedentResponse>` including `List<PlanStepResponse>`.

### GET /trials/{trialId}/amendments/{amendmentId}/precedents

Builds query with `problem = amendment.proposedChange` for text matching.
Returns `List<AmendmentPrecedentResponse>`.

All three endpoints `@RolesAllowed` consistent with parent resource. Scoped by
`tenantId` from `CurrentPrincipal`.

### Response Records (api module)

```java
record AePrecedentResponse(double score, String grade, String eventType,
    String escalationOutcome, boolean dsmbEscalated, String problem, String outcome)

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
