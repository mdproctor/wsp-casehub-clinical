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

Processes each `PlanTrace` step from the retrieved case and assigns an `AdaptationAction` + adjusted priority + reason. Four rules evaluated in order:

**Rule 1 — Outcome-based suppression.** Step `stepOutcome` was `FAILED` or `TERMINATED` → `SUPPRESSED`, priority 0. Reason: "Step failed in past similar case."

**Rule 2 — Outcome-based boost.** Step `stepOutcome` was `COMPLETED` → `BOOSTED`, priority +10. Reason: "Step succeeded in past similar case (similarity: X.XX)."

**Rule 3 — Grade escalation boost.** Current AE grade > retrieved case's grade → all safety-related steps (`safety-monitoring`, `data-safety-monitoring`) get additional +5 priority regardless of outcome. Reason: "Higher severity than precedent — elevated urgency."

**Rule 4 — SUSAR addition.** Current AE has `unexpected=true` AND `suspected=true` but retrieved case did not → ADD a synthetic `susar-oversight` step with priority 20. Reason: "Current AE meets SUSAR criteria — not present in precedent case."

Steps matching none of the above → `RETAINED` with original priority.

Priority values are relative ordering hints for advisory display, not engine execution order — case bindings still fire based on their `contextChange` filters.

Safety-related capabilities for Rule 3: `safety-monitoring`, `data-safety-monitoring`.

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

### AeEscalationPlanRetriever

**Package:** `io.casehub.clinical.cbr`
**CDI:** `@ApplicationScoped`

Orchestrates retrieval + adaptation:

1. Build query features via `AeCbrFeatureBuilder.buildQueryFeatures()` (existing — grade, eventType, trialPhase, unexpected, suspected, treatmentArm, priorAeCount)
2. Call `ClinicalCbrService.retrieveWithAudit()` — top-5 similar `PlanCbrCase` records with audit trail
3. If no results → return `EscalationPlanRecommendation.none()`
4. Take top-scoring case, call `PlanAdapter.adapt("clinical-ae", topCase, currentFeatures)` → `AdaptedPlan`
5. Build `EscalationPlanRecommendation` from adapted plan + retrieval metadata

Error handling: retrieval failures are caught and logged — never block escalation case creation. If CBR is unavailable, the case starts without a recommendation.

### Integration in AeEscalationCaseService

In `prepareAndMarkRequested()`, after building the existing context (patientContext, siteContext):

```java
EscalationPlanRecommendation plan = planRetriever.retrieve(ae, enrollment, trial, priorAeCount);
if (plan.hasRecommendation()) {
    ctx.put("escalationPlanRecommendation", plan.toContextMap());
}
```

Same pattern as existing `patientContext` and `siteContext` injection.

### EscalationPlanResource

**Endpoint:** `GET /api/adverse-events/{aeId}/escalation-plans`
**Security:** `@RolesAllowed({SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR})`

Returns `EscalationPlanRecommendation` as JSON — queryable at any point during the AE lifecycle.

- Looks up AE, enrollment, trial, prior AE count
- Calls `AeEscalationPlanRetriever.retrieve()` — same path as injection flow
- 404 if AE not found; 200 with `retrievedCaseCount: 0` if no similar cases

Response shape:

```json
{
  "retrievedCaseCount": 3,
  "topSimilarityScore": 0.87,
  "traceId": "uuid",
  "explanation": "3 similar past cases found...",
  "steps": [
    { "bindingName": "safety-review", "capability": "safety-monitoring",
      "action": "BOOSTED", "priority": 15,
      "reason": "Step succeeded in past similar case (similarity: 0.87)" },
    { "bindingName": "dsmb-escalation", "capability": "data-safety-monitoring",
      "action": "RETAINED", "priority": 5, "reason": null },
    { "bindingName": "susar-oversight", "capability": "safety-monitoring",
      "action": "ADDED", "priority": 20,
      "reason": "Current AE meets SUSAR criteria — not present in precedent case" }
  ]
}
```

## Testing

### Unit tests (pure logic, no CDI)

| Test class | Covers |
|------------|--------|
| `ClinicalPlanAdapterTest` | Each adaptation rule independently: outcome suppression, outcome boost, grade escalation boost, SUSAR addition, passthrough for non-AE case types, combined rules on same plan |
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
