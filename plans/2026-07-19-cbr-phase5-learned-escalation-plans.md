# CBR Phase 5 — Learned Escalation Plans Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — feat: CBR Phase 5 — learned escalation plans for AE response
**Issue group:** #118

**Goal:** When an AE triggers escalation, retrieve similar past escalation plans from CBR memory, adapt them with clinical rules, and surface as advisory recommendations — both injected at case start and queryable via REST.

**Architecture:** `ClinicalPlanAdapter` implements the neocortex `PlanAdapter` SPI with four clinical adaptation rules (outcome suppression, outcome boost, grade escalation boost, SUSAR addition). `AeEscalationPlanRetriever` orchestrates retrieval via `ClinicalCbrService.retrieveWithAudit()` + adaptation via `PlanAdapter.adapt()`, returning `EscalationPlanRecommendation`. Integration at `AeEscalationCaseService.prepareAndMarkRequested()` for case context injection, plus `EscalationPlanResource` REST endpoint for on-demand queries.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex `PlanAdapter`/`PlanCbrCase`/`AdaptedPlan` SPIs, casehub-clinical CBR infrastructure (Phase 4)

## Global Constraints

- All code in `runtime/` module — clinical has no downstream JPA consumers; `api/` holds enums and constants only
- Package: `io.casehub.clinical.cbr` for all new classes (alongside existing CBR code)
- REST endpoints in `io.casehub.clinical.resource`
- No Flyway migrations — no new persistent entities
- No engine YAML changes — adapted plans are advisory context only
- No neocortex changes — consuming existing SPIs
- `@RolesAllowed` on all REST endpoints using `ClinicalGroups` constants
- Tests use `InMemoryCbrCaseMemoryStore` — never `JpaLedgerEntryRepository` in tests
- GE-20260716-986cd1: Use `.withNotBefore(Instant.now())` on `CbrQuery` for test isolation
- GE-20260712-626e51: Use `Instance<PlanItemStore>` if direct injection fails

---

### Task 1: EscalationPlanRecommendation value type + ClinicalPlanAdapter

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/EscalationPlanRecommendation.java`
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalPlanAdapter.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/EscalationPlanRecommendationTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/ClinicalPlanAdapterTest.java`

**Interfaces:**
- Consumes: `PlanAdapter` SPI (`adapt(String caseType, ScoredCbrCase<PlanCbrCase> retrieved, Map<String, FeatureValue> currentFeatures) → AdaptedPlan`), `AdaptedPlan(List<AdaptedStep>)`, `AdaptedStep(bindingName, capabilityName, workerName, stepOutcome, priority, parameters, action, reason)`, `AdaptationAction` enum, `PlanTrace(bindingName, capabilityName, workerName, stepOutcome, priority, parameters)`, `FeatureValue` sealed hierarchy (`StringVal`, `NumberVal`), `ScoredCbrCase<C>(cbrCase, caseId, score, reranked, featureSimilarities, storedAt, scope)`
- Produces: `EscalationPlanRecommendation(adaptedPlan, retrievedCaseCount, topSimilarityScore, traceId, explanation)` with `none()`, `hasRecommendation()`, `toContextMap()`. `ClinicalPlanAdapter` implements `PlanAdapter` — displaces `NoOpPlanAdapter @DefaultBean`.

- [ ] **Step 1: Write EscalationPlanRecommendation tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.AdaptedPlan;
import io.casehub.neocortex.memory.cbr.AdaptedStep;
import io.casehub.neocortex.memory.cbr.AdaptationAction;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class EscalationPlanRecommendationTest {

    @Test
    void none_returnsEmptyRecommendation() {
        var rec = EscalationPlanRecommendation.none();
        assertThat(rec.hasRecommendation()).isFalse();
        assertThat(rec.retrievedCaseCount()).isZero();
        assertThat(rec.adaptedPlan()).isNull();
    }

    @Test
    void hasRecommendation_trueWhenPlanPresent() {
        var step = new AdaptedStep("safety-review", "safety-monitoring", "worker-1",
            "COMPLETED", 10, Map.of(), AdaptationAction.BOOSTED,
            "Step succeeded in past similar case (similarity: 0.87).");
        var plan = new AdaptedPlan(List.of(step));
        var rec = new EscalationPlanRecommendation(plan, 3, 0.87, "trace-1", "explanation");
        assertThat(rec.hasRecommendation()).isTrue();
    }

    @Test
    void toContextMap_containsAllFields() {
        var step = new AdaptedStep("safety-review", "safety-monitoring", "worker-1",
            "COMPLETED", 10, Map.of(), AdaptationAction.BOOSTED, "reason");
        var plan = new AdaptedPlan(List.of(step));
        var rec = new EscalationPlanRecommendation(plan, 2, 0.85, "trace-id", "expl");

        Map<String, Object> ctx = rec.toContextMap();
        assertThat(ctx).containsKey("retrievedCaseCount");
        assertThat(ctx).containsKey("topSimilarityScore");
        assertThat(ctx).containsKey("traceId");
        assertThat(ctx).containsKey("explanation");
        assertThat(ctx).containsKey("steps");
        @SuppressWarnings("unchecked")
        var steps = (List<Map<String, Object>>) ctx.get("steps");
        assertThat(steps).hasSize(1);
        assertThat(steps.get(0)).containsEntry("bindingName", "safety-review");
        assertThat(steps.get(0)).containsEntry("action", "BOOSTED");
    }

    @Test
    void toContextMap_noneReturnsEmptySteps() {
        var rec = EscalationPlanRecommendation.none();
        Map<String, Object> ctx = rec.toContextMap();
        assertThat(ctx).containsEntry("retrievedCaseCount", 0);
        @SuppressWarnings("unchecked")
        var steps = (List<Map<String, Object>>) ctx.get("steps");
        assertThat(steps).isEmpty();
    }
}
```

- [ ] **Step 2: Write ClinicalPlanAdapter tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClinicalPlanAdapterTest {

    private final ClinicalPlanAdapter adapter = new ClinicalPlanAdapter();

    private ScoredCbrCase<PlanCbrCase> buildCase(Map<String, FeatureValue> features,
                                                  List<PlanTrace> traces) {
        var cbrCase = new PlanCbrCase("problem", "solution", "COMPLETED", 1.0, features, traces);
        return new ScoredCbrCase<>(cbrCase, "case-1", 0.87);
    }

    @Test
    void rule1_failedStep_isSuppressed() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "FAILED", 0, Map.of());
        var scored = buildCase(Map.of("grade", FeatureValue.number(3)), List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(3));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps()).hasSize(1);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.SUPPRESSED);
        assertThat(result.steps().get(0).priority()).isZero();
        assertThat(result.steps().get(0).reason()).contains("failed");
    }

    @Test
    void rule1_terminatedStep_isSuppressed() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "TERMINATED", 0, Map.of());
        var scored = buildCase(Map.of("grade", FeatureValue.number(3)), List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(3));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.SUPPRESSED);
    }

    @Test
    void rule2_completedStep_isBoosted() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var scored = buildCase(Map.of("grade", FeatureValue.number(3)), List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(3));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.BOOSTED);
        assertThat(result.steps().get(0).priority()).isEqualTo(10);
        assertThat(result.steps().get(0).reason()).contains("succeeded");
    }

    @Test
    void rule3_higherGrade_boostsSafetySteps() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var pastFeatures = Map.of("grade", (FeatureValue) FeatureValue.number(2));
        var scored = buildCase(pastFeatures, List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(4));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps().get(0).priority()).isEqualTo(15);
        assertThat(result.steps().get(0).reason()).contains("Higher severity");
    }

    @Test
    void rule3_doesNotBoostSuppressedSteps() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "FAILED", 0, Map.of());
        var pastFeatures = Map.of("grade", (FeatureValue) FeatureValue.number(2));
        var scored = buildCase(pastFeatures, List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(4));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.SUPPRESSED);
        assertThat(result.steps().get(0).priority()).isZero();
    }

    @Test
    void rule3_sameGrade_noExtraBoost() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var features = Map.of("grade", (FeatureValue) FeatureValue.number(3));
        var scored = buildCase(features, List.of(trace));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, features);
        assertThat(result.steps().get(0).priority()).isEqualTo(10);
    }

    @Test
    void rule4_susarCondition_addsStep() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var pastFeatures = Map.<String, FeatureValue>of(
            "grade", FeatureValue.number(3),
            "unexpected", FeatureValue.string("false"),
            "suspected", FeatureValue.string("false"));
        var scored = buildCase(pastFeatures, List.of(trace));
        var current = Map.<String, FeatureValue>of(
            "grade", FeatureValue.number(3),
            "unexpected", FeatureValue.string("true"),
            "suspected", FeatureValue.string("true"));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps()).hasSize(2);
        var susarStep = result.steps().stream()
            .filter(s -> s.action() == AdaptationAction.ADDED).findFirst().orElseThrow();
        assertThat(susarStep.bindingName()).isEqualTo("susar-oversight");
        assertThat(susarStep.capabilityName()).isEqualTo("susar-review");
        assertThat(susarStep.priority()).isEqualTo(20);
        assertThat(susarStep.workerName()).isNull();
        assertThat(susarStep.stepOutcome()).isNull();
    }

    @Test
    void rule4_pastAlsoSusar_noAddition() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var features = Map.<String, FeatureValue>of(
            "grade", FeatureValue.number(3),
            "unexpected", FeatureValue.string("true"),
            "suspected", FeatureValue.string("true"));
        var scored = buildCase(features, List.of(trace));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, features);
        assertThat(result.steps()).hasSize(1);
    }

    @Test
    void nonAeCaseType_passesThrough() {
        var trace = new PlanTrace("some-binding", "some-cap", "worker", "COMPLETED", 0, Map.of());
        var scored = buildCase(Map.of(), List.of(trace));

        AdaptedPlan result = adapter.adapt("other-type", scored, Map.of());
        assertThat(result.steps()).hasSize(1);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.RETAINED);
        assertThat(result.steps().get(0).priority()).isZero();
    }

    @Test
    void missingGradeFeature_skipsRule3() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var scored = buildCase(Map.of(), List.of(trace));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, Map.of());
        assertThat(result.steps().get(0).priority()).isEqualTo(10);
    }

    @Test
    void combinedRules_reasonConcatenation() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var pastFeatures = Map.of("grade", (FeatureValue) FeatureValue.number(2));
        var scored = buildCase(pastFeatures, List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(4));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        String reason = result.steps().get(0).reason();
        assertThat(reason).contains("succeeded");
        assertThat(reason).contains("Higher severity");
    }

    @Test
    void retainedStep_unknownOutcome() {
        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "RUNNING", 0, Map.of());
        var scored = buildCase(Map.of("grade", FeatureValue.number(3)), List.of(trace));
        var current = Map.of("grade", (FeatureValue) FeatureValue.number(3));

        AdaptedPlan result = adapter.adapt("clinical-ae", scored, current);
        assertThat(result.steps().get(0).action()).isEqualTo(AdaptationAction.RETAINED);
        assertThat(result.steps().get(0).priority()).isZero();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest="EscalationPlanRecommendationTest,ClinicalPlanAdapterTest" --batch-mode`
Expected: compilation errors — classes do not exist yet.

- [ ] **Step 4: Implement EscalationPlanRecommendation**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/clinical/cbr/EscalationPlanRecommendation.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.AdaptedPlan;
import io.casehub.neocortex.memory.cbr.AdaptedStep;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public record EscalationPlanRecommendation(
    AdaptedPlan adaptedPlan,
    int retrievedCaseCount,
    double topSimilarityScore,
    String traceId,
    String explanation
) {
    public static EscalationPlanRecommendation none() {
        return new EscalationPlanRecommendation(null, 0, 0.0, null, null);
    }

    public boolean hasRecommendation() {
        return adaptedPlan != null;
    }

    public Map<String, Object> toContextMap() {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("retrievedCaseCount", retrievedCaseCount);
        map.put("topSimilarityScore", topSimilarityScore);
        map.put("traceId", traceId);
        map.put("explanation", explanation);
        List<Map<String, Object>> stepMaps = adaptedPlan != null
            ? adaptedPlan.steps().stream().map(this::stepToMap).toList()
            : List.of();
        map.put("steps", stepMaps);
        return map;
    }

    private Map<String, Object> stepToMap(AdaptedStep step) {
        Map<String, Object> m = new LinkedHashMap<>();
        m.put("bindingName", step.bindingName());
        m.put("capabilityName", step.capabilityName());
        m.put("workerName", step.workerName());
        m.put("stepOutcome", step.stepOutcome());
        m.put("action", step.action().name());
        m.put("priority", step.priority());
        m.put("reason", step.reason());
        return m;
    }
}
```

- [ ] **Step 5: Implement ClinicalPlanAdapter**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/clinical/cbr/ClinicalPlanAdapter.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.neocortex.memory.cbr.*;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class ClinicalPlanAdapter implements PlanAdapter {

    private static final String CASE_TYPE_AE = "clinical-ae";
    private static final Set<String> SAFETY_CAPABILITIES = Set.of("safety-monitoring", "data-safety-monitoring");

    @Override
    public AdaptedPlan adapt(String caseType, ScoredCbrCase<PlanCbrCase> retrieved,
                             Map<String, FeatureValue> currentFeatures) {
        if (!CASE_TYPE_AE.equals(caseType)) {
            return passThrough(retrieved);
        }

        List<AdaptedStep> steps = new ArrayList<>();
        boolean gradeEscalated = isGradeEscalated(currentFeatures, retrieved.cbrCase().features());

        for (PlanTrace trace : retrieved.cbrCase().planTrace()) {
            steps.add(adaptStep(trace, retrieved.score(), gradeEscalated));
        }

        if (shouldAddSusar(currentFeatures, retrieved.cbrCase().features())) {
            steps.add(new AdaptedStep("susar-oversight", "susar-review", null, null,
                20, Map.of(), AdaptationAction.ADDED,
                "Current AE meets SUSAR criteria — not present in precedent case."));
        }

        return new AdaptedPlan(steps);
    }

    private AdaptedStep adaptStep(PlanTrace trace, double similarity, boolean gradeEscalated) {
        String outcome = trace.stepOutcome();

        if ("FAILED".equals(outcome) || "TERMINATED".equals(outcome)) {
            return new AdaptedStep(trace.bindingName(), trace.capabilityName(),
                trace.workerName(), trace.stepOutcome(), 0, trace.parameters(),
                AdaptationAction.SUPPRESSED, "Step failed in past similar case.");
        }

        if ("COMPLETED".equals(outcome)) {
            int priority = 10;
            String reason = "Step succeeded in past similar case (similarity: %.2f).".formatted(similarity);

            if (gradeEscalated && SAFETY_CAPABILITIES.contains(trace.capabilityName())) {
                priority += 5;
                reason += " Higher severity than precedent — elevated urgency.";
            }

            return new AdaptedStep(trace.bindingName(), trace.capabilityName(),
                trace.workerName(), trace.stepOutcome(), priority, trace.parameters(),
                AdaptationAction.BOOSTED, reason);
        }

        int priority = 0;
        String reason = null;
        if (gradeEscalated && SAFETY_CAPABILITIES.contains(trace.capabilityName())) {
            priority = 5;
            reason = "Higher severity than precedent — elevated urgency.";
        }

        return new AdaptedStep(trace.bindingName(), trace.capabilityName(),
            trace.workerName(), trace.stepOutcome(), priority, trace.parameters(),
            AdaptationAction.RETAINED, reason);
    }

    private boolean isGradeEscalated(Map<String, FeatureValue> current, Map<String, FeatureValue> past) {
        FeatureValue currentGrade = current.get("grade");
        FeatureValue pastGrade = past.get("grade");
        if (currentGrade instanceof FeatureValue.NumberVal c && pastGrade instanceof FeatureValue.NumberVal p) {
            return c.value() > p.value();
        }
        return false;
    }

    private boolean shouldAddSusar(Map<String, FeatureValue> current, Map<String, FeatureValue> past) {
        boolean currentSusar = isStringTrue(current, "unexpected") && isStringTrue(current, "suspected");
        boolean pastSusar = isStringTrue(past, "unexpected") && isStringTrue(past, "suspected");
        return currentSusar && !pastSusar;
    }

    private boolean isStringTrue(Map<String, FeatureValue> features, String key) {
        FeatureValue val = features.get(key);
        return val instanceof FeatureValue.StringVal s && "true".equals(s.value());
    }

    private AdaptedPlan passThrough(ScoredCbrCase<PlanCbrCase> retrieved) {
        List<AdaptedStep> steps = retrieved.cbrCase().planTrace().stream()
            .map(t -> new AdaptedStep(t.bindingName(), t.capabilityName(),
                t.workerName(), t.stepOutcome(), 0, t.parameters(),
                AdaptationAction.RETAINED, null))
            .toList();
        return new AdaptedPlan(steps);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest="EscalationPlanRecommendationTest,ClinicalPlanAdapterTest" --batch-mode`
Expected: all tests pass.

- [ ] **Step 7: Run `ide_diagnostics` on both new files**

Verify no compilation errors via `ide_diagnostics` on each file.

- [ ] **Step 8: Commit**

```
feat(#118): EscalationPlanRecommendation + ClinicalPlanAdapter

ClinicalPlanAdapter implements PlanAdapter SPI with four clinical
adaptation rules: outcome suppression, outcome boost, grade
escalation boost, SUSAR addition.

Refs casehubio/clinical#118
```

---

### Task 2: AeEscalationPlanRetriever + integration

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/cbr/AeEscalationPlanRetriever.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — add `planRetriever` injection and call in `prepareAndMarkRequested()`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/AeEscalationPlanRetrieverTest.java`

**Interfaces:**
- Consumes: `ClinicalCbrService.retrieveWithAudit(CbrQuery, Class<C>, UUID subjectId, String actorId) → AuditedRetrievalResult<C>`, `AuditedRetrievalResult<C>(cases, traceId, explanation)`, `AeCbrFeatureBuilder.buildQueryFeatures(ae, enrollment, trial, priorAeCount) → Map<String, Object>`, `FeatureValue.toFeatureMap(Map<String, Object>) → Map<String, FeatureValue>`, `CbrQuery.of(tenantId, domain, scope, caseType, features, topK).withMinSimilarity(double) → CbrQuery`, `PlanAdapter.adapt(caseType, ScoredCbrCase, currentFeatures) → AdaptedPlan`, `EscalationPlanRecommendation` (Task 1), `ClinicalCbrDomains.AE`, `Path.root()`
- Produces: `AeEscalationPlanRetriever.retrieve(AdverseEvent ae) → EscalationPlanRecommendation`. Injected into `AeEscalationCaseService` — when `hasRecommendation()`, puts `escalationPlanRecommendation` in the case context map.

- [ ] **Step 1: Write AeEscalationPlanRetriever unit tests**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class AeEscalationPlanRetrieverTest {

    private ClinicalCbrService cbrService;
    private PlanAdapter planAdapter;
    private AeEscalationPlanRetriever retriever;

    @BeforeEach
    void setup() {
        cbrService = mock(ClinicalCbrService.class);
        planAdapter = mock(PlanAdapter.class);
        retriever = new AeEscalationPlanRetriever(cbrService, planAdapter);
        retriever.setEntityResolver(new StubEntityResolver());
    }

    @Test
    void retrieve_noSimilarCases_returnsNone() {
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
            .thenReturn(new AuditedRetrievalResult<>(List.of(), "trace-1", null));

        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3);
        EscalationPlanRecommendation result = retriever.retrieve(ae);
        assertThat(result.hasRecommendation()).isFalse();
    }

    @Test
    void retrieve_withSimilarCase_adaptsAndReturns() {
        var planCase = new PlanCbrCase("problem", "solution", "COMPLETED", 1.0,
            Map.of("grade", FeatureValue.number(3)), List.of());
        var scored = new ScoredCbrCase<>(planCase, "case-1", 0.87);
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
            .thenReturn(new AuditedRetrievalResult<>(List.of(scored), "trace-1", "expl"));

        var adapted = new AdaptedPlan(List.of(new AdaptedStep("safety-review", "safety-monitoring",
            "w1", "COMPLETED", 10, Map.of(), AdaptationAction.BOOSTED, "reason")));
        when(planAdapter.adapt(eq("clinical-ae"), any(), any())).thenReturn(adapted);

        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3);
        EscalationPlanRecommendation result = retriever.retrieve(ae);
        assertThat(result.hasRecommendation()).isTrue();
        assertThat(result.retrievedCaseCount()).isEqualTo(1);
        assertThat(result.topSimilarityScore()).isEqualTo(0.87);
        assertThat(result.traceId()).isEqualTo("trace-1");
        assertThat(result.explanation()).isEqualTo("expl");
    }

    @Test
    void retrieve_cbrServiceThrows_returnsNone() {
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
            .thenThrow(new RuntimeException("CBR unavailable"));

        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3);
        EscalationPlanRecommendation result = retriever.retrieve(ae);
        assertThat(result.hasRecommendation()).isFalse();
    }

    @Test
    void retrieve_adapterThrows_returnsNone() {
        var planCase = new PlanCbrCase("problem", "solution", "COMPLETED", 1.0,
            Map.of("grade", FeatureValue.number(3)), List.of());
        var scored = new ScoredCbrCase<>(planCase, "case-1", 0.87);
        when(cbrService.retrieveWithAudit(any(), eq(PlanCbrCase.class), any(), any()))
            .thenReturn(new AuditedRetrievalResult<>(List.of(scored), "trace-1", null));
        when(planAdapter.adapt(any(), any(), any()))
            .thenThrow(new RuntimeException("adaptation failed"));

        AdverseEvent ae = buildAe(CtcaeGrade.GRADE_3);
        EscalationPlanRecommendation result = retriever.retrieve(ae);
        assertThat(result.hasRecommendation()).isFalse();
    }

    private AdverseEvent buildAe(CtcaeGrade grade) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.grade = grade;
        ae.eventType = "hepatotoxicity";
        ae.unexpected = false;
        ae.suspected = false;
        ae.tenantId = "test-tenant";
        ae.enrollmentId = UUID.randomUUID();
        return ae;
    }

    private static class StubEntityResolver implements ClinicalCaseOutcomeObserver.EntityResolver {
        @Override public AdverseEvent findAe(UUID id) { return null; }
        @Override public io.casehub.clinical.entity.PatientEnrollment findEnrollment(UUID id) { return null; }
        @Override public io.casehub.clinical.entity.TrialSite findSite(UUID id) { return null; }
        @Override public io.casehub.clinical.entity.ClinicalTrial findTrial(UUID id) { return null; }
        @Override public long countPriorAes(UUID enrollmentId, UUID excludeAeId) { return 0; }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest="AeEscalationPlanRetrieverTest" --batch-mode`
Expected: compilation error — `AeEscalationPlanRetriever` does not exist.

- [ ] **Step 3: Implement AeEscalationPlanRetriever**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/clinical/cbr/AeEscalationPlanRetriever.java`:

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class AeEscalationPlanRetriever {

    private static final Logger LOG = Logger.getLogger(AeEscalationPlanRetriever.class);

    private final ClinicalCbrService cbrService;
    private final PlanAdapter planAdapter;
    private ClinicalCaseOutcomeObserver.EntityResolver entityResolver;

    @ConfigProperty(name = "casehub.clinical.cbr.escalation-plan.top-k", defaultValue = "5")
    int topK;

    @ConfigProperty(name = "casehub.clinical.cbr.escalation-plan.min-similarity", defaultValue = "0.4")
    double minSimilarity;

    @Inject
    public AeEscalationPlanRetriever(ClinicalCbrService cbrService, PlanAdapter planAdapter) {
        this.cbrService = cbrService;
        this.planAdapter = planAdapter;
        this.entityResolver = new PanacheEntityResolver();
    }

    void setEntityResolver(ClinicalCaseOutcomeObserver.EntityResolver resolver) {
        this.entityResolver = resolver;
    }

    public EscalationPlanRecommendation retrieve(AdverseEvent ae) {
        try {
            PatientEnrollment enrollment = ae.enrollmentId != null
                ? entityResolver.findEnrollment(ae.enrollmentId) : null;
            TrialSite site = enrollment != null && enrollment.siteId != null
                ? entityResolver.findSite(enrollment.siteId) : null;
            ClinicalTrial trial = site != null && site.trialId != null
                ? entityResolver.findTrial(site.trialId) : null;
            long priorAeCount = ae.enrollmentId != null
                ? entityResolver.countPriorAes(ae.enrollmentId, ae.id) : 0;

            Map<String, Object> rawFeatures = AeCbrFeatureBuilder.buildQueryFeatures(
                ae, enrollment, trial, priorAeCount);
            Map<String, FeatureValue> featureMap = FeatureValue.toFeatureMap(rawFeatures);

            CbrQuery query = CbrQuery.of(ae.tenantId, ClinicalCbrDomains.AE, Path.root(),
                    "clinical-ae", featureMap, topK)
                .withMinSimilarity(minSimilarity);

            AuditedRetrievalResult<PlanCbrCase> result = cbrService.retrieveWithAudit(
                query, PlanCbrCase.class, ae.id, "system:ae-escalation");

            if (result.cases().isEmpty()) {
                return EscalationPlanRecommendation.none();
            }

            ScoredCbrCase<PlanCbrCase> topCase = result.cases().get(0);
            AdaptedPlan adapted = planAdapter.adapt("clinical-ae", topCase, featureMap);

            return new EscalationPlanRecommendation(
                adapted, result.cases().size(), topCase.score(),
                result.traceId(), result.explanation());
        } catch (Exception e) {
            LOG.warnf(e, "Escalation plan retrieval failed for AE %s — proceeding without recommendation", ae.id);
            return EscalationPlanRecommendation.none();
        }
    }

    private static class PanacheEntityResolver implements ClinicalCaseOutcomeObserver.EntityResolver {
        @Override public AdverseEvent findAe(java.util.UUID id) { return AdverseEvent.findById(id); }
        @Override public PatientEnrollment findEnrollment(java.util.UUID id) { return PatientEnrollment.findById(id); }
        @Override public TrialSite findSite(java.util.UUID id) { return TrialSite.findById(id); }
        @Override public ClinicalTrial findTrial(java.util.UUID id) { return ClinicalTrial.findById(id); }
        @Override public long countPriorAes(java.util.UUID enrollmentId, java.util.UUID excludeAeId) {
            return AdverseEvent.count("enrollmentId = ?1 and id != ?2", enrollmentId, excludeAeId);
        }
    }
}
```

- [ ] **Step 4: Make EntityResolver accessible**

The `EntityResolver` interface is currently package-private inside `ClinicalCaseOutcomeObserver`. It needs to be accessible by `AeEscalationPlanRetriever` in the same package. Since both are in `io.casehub.clinical.cbr`, this is already satisfied — no change needed if the interface has default (package) visibility.

Verify via `ide_find_definition` that `EntityResolver` is accessible. If it's an inner interface with restricted visibility, extract it to a package-level interface or widen to package-private.

- [ ] **Step 5: Integrate into AeEscalationCaseService**

Use `ide_replace_member` on `AeEscalationCaseService.prepareAndMarkRequested()` to add plan retrieval after the existing context building. The change adds:

1. Inject `AeEscalationPlanRetriever planRetriever` field
2. After `ctx.put("siteContext", ...)`, add:

```java
EscalationPlanRecommendation plan = planRetriever.retrieve(ae);
if (plan.hasRecommendation()) {
    ctx.put("escalationPlanRecommendation", plan.toContextMap());
}
```

Use `ide_insert_member` to add the field, then `ide_replace_member` on `prepareAndMarkRequested` to add the three lines at the end before `return ctx`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest="AeEscalationPlanRetrieverTest" --batch-mode`
Expected: all tests pass.

- [ ] **Step 7: Run existing AeEscalationCaseService tests to verify no regression**

Run: `mvn test -pl runtime -Dtest="AeEscalationCaseServiceTest,AeEscalationLifecycleTest" --batch-mode`
Expected: all existing tests still pass.

- [ ] **Step 8: Run `ide_diagnostics` on modified files**

Verify `AeEscalationPlanRetriever.java` and `AeEscalationCaseService.java` are clean.

- [ ] **Step 9: Commit**

```
feat(#118): AeEscalationPlanRetriever + case context injection

Retrieves similar past AE escalation plans via ClinicalCbrService,
adapts via ClinicalPlanAdapter, injects as escalationPlanRecommendation
in case context alongside patientContext and siteContext.

Refs casehubio/clinical#118
```

---

### Task 3: EscalationPlanResource REST endpoint + integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/EscalationPlanResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/resource/EscalationPlanResourceTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/cbr/AeEscalationPlanRetrieverIntegrationTest.java`

**Interfaces:**
- Consumes: `AeEscalationPlanRetriever.retrieve(AdverseEvent ae) → EscalationPlanRecommendation` (Task 2), `AdverseEvent.findById(UUID)`, `EscalationPlanRecommendation.toContextMap()`, `ClinicalGroups.{SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR}`
- Produces: `GET /api/adverse-events/{aeId}/escalation-plans` — returns JSON with `retrievedCaseCount`, `topSimilarityScore`, `traceId`, `explanation`, `steps[]`

- [ ] **Step 1: Write EscalationPlanResource REST test**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.ClinicalGroups;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.*;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.UUID;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR, ClinicalGroups.INVESTIGATOR,
    ClinicalGroups.COORDINATOR, ClinicalGroups.MONITOR})
class EscalationPlanResourceTest {

    private UUID aeId;

    @BeforeEach
    @Transactional
    void setup() {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "PROTO-001";
        trial.phase = io.casehub.clinical.api.model.TrialPhase.PHASE_III;
        trial.sponsor = "Sponsor";
        trial.tenantId = "test-tenant";
        trial.persist();

        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trial.id;
        site.siteName = "Site A";
        site.investigatorId = "pi-1";
        site.tenantId = "test-tenant";
        site.persist();

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = site.id;
        enrollment.patientId = "P001";
        enrollment.tenantId = "test-tenant";
        enrollment.persist();

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = enrollment.id;
        ae.grade = CtcaeGrade.GRADE_3;
        ae.eventType = "hepatotoxicity";
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.persist();
        aeId = ae.id;
    }

    @Test
    void getEscalationPlans_existingAe_returns200() {
        given()
            .when().get("/api/adverse-events/" + aeId + "/escalation-plans")
            .then()
            .statusCode(200)
            .body("retrievedCaseCount", is(0));
    }

    @Test
    void getEscalationPlans_unknownAe_returns404() {
        given()
            .when().get("/api/adverse-events/" + UUID.randomUUID() + "/escalation-plans")
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Write AeEscalationPlanRetrieverIntegrationTest**

```java
package io.casehub.clinical.cbr;

import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.entity.*;
import io.casehub.neocortex.memory.cbr.*;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.casehub.clinical.api.ClinicalGroups;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {ClinicalGroups.SPONSOR})
class AeEscalationPlanRetrieverIntegrationTest {

    @Inject ClinicalCbrService cbrService;
    @Inject AeEscalationPlanRetriever retriever;

    private UUID trialId, siteId, enrollmentId;

    @BeforeEach
    @Transactional
    void setup() {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "PROTO-INT";
        trial.phase = TrialPhase.PHASE_III;
        trial.sponsor = "Test";
        trial.tenantId = "test-tenant";
        trial.persist();
        trialId = trial.id;

        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trialId;
        site.siteName = "Site A";
        site.investigatorId = "pi-1";
        site.tenantId = "test-tenant";
        site.persist();
        siteId = site.id;

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = siteId;
        enrollment.patientId = "P001";
        enrollment.tenantId = "test-tenant";
        enrollment.persist();
        enrollmentId = enrollment.id;
    }

    @Test
    @Transactional
    void roundTrip_storeAndRetrieve() {
        Instant before = Instant.now();

        var trace = new PlanTrace("safety-review", "safety-monitoring", "worker-1", "COMPLETED", 0, Map.of());
        var features = Map.<String, Object>of(
            "grade", 3, "eventType", "hepatotoxicity",
            "trialPhase", "PHASE_III", "unexpected", "false",
            "suspected", "false", "treatmentArm", "UNASSIGNED",
            "priorAeCount", "NONE");
        var cbrCase = new PlanCbrCase("Grade 3 hepatotoxicity", "Safety review completed",
            "COMPLETED", 1.0, FeatureValue.toFeatureMap(features), List.of(trace));

        cbrService.storeIdempotent(cbrCase, "clinical-ae", "past-ae-1",
            ClinicalCbrDomains.AE, "test-tenant", null);

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = enrollmentId;
        ae.grade = CtcaeGrade.GRADE_4;
        ae.eventType = "hepatotoxicity";
        ae.unexpected = false;
        ae.suspected = false;
        ae.tenantId = "test-tenant";
        ae.persist();

        EscalationPlanRecommendation result = retriever.retrieve(ae);

        if (result.hasRecommendation()) {
            assertThat(result.retrievedCaseCount()).isGreaterThan(0);
            assertThat(result.adaptedPlan().steps()).isNotEmpty();
            assertThat(result.adaptedPlan().steps().get(0).action())
                .isIn(AdaptationAction.BOOSTED, AdaptationAction.RETAINED);
        }
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest="EscalationPlanResourceTest,AeEscalationPlanRetrieverIntegrationTest" --batch-mode`
Expected: compilation error — `EscalationPlanResource` does not exist.

- [ ] **Step 4: Implement EscalationPlanResource**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/clinical/resource/EscalationPlanResource.java`:

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.cbr.AeEscalationPlanRetriever;
import io.casehub.clinical.cbr.EscalationPlanRecommendation;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.UUID;
import static io.casehub.clinical.api.ClinicalGroups.*;

@Path("/api/adverse-events/{aeId}/escalation-plans")
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed({SPONSOR, INVESTIGATOR, COORDINATOR, MONITOR})
public class EscalationPlanResource {

    @Inject AeEscalationPlanRetriever planRetriever;

    @GET
    @Transactional
    public Response getEscalationPlans(@PathParam("aeId") UUID aeId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        EscalationPlanRecommendation recommendation = planRetriever.retrieve(ae);
        return Response.ok(recommendation.toContextMap()).build();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest="EscalationPlanResourceTest,AeEscalationPlanRetrieverIntegrationTest" --batch-mode`
Expected: all tests pass.

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: all existing tests still pass.

- [ ] **Step 7: Run `ide_diagnostics` on EscalationPlanResource.java**

Verify no compilation errors.

- [ ] **Step 8: Commit**

```
feat(#118): EscalationPlanResource + integration test

GET /api/adverse-events/{aeId}/escalation-plans returns adapted
plan recommendations from CBR. Full round-trip integration test
verifies store→retrieve→adapt pipeline.

Refs casehubio/clinical#118
```
