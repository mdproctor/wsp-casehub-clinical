# CBR Phase 5 — Learned Escalation Plans for AE Response

**Issue:** casehubio/clinical#118
**Epic:** casehubio/clinical#115 (CBR roadmap)
**Depends on:** #117 (Phase 4 — outcome recording + retrieval audit trail) ✅ closed
**Date:** 2026-07-19

## Problem

When a new adverse event triggers escalation, the system starts a fresh case with no knowledge of what worked in the past. Phase 4 already stores `PlanCbrCase` records with `PlanTrace` steps on case completion (RETAIN) and retrieves similar cases with an audit trail (RETRIEVE). What's missing is the REUSE step — adapting past plans for the current context and surfacing them as advisory recommendations.

## Solution

Three new components implement the CBR Reuse step for clinical AE escalation:

1. **`ClinicalPlanAdapter`** — `PlanAdapter` SPI implementation with four clinical adaptation rules
2. **`AeEscalationPlanRetriever`** — orchestrates retrieval + adaptation, returns `EscalationPlanRecommendation`
3. **`EscalationPlanResource`** — REST endpoint for on-demand plan queries

Integration point: `AeEscalationCaseService.prepareAndMarkRequested()` injects the adapted plan summary into the case context alongside existing `patientContext` and `siteContext`.

## Components

### ClinicalPlanAdapter

**Package:** `io.casehub.clinical.cbr`
**CDI:** `@ApplicationScoped` — displaces `NoOpPlanAdapter @DefaultBean` by being a concrete non-`@DefaultBean` bean. No `selected-alternatives` needed.

**Adaptation tracking:** `TrackingPlanAdapter` (neocortex `@Decorator`, activated by `casehub.cbr.adaptation-tracking.enabled=true`) automatically wraps this adapter, recording `AdaptationTrace` events via CDI for FDA audit compliance. No code changes needed — the decorator is transparent to `ClinicalPlanAdapter`.

Processes each `PlanTrace` step from the retrieved case and assigns an `AdaptationAction` + priority + reason. All rule conditions extract values from the feature maps passed to `PlanAdapter.adapt()` — the adapter has no access to domain entities (`AdverseEvent`, etc.), only to `currentFeatures` (`Map<String, FeatureValue>`) and `retrieved.cbrCase().features()`. Grade values are `NumberVal` (stored as `ae.grade.ordinal() + 1`); `unexpected`/`suspected` are `StringVal` (stored as `String.valueOf(boolean)`, producing `"true"`/`"false"`). If a required feature key is absent from either map, the rule that depends on it is skipped.

Four rules evaluated in order:

**Rule 1 — Outcome-based suppression.** Step `stepOutcome` is `"FAILED"` or `"TERMINATED"` → `SUPPRESSED`, priority 0. Reason: `"Step failed in past similar case."`

**Rule 2 — Outcome-based boost.** Step `stepOutcome` is `"COMPLETED"` → `BOOSTED`, priority 10. Reason: `"Step succeeded in past similar case (similarity: X.XX)."`

**Rule 3 — Grade escalation boost.** Applies only to non-SUPPRESSED steps (i.e., steps assigned BOOSTED or RETAINED by Rules 1–2). Condition: `currentFeatures.get("grade")` (`NumberVal`) > `retrieved.cbrCase().features().get("grade")` (`NumberVal`). When true, all non-SUPPRESSED steps with safety-related capabilities (`safety-monitoring`, `data-safety-monitoring`) get +5 priority. Reason: `"Higher severity than precedent — elevated urgency."` When Rule 3 applies on top of Rule 2, the reason is concatenated: `"Step succeeded in past similar case (similarity: X.XX). Higher severity than precedent — elevated urgency."` When Rule 3 applies to a RETAINED step, it sets the reason directly (replacing `null`).

**Rule 4 — SUSAR addition.** Condition: `currentFeatures.get("unexpected")` is `StringVal("true")` AND `currentFeatures.get("suspected")` is `StringVal("true")`, AND the retrieved case does not have both (`unexpected ≠ "true"` or `suspected ≠ "true"` in `retrieved.cbrCase().features()`). When true, ADD a synthetic step with all `AdaptedStep` fields:
- `bindingName`: `"susar-oversight"`
- `capabilityName`: `"susar-review"`
- `workerName`: `null` (synthetic step — no historical worker)
- `stepOutcome`: `null` (synthetic step — no historical outcome)
- `priority`: `20`
- `parameters`: `Map.of()` (empty)
- `action`: `ADDED`
- `reason`: `"Current AE meets SUSAR criteria — not present in precedent case."`

Steps matching none of the above → `RETAINED`, priority 0.

**Priority model:** All priority values are absolute assignments, not adjustments from stored baselines. Phase 4 (`ClinicalCaseOutcomeObserver.buildPlanTraces()`) stores all `PlanTrace` entries with priority 0 — the adapter assigns priorities from this zero baseline. Resulting ordering: SUPPRESSED/RETAINED (0) < safety-boosted-only (5) < BOOSTED (10) < BOOSTED + safety-boost (15) < SUSAR (20). These are advisory ordering hints for display, not engine execution order — case bindings fire based on their `contextChange` filters.

**Current data scope:** `BINDING_CAPABILITY_MAP` currently stores only `safety-review` → `safety-monitoring` and `dsmb-escalation` → `data-safety-monitoring`. Rule 3's capability filter therefore applies to all stored steps today. The rule is expressed as a capability filter for forward compatibility — when `BINDING_CAPABILITY_MAP` is expanded to capture additional non-safety bindings, Rule 3 will correctly subset to safety-related steps only.

Non-AE case types pass through unchanged (return traces as RETAINED steps).

### EscalationPlanRecommendation

**Package:** `io.casehub.clinical.cbr`

```java
record EscalationPlanRecommendation(
    AdaptedPlan adaptedPlan,
    int retrievedCaseCount,
    double topSimilarityScore,
    String traceId,
    String explanation
)
```

- `none()` static factory: null adaptedPlan, 0 count — used when no similar cases exist
- `hasRecommendation()`: returns `adaptedPlan != null`
- `toContextMap()`: serialises into a JQ-navigable `Map<String, Object>` for engine case context injection
- `explanation`: the retrieval explanation from `AuditedRetrievalResult` (generated by `ExplanationRenderer` — covers similarity rationale and feature comparison). Adaptation decisions (why each step was boosted, suppressed, or added) are captured per-step in `AdaptedStep.reason`, not duplicated into this field. When `TrackingPlanAdapter` is enabled, full adaptation traces are recorded separately via `AdaptationTrace`.

### AeEscalationPlanRetriever

**Package:** `io.casehub.clinical.cbr`
**CDI:** `@ApplicationScoped`

**Dependencies:** `ClinicalCbrService`, `PlanAdapter`, `EntityResolver` (same internal pattern as `ClinicalCaseOutcomeObserver` — resolution chain: `ae.enrollmentId → enrollment → enrollment.siteId → site → site.trialId → trial`, plus `countPriorAes()`)

**Method signature:** `retrieve(AdverseEvent ae)` — the retriever encapsulates all entity resolution and feature building internally. Callers pass only the AE entity.

Orchestrates retrieval + adaptation:

1. **Resolve entities** from the `AdverseEvent`: enrollment, site, trial, priorAeCount via internal `EntityResolver` (same chain as `ClinicalCaseOutcomeObserver`)
2. **Build query features** via `AeCbrFeatureBuilder.buildQueryFeatures(ae, enrollment, trial, priorAeCount)` → `Map<String, Object>`
3. **Convert to FeatureValue map** via `FeatureValue.toFeatureMap(rawFeatures)` → `Map<String, FeatureValue>` (required by both `CbrQuery` and `PlanAdapter.adapt()`)
4. **Build `CbrQuery`:**
   ```java
   CbrQuery.of(ae.tenantId, ClinicalCbrDomains.AE, Path.root(), "clinical-ae", featureMap, topK)
           .withMinSimilarity(minSimilarity)
   ```
   Configuration properties:
   - `casehub.clinical.cbr.escalation-plan.top-k` — default `5`
   - `casehub.clinical.cbr.escalation-plan.min-similarity` — default `0.4`
5. **Call `ClinicalCbrService.retrieveWithAudit(query, PlanCbrCase.class, ae.id, "system:ae-escalation")`** — returns `AuditedRetrievalResult<PlanCbrCase>` with cases, traceId, and explanation
6. If no results → return `EscalationPlanRecommendation.none()`
7. Take top-scoring case, call `PlanAdapter.adapt("clinical-ae", topCase, featureMap)` → `AdaptedPlan`
8. **Build `EscalationPlanRecommendation`** from adapted plan + retrieval metadata (count, top score, traceId, explanation)

Error handling: retrieval or adaptation failures are caught and logged — never block escalation case creation. If CBR is unavailable or adaptation fails, the case starts without a recommendation.

### Integration in AeEscalationCaseService

In `prepareAndMarkRequested()`, after building the existing context (patientContext, siteContext):

```java
EscalationPlanRecommendation plan = planRetriever.retrieve(ae);
if (plan.hasRecommendation()) {
    ctx.put("escalationPlanRecommendation", plan.toContextMap());
}
```

Same pattern as existing `patientContext` and `siteContext` injection. The retriever encapsulates entity resolution internally — `prepareAndMarkRequested()` passes only the `AdverseEvent` entity already loaded at line 59.

### EscalationPlanResource

**Endpoint:** `GET /api/adverse-events/{aeId}/escalation-plans`
**Security:** `@RolesAllowed({SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR})`

Returns `EscalationPlanRecommendation` as JSON — queryable at any point during the AE lifecycle. Response uses `@JsonInclude(NON_NULL)` — null fields are omitted. The `adaptedPlan.steps` list is hoisted to a top-level `steps` array via a response DTO projection.

- Looks up AE by ID, calls `AeEscalationPlanRetriever.retrieve()` — same path as injection flow
- 404 if AE not found; 200 with `retrievedCaseCount: 0` if no similar cases

Response shape:

```json
{
  "retrievedCaseCount": 3,
  "topSimilarityScore": 0.87,
  "traceId": "uuid",
  "explanation": "3 similar past cases found...",
  "steps": [
    { "bindingName": "safety-review", "capabilityName": "safety-monitoring",
      "workerName": "safety-monitor-worker", "stepOutcome": "COMPLETED",
      "action": "BOOSTED", "priority": 15,
      "parameters": {},
      "reason": "Step succeeded in past similar case (similarity: 0.87). Higher severity than precedent — elevated urgency." },
    { "bindingName": "dsmb-escalation", "capabilityName": "data-safety-monitoring",
      "workerName": "dsmb-escalation-worker", "stepOutcome": "COMPLETED",
      "action": "BOOSTED", "priority": 15,
      "parameters": {},
      "reason": "Step succeeded in past similar case (similarity: 0.87). Higher severity than precedent — elevated urgency." },
    { "bindingName": "susar-oversight", "capabilityName": "susar-review",
      "action": "ADDED", "priority": 20,
      "parameters": {},
      "reason": "Current AE meets SUSAR criteria — not present in precedent case." }
  ]
}
```

## Testing

### Unit tests (pure logic, no CDI)

| Test class | Covers |
|------------|--------|
| `ClinicalPlanAdapterTest` | Each adaptation rule independently: outcome suppression, outcome boost, grade escalation boost, SUSAR addition, passthrough for non-AE case types, combined rules on same plan, Rule 3 exclusion for SUPPRESSED steps (priority stays 0), reason concatenation when Rules 2+3 combine, missing feature graceful skip |
| `AeEscalationPlanRetrieverTest` | Mock ClinicalCbrService + PlanAdapter — retrieval→adaptation→recommendation flow, empty results, exception resilience |
| `EscalationPlanRecommendationTest` | `none()` factory, `hasRecommendation()`, `toContextMap()` serialisation |

### Integration tests (`@QuarkusTest`)

| Test class | Covers |
|------------|--------|
| `AeEscalationPlanRetrieverIntegrationTest` | Full round-trip: store PlanCbrCase via ClinicalCaseOutcomeObserver, then retrieve+adapt via AeEscalationPlanRetriever. Uses `InMemoryCbrCaseMemoryStore`. |
| `EscalationPlanResourceTest` | REST: 200 with recommendation, 200 empty, 404 not found, role-based access |

### Test isolation patterns

- GE-20260716-986cd1: Use `.withNotBefore(Instant.now())` on `CbrQuery` to isolate from cross-test store bleed
- GE-20260712-626e51: Use `Instance<PlanItemStore>` if direct injection fails in `@QuarkusTest`

## Boundaries

- No Flyway migrations — no new persistent entities
- No engine YAML changes — adapted plans are advisory context, not execution directives
- No neocortex changes — consuming existing SPIs (`PlanAdapter`, `PlanCbrCase`, `AdaptedPlan`)
- `ClinicalPlanAdapter` displaces `NoOpPlanAdapter` for ALL case types in clinical — non-AE case types pass through unchanged (RETAINED)
