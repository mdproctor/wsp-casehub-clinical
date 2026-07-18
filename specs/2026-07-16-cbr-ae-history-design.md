# CBR Over Adverse Event History — #78 Design Spec

**Issue:** casehubio/clinical#78
**Branch:** issue-78-cbr-ae-history
**Date:** 2026-07-16

## Context

Phase 1 (#116) delivered baseline CBR wiring: three CDI event writers (AE, deviation, amendment), feature schemas, `ClinicalCbrService`, REST precedent endpoints, and a webui panel. All tests pass. The InMemoryCbrCaseMemoryStore already uses `CbrSimilarityScorer` for weighted feature-based scoring.

neocortex#68 (CBR production readiness) is now closed. All four foundation dependencies are closed: engine#477 (CaseOutcomeObserver SPI), engine#478 (CaseRetriever integration), neural-text#20 (CaseRetriever contract), platform#87 (CbrCaseEntry type).

This issue completes #78's original scope: enriched AE features, CaseOutcomeObserver integration, and routing enrichment via plan traces.

## What Changes

### 1. AE Feature Enrichment

Add two features from #78's original spec to the AE schema and writer:

| Feature | Source | Type | Schema |
|---------|--------|------|--------|
| `treatmentArm` | `PatientEnrollment.treatmentArm` | String | `FeatureField.categorical("treatmentArm")` |
| `priorAeCount` | Count of AEs for same enrollmentId | String bucket | `FeatureField.categorical("priorAeCount")` |

`priorAeCount` buckets: `NONE` (0), `ONE` (1), `MULTIPLE` (2+). Bucketed rather than raw count because CBR categorical similarity is exact-match — the clinical distinction is between "first event", "second event", and "recurrent pattern", not between 3 vs 4.

`treatmentArm` is a new field on `PatientEnrollment`. It's nullable (not all trials use treatment arms). When null, the feature value is `"UNASSIGNED"`.

**Files modified:**
- `PatientEnrollment` — add `treatmentArm` field
- `ClinicalCbrSchemaInitializer.aeSchema()` — add two fields (11 total)
- `AeCbrFeatureBuilder` — shared feature extraction (treatmentArm from enrollment, priorAeCount bucketed)
- `AePrecedentResponse` — add `treatmentArm`, `priorAeCount`, and `List<PlanStepResponse> steps` fields (architectural consistency with `DeviationPrecedentResponse`)
- `TrialDashboardResource` — extend query feature map with `treatmentArm` and `priorAeCount`; update `mapToAeResponse()` signature from `ScoredCbrCase<FeatureVectorCbrCase>` to `ScoredCbrCase<PlanCbrCase>` and map plan traces to `PlanStepResponse`
- Flyway migration — add `treatment_arm` column to `patient_enrollment`

### 2. Clinical-Specific Query Weights

Replace uniform default weights with clinically meaningful weights. Severity and organ system dominate AE similarity; boolean flags and outcome features contribute less.

```java
CbrQuery.of(tenantId, ClinicalCbrDomains.AE, "clinical-ae", features, 10)
    .withMinSimilarity(0.3)
    .withVectorWeight(0.0)
    // Problem features — weighted by clinical relevance
    .withWeight("grade", 3.0)
    .withWeight("eventType", 2.5)
    .withWeight("treatmentArm", 1.5)
    .withWeight("unexpected", 1.5)
    .withWeight("priorAeCount", 1.0)
    .withWeight("trialPhase", 1.0)
    .withWeight("suspected", 1.0)
    // Outcome features — zero weight (search on problem, display in results)
    .withWeight("safetyReviewOutcome", 0.0)
    .withWeight("dsmbEscalated", 0.0)
    .withWeight("indReportFiled", 0.0)
    .withWeight("susarOversight", 0.0);
```

**Rationale:**
- `grade` (3.0): CTCAE severity is the primary driver of response pathway. A Grade 3 hepatotoxicity is far more informative to another Grade 3 hepatotoxicity than a Grade 1 headache.
- `eventType` (2.5): organ system determines clinical response protocol. Same organ class = similar management.
- `treatmentArm` (1.5): same treatment group implies similar drug exposure profile.
- `unexpected` (1.5): unexpected events require different reporting and escalation.
- Others at 1.0: relevant but not dominant.
- Outcome features at 0.0: these are solution data, not problem data. Shown in precedent results but not used for retrieval.

**Files modified:**
- `TrialDashboardResource` — update AE precedent query construction: extend query feature map with `treatmentArm` (from enrollment traversal) and `priorAeCount` (from count query, bucketed); change retrieval type from `FeatureVectorCbrCase.class` to `PlanCbrCase.class`

### 3. AE CBR Writer Consolidation

Delete `AeResolutionCbrWriter`. Its CBR storage responsibility moves entirely to `ClinicalCaseOutcomeObserver` (§4), which becomes the sole CBR writer for AE cases.

**Why:** `AeResolutionCbrWriter` fires via the async CDI chain `CaseLifecycleEvent` → `AeEscalationListener` → `AeEscalationCompletedEvent` — at least two async hops after the engine begins case close processing. `CaseOutcomeObserver.onOutcome()` is a synchronous SPI call made by the engine during case close (called on a Vert.x worker thread with `blocking = true`). If both store CBR cases, the writer's later `storeIdempotent()` (erase-before-store) would overwrite the observer's enriched case, erasing plan trace data. A single writer at case close has all the data and no race condition.

**Why PlanCbrCase:** `CbrAgentRoutingStrategy.analyseExperiences()` reads `ExperiencePlanStep` entries from `PlanCbrCase.planTrace()`. `FeatureVectorCbrCase` has no plan trace — the routing strategy would ignore it entirely.

**Files deleted:**
- `AeResolutionCbrWriter` — CBR responsibility moves to observer
- `AeResolutionCbrWriterTest` — replaced by observer tests
- `AeResolutionCbrWriterIntegrationTest` — replaced by observer integration tests

**Files created:**
- `AeCbrFeatureBuilder` — shared feature extraction logic used by the observer (§4) and the precedent query endpoint (§2)

### 4. ClinicalCaseOutcomeObserver

New class implementing `CaseOutcomeObserver` (engine#477 SPI), `@ApplicationScoped`.

**Multi-observer contract:** The engine's `CaseStatusChangedHandler` injects `Instance<CaseOutcomeObserver>` and iterates all discovered implementations (`for (CaseOutcomeObserver observer : outcomeObservers)`). The engine's own `CbrCaseRetainObserver` (`@ApplicationScoped`) also implements this SPI — both observers fire for every case close. No CDI ambiguity (`Instance<>` iterates, not single-inject). The two observers do not interfere: `CbrCaseRetainObserver` returns early for AE cases because the ae-escalation case definition has no `CbrConfig` (`if (config == null) return;`). Each observer handles exceptions independently — the engine catches and logs failures without propagating.

**Transaction strategy:** The observer runs on the engine's Vert.x worker thread (`blocking = true`). The SPI javadoc explicitly permits "blocking work directly, including JPA writes and @Transactional operations." The AE storage path (§4A) accesses Panache Active Record entities (`AdverseEvent`, `PatientEnrollment`) on the default persistence unit, which requires an active Hibernate session. The `onOutcome()` method itself is annotated `@Transactional` — not an internal method, because CDI interceptors do not fire on self-calls (`this.method()` bypasses the proxy). For non-AE cases the transaction boundary is harmless (only `CbrCaseMemoryStore.recordOutcome()` runs, which manages its own persistence context). `PlanItemStore` and `CbrCaseMemoryStore` manage their own persistence contexts independently.

Two responsibilities:

**A. AE CBR case storage (replaces AeResolutionCbrWriter):**

When an AE escalation case closes, the observer builds and stores the complete `PlanCbrCase` in a single write — features and plan traces together, no second-phase enrichment.

Steps:
1. Identifies the case as AE by checking `event.caseFileSnapshot().get("aeId") != null` — same discriminator pattern as `AeEscalationListener` (checks for domain key presence rather than matching `caseType` string). The `caseType()` value is `"ae-escalation"` (from the YAML `name` field) but snapshot-key discrimination is more robust against YAML renames.
2. Extracts `aeId` from the snapshot, looks up `AdverseEvent.findById(aeId)` for entity-specific fields (`eventType`, `suspected`, `regulatorySubmissionStatus`, `susarOversightStatus`, `enrollmentId`).
3. Resolves `safetyReviewOutcome` and `dsmbEscalated` from the snapshot — these are worker output values written by the YAML's outputMapping (`{ safetyReview: . }` for the safety-review binding, `{ dsmbEscalation: . }` for the dsmb-escalation binding).
4. Builds clinical features via `AeCbrFeatureBuilder` — all 11 features including `treatmentArm` (from `PatientEnrollment` entity traversal) and `priorAeCount` (from count query, bucketed).
5. Queries `PlanItemStore.findByCaseId(event.caseId(), event.tenancyId())` for the execution trace. `PlanItemStore` is an SPI in `casehub-engine-common`; the engine's own `CbrCaseRetainObserver` uses this same API for plan trace construction. Each `PlanItemRecord` carries `bindingName()`, `executorName()` (worker identity), and `status()` (terminal outcome).
6. Filters to terminal plan items with non-null `executorName`, maps each to `PlanTrace(bindingName, capabilityName, executorName, outcomeString, priority, Map.of())`. The capability name comes from a hardcoded clinical binding→capability map:
   - `safety-review` → `safety-monitoring`
   - `dsmb-escalation` → `data-safety-monitoring`

   **Why hardcoded, not from case definition:** The engine's `CbrCaseRetainObserver.buildCapabilityNameMap()` resolves capability names via `binding.target() instanceof CapabilityTarget ct`. AE escalation bindings are `humanTask` (not `CapabilityTarget`), so the engine's map is empty for these cases. The clinical observer provides its own domain-specific mapping because these binding→capability associations are stable within the clinical domain and have no engine-level equivalent.
7. Builds `PlanCbrCase(problem, solution, outcomeLabel, 1.0, features, planTraces)`.
8. Stores via `ClinicalCbrService.storeIdempotent()` keyed by `aeId.toString()`.

**Why PlanItemStore, not caseFileSnapshot:** The `caseFileSnapshot` is the working-layer context (blackboard) containing domain values set by output mappings — it does NOT contain worker assignment information. The `CaseOutcomeEvent.metadata()` is documented as "currently empty, reserved for future use." Worker→binding assignments are engine-internal execution records stored in `PlanItemStore`. This is the proven extraction path — the engine's own `CbrCaseRetainObserver` uses it identically.

**B. Outcome recording (all case types):**

For ALL case types (AE, deviation, amendment), calls `CbrCaseMemoryStore.recordOutcome()` to adjust the stored case's confidence. This is the CBR Revise step — successful outcomes reinforce the case's confidence; poor outcomes reduce it.

Entity ID resolution from snapshot keys per case type:
- AE escalation: `caseFileSnapshot.get("aeId")` → entityId
- Deviation review: `caseFileSnapshot.get("deviationId")` → entityId
- Amendment review: `caseFileSnapshot.get("amendmentId")` → entityId

If the expected key is absent, the observer logs at debug level and skips outcome recording for that case (unknown case type — not a clinical case). Debug rather than warn because the observer fires for all engine case types, and non-clinical cases are expected in shared deployments.

The outcome mapping:
- `outcomeLabel` is the engine's terminal status: `"COMPLETED"`, `"FAULTED"`, `"CANCELLED"`
- Map to `CbrOutcome` — COMPLETED → high successRate, FAULTED/CANCELLED → low

**Files created:**
- `ClinicalCaseOutcomeObserver` — implements `CaseOutcomeObserver`, `@ApplicationScoped`

### 5. Routing Enrichment

AE CBR cases are stored as `PlanCbrCase` with plan traces including explicit capability names (§4A step 6). This provides two levels of value:

**Immediate: precedent display.** The `AePrecedentResponse` (§1) includes `List<PlanStepResponse> steps`, showing clinicians which safety officer handled past similar events and their outcomes. This is the primary user-facing benefit — same architectural pattern as deviation precedents.

**Future: automated routing.** The engine's `CbrAgentRoutingStrategy.analyseExperiences()` scores workers by matching `capabilityName` from plan traces against the current binding's capability. For `CapabilityTarget` bindings, this works end-to-end. AE escalation uses `humanTask` bindings, which the engine's routing pipeline does not currently route via `CbrAgentRoutingStrategy`. The plan traces are stored with the correct structure (binding name, capability name, worker identity, outcome) so that when the engine adds humanTask routing enrichment, AE precedent data is ready. This is tracked as a future engine enhancement (casehubio/engine#741).

**Deviation/amendment writers unchanged:** Their "workers" are PIs and IRB committees (domain entities, not engine-assigned agents). The deviation writer already stores `PlanCbrCase` with complete plan traces from domain events.

### 6. Existing Code Unchanged

The following Phase 1 code is NOT modified (already complete):
- `ClinicalCbrDomains` — AE, DEVIATION, AMENDMENT domains
- `ClinicalCbrService` — erase-before-store wrapper
- `DeviationResolutionCbrWriter` — complete with PlanCbrCase + plan traces
- `AmendmentResolutionCbrWriter` — complete with TextualCbrCase
- `DeviationPrecedentResponse`, `AmendmentPrecedentResponse` — unchanged
- `ClinicalCbrSchemaInitializer` deviation/amendment schemas — unchanged
- Webui precedent panel — unchanged (consumes REST API, format unchanged)
- `CbrCdiWiringTest` — unchanged

## Data Flow

```
AE Reported
  → Engine case starts (ae-escalation)
  → Safety monitoring worker assigned by engine
  → Workers complete tasks, outcomes written to case context
  → Engine case closes (COMPLETED)
  → ClinicalCaseOutcomeObserver.onOutcome() fires (synchronous SPI call)
      → Identifies AE case via aeId in snapshot
      → Loads AdverseEvent entity for clinical features
      → Queries PlanItemStore for execution trace (binding→worker mappings)
      → Builds PlanCbrCase(features, planTrace=[safety-review→officer-X])
      → storeIdempotent() stores complete case in one write
      → recordOutcome() adjusts confidence
      ↓
  → Next AE of same type reported
  → Clinician queries precedent endpoint
  → TrialDashboardResource retrieves similar PlanCbrCase entries
  → AePrecedentResponse includes plan steps (which officer, what outcome)
  → Clinician sees prior handling patterns for similar AEs
```

## Testing Strategy

### Unit Tests
- `AeCbrFeatureBuilderTest` — feature extraction with all field combinations (null treatmentArm, zero prior AEs, null eventType, etc.)
- `ClinicalCaseOutcomeObserverTest` — mock `CbrCaseMemoryStore` and `PlanItemStore`; verify: AE case identification via aeId snapshot key, feature extraction via builder, plan trace construction from PlanItemRecords, outcome recording per case type, non-AE case types skip CBR storage but still record outcome

### Integration Tests
- `ClinicalCaseOutcomeObserverIntegrationTest` — @QuarkusTest: create AdverseEvent entity, fire CaseOutcomeEvent with aeId in snapshot, verify PlanCbrCase stored with correct features and plan traces, verify outcome recorded
- `PrecedentEndpointTest` — update: verify new fields in AePrecedentResponse (treatmentArm, priorAeCount, steps), verify weighted query returns ranked results, verify retrieval uses PlanCbrCase.class

### Boundary/Edge Cases
- AE with null treatmentArm → feature value "UNASSIGNED"
- AE with no prior AEs → priorAeCount = "NONE"
- CaseOutcomeObserver for non-AE case type (deviation, amendment) → only recordOutcome(), no AE CBR storage
- CaseOutcomeObserver for unknown case type (no recognized snapshot key) → log warning, skip
- PlanItemStore returns no terminal plan items with executorName → PlanCbrCase with empty plan trace (case still stored with features)
- PlanItemRecord with bindingName not in clinical capability map → plan item skipped (only mapped bindings produce plan traces)
- AdverseEvent entity not found for aeId in snapshot → log warning, skip CBR storage
- CaseOutcomeEvent with outcomeLabel "FAULTED" → low confidence in recordOutcome()

## Migration

One Flyway migration: add `treatment_arm VARCHAR(255)` column to `patient_enrollment` table (default datasource).

Version: next available in V100–V999 range for default datasource.

## Dependencies

All closed:
- engine#477 CaseOutcomeObserver SPI ✅
- engine#478 CaseRetriever integration ✅
- neural-text#20 CaseRetriever contract ✅
- platform#87 CbrCaseEntry type ✅
- neocortex#68 CBR production readiness ✅

## Out of Scope

- Phase 2 (#82): weighted similarity scoring beyond clinical weights (neocortex owns)
- Phase 3 (#83): semantic/hybrid retrieval with embeddings
- Phase 4 (#117): outcome audit trail + retrieval traceability
- Phase 5 (#118): learned escalation plans from precedent
- Configurable weights (hardcoded is sufficient for pre-release)
- Non-escalated AE CBR cases (Grade 1-2 without engine cases)
- Engine humanTask routing enrichment via CBR plan traces (casehubio/engine#741 — plan traces are stored and ready)
