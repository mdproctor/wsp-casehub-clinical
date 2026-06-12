# ActionRiskClassifier — Layer 8 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `ClinicalActionRiskClassifier` (Layer 8) — a SUSAR criteria evaluator worker that gates Grade 4/5 unexpected AEs before reporting, wired into the existing `ae-escalation.yaml` case.

**Architecture:** `ClinicalActionType` enum in `api/model/` carries all gate metadata per constant. `ClinicalActionRiskClassifier @ApplicationScoped @RiskClassifier` delegates to it (no switch statements). `SusarCriteriaEvaluator @DefaultBean` implements `SusarEvaluatorFunction` (named CDI interface extending `Function<Map,WorkerResult>`) and registers as a Worker in `ClinicalAdverseEventCaseHub.getDefinition()`. The engine's `ChainedReactiveActionRiskClassifier` discovers the classifier automatically; `ActionGateWorkItemHandler` (on classpath from Layer 5) creates the gate WorkItem on plan.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api 0.2-SNAPSHOT (ActionRiskClassifier, PlannedAction, Worker, Capability, WorkerResult), casehub-engine-blackboard (PlanningStrategyLoopControl — already on classpath), JUnit 5 + AssertJ + Awaitility, Mockito (@InjectMock for policy override in integration test)

**Spec:** `specs/2026-06-11-action-risk-classifier-design.md`

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `runtime/.../db/migration/default/V111__adverse_event_unexpectedness.sql` | Create | Add `unexpected`, `suspected` columns |
| `runtime/.../entity/AdverseEvent.java` | Modify | Add `unexpected`, `suspected` fields |
| `runtime/.../resource/PatientResource.java` | Modify | Add request fields; set entity fields before service call |
| `runtime/.../service/AeEscalationCaseService.java` | Modify | Propagate `unexpected`, `suspected` into case context |
| `api/.../api/model/ClinicalActionType.java` | Create | Enum with 5 regulatory gate constants |
| `api/.../api/spi/SusarEvaluatorFunction.java` | Create | Named CDI interface for CDI displacement contract |
| `runtime/.../routing/ClinicalActionRiskClassifier.java` | Create | @RiskClassifier bean — delegates to enum |
| `runtime/.../service/SusarCriteriaEvaluator.java` | Create | @DefaultBean worker function |
| `runtime/.../service/ClinicalAdverseEventCaseHub.java` | Modify | Override `getDefinition()` to register worker |
| `runtime/.../resources/clinical/ae-escalation.yaml` | Modify | Add `susar-assessment` binding |
| `api/.../api/model/ClinicalActionTypeTest.java` | Create | Unit tests — enum |
| `runtime/.../routing/ClinicalActionRiskClassifierTest.java` | Create | Unit tests — classifier |
| `runtime/.../service/SusarCriteriaEvaluatorTest.java` | Create | Unit tests — evaluator |
| `runtime/.../service/SusarActionGateLifecycleTest.java` | Create | @QuarkusTest integration test |

---

## Task 1: Data model — entity fields and Flyway migration

**Files:**
- Create: `runtime/src/main/resources/db/migration/default/V111__adverse_event_unexpectedness.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`

- [ ] **Step 1.1: Create the migration**

```sql
-- runtime/src/main/resources/db/migration/default/V111__adverse_event_unexpectedness.sql
ALTER TABLE adverse_event ADD COLUMN unexpected BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE adverse_event ADD COLUMN suspected  BOOLEAN NOT NULL DEFAULT TRUE;
```

- [ ] **Step 1.2: Add entity fields to `AdverseEvent.java`**

After the existing `engineCaseId` field:
```java
@Column(nullable = false)
public boolean unexpected = false;

/** Conservative default per ICH E2A §I.A.1: all AEs assumed IMP-suspected unless explicitly false. */
@Column(nullable = false)
public boolean suspected = true;
```

- [ ] **Step 1.3: Verify compilation**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS`. Existing tests still pass — `unexpected` defaults to `false`, `suspected` defaults to `true`, so all existing entity construction compiles unchanged.

- [ ] **Step 1.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/db/migration/default/V111__adverse_event_unexpectedness.sql \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(entity): add unexpected and suspected fields to AdverseEvent — Layer 8 SUSAR criteria — Refs #47"
```

---

## Task 2: PatientResource — request fields and entity assignment

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`

- [ ] **Step 2.1: Add fields to `ReportAdverseEventRequest` record**

Replace:
```java
public record ReportAdverseEventRequest(
    @NotNull CtcaeGrade grade,
    @NotNull Instant occurredAt,
    EventActuality actuality
) {}
```
With:
```java
public record ReportAdverseEventRequest(
    @NotNull CtcaeGrade grade,
    @NotNull Instant occurredAt,
    EventActuality actuality,
    Boolean unexpected,
    Boolean suspected
) {}
```

- [ ] **Step 2.2: Set entity fields in `reportAdverseEvent()` before calling service**

After the line `ae.occurredAt = req.occurredAt();`, add:
```java
ae.unexpected = req.unexpected() != null ? req.unexpected() : false;
ae.suspected  = req.suspected()  != null ? req.suspected()  : true;
```

- [ ] **Step 2.3: Verify compilation**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): add unexpected and suspected fields to ReportAdverseEventRequest — Refs #47"
```

---

## Task 3: AeEscalationCaseService — propagate fields into case context

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java`

- [ ] **Step 3.1: Add context keys to `prepareAndMarkRequested()`**

At the end of the context construction block, after `ctx.put("siteContext", ...)`, add:
```java
ctx.put("unexpected", ae.unexpected);
ctx.put("suspected",  ae.suspected);
```

The entity is already loaded at this call site (`AdverseEvent ae = AdverseEvent.findById(event.aeId())`), so no additional query is needed.

- [ ] **Step 3.2: Verify compilation**

```bash
mvn compile -pl api,runtime --batch-mode
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): propagate unexpected and suspected into AE escalation case context — Refs #47"
```

---

## Task 4: ClinicalActionType enum (api module)

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/ClinicalActionType.java`
- Create: `api/src/test/java/io/casehub/clinical/api/model/ClinicalActionTypeTest.java`

- [ ] **Step 4.1: Write the failing test**

```java
// api/src/test/java/io/casehub/clinical/api/model/ClinicalActionTypeTest.java
package io.casehub.clinical.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class ClinicalActionTypeTest {

    @Test
    void susar_criteria_decision_has_correct_metadata() {
        ClinicalActionType type = ClinicalActionType.SUSAR_CRITERIA_DECISION;
        assertThat(type.actionType()).isEqualTo("susar.criteria.decision");
        assertThat(type.candidateGroups()).containsExactly("qualified-investigator");
        assertThat(type.reversible()).isFalse();
        assertThat(type.scope()).isEqualTo("casehubio/clinical/oversight");
        assertThat(type.expiresIn()).isNull();
        assertThat(type.reason()).contains("SUSAR");
    }

    @Test
    void susar_regulatory_filing_has_correct_metadata() {
        assertThat(ClinicalActionType.SUSAR_REGULATORY_FILING.actionType()).isEqualTo("susar.regulatory.filing");
        assertThat(ClinicalActionType.SUSAR_REGULATORY_FILING.candidateGroups()).containsExactly("qualified-investigator");
        assertThat(ClinicalActionType.SUSAR_REGULATORY_FILING.reversible()).isFalse();
    }

    @Test
    void patient_withdrawal_has_correct_metadata() {
        assertThat(ClinicalActionType.PATIENT_WITHDRAWAL.actionType()).isEqualTo("patient.withdrawal");
        assertThat(ClinicalActionType.PATIENT_WITHDRAWAL.candidateGroups()).containsExactly("principal-investigator");
        assertThat(ClinicalActionType.PATIENT_WITHDRAWAL.reversible()).isFalse();
    }

    @Test
    void dose_modification_is_reversible() {
        assertThat(ClinicalActionType.DOSE_MODIFICATION.actionType()).isEqualTo("dose.modification");
        assertThat(ClinicalActionType.DOSE_MODIFICATION.candidateGroups()).containsExactly("principal-investigator");
        assertThat(ClinicalActionType.DOSE_MODIFICATION.reversible()).isTrue();
    }

    @Test
    void protocol_deviation_recording_has_two_candidate_groups() {
        ClinicalActionType type = ClinicalActionType.PROTOCOL_DEVIATION_RECORDING;
        assertThat(type.actionType()).isEqualTo("protocol.deviation.recording");
        assertThat(type.candidateGroups()).containsExactly("principal-investigator", "irb-committee");
        assertThat(type.reversible()).isFalse();
    }

    @Test
    void fromActionType_round_trips_all_constants() {
        for (ClinicalActionType type : ClinicalActionType.values()) {
            assertThat(ClinicalActionType.fromActionType(type.actionType())).contains(type);
        }
    }

    @Test
    void fromActionType_returns_empty_for_unknown() {
        assertThat(ClinicalActionType.fromActionType("unknown.action")).isEmpty();
    }

    @Test
    void fromActionType_returns_empty_for_null() {
        assertThat(ClinicalActionType.fromActionType(null)).isEmpty();
    }

    @Test
    void all_types_share_oversight_scope() {
        for (ClinicalActionType type : ClinicalActionType.values()) {
            assertThat(type.scope()).isEqualTo("casehubio/clinical/oversight");
        }
    }

    @Test
    void susar_criteria_has_narrower_groups_than_deviation_recording() {
        // Fewer candidateGroups = more restrictive in ChainedReactiveActionRiskClassifier.narrower()
        // per GE-20260607-326c7e
        assertThat(ClinicalActionType.SUSAR_CRITERIA_DECISION.candidateGroups()).hasSize(1);
        assertThat(ClinicalActionType.PROTOCOL_DEVIATION_RECORDING.candidateGroups()).hasSize(2);
    }
}
```

- [ ] **Step 4.2: Run test to verify it fails (compilation)**

```bash
mvn test -pl api -Dtest=ClinicalActionTypeTest --batch-mode 2>&1 | grep "ERROR\|error:" | head -5
```

Expected: compilation error — `ClinicalActionType` does not exist.

- [ ] **Step 4.3: Implement `ClinicalActionType`**

```java
// api/src/main/java/io/casehub/clinical/api/model/ClinicalActionType.java
package io.casehub.clinical.api.model;

import java.time.Duration;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

/**
 * Typed taxonomy of consequential clinical trial agent actions requiring oversight gates.
 * All types are ALWAYS-gated — these are regulatory obligations, not configurable policy.
 *
 * <p>candidateGroups semantics (GE-20260607-326c7e): fewer entries = more restrictive
 * in ChainedReactiveActionRiskClassifier.narrower(). SUSAR types (1 group) are the
 * tightest gates. Protocol deviation recording (2 groups) is broadest.
 *
 * <p>Classification logic lives in {@code ClinicalActionRiskClassifier}. This enum owns
 * only the data — pure Java, no framework dependencies.
 */
public enum ClinicalActionType {

    SUSAR_CRITERIA_DECISION(
        false,
        List.of("qualified-investigator"),
        "SUSAR criteria met — clinician sign-off required before regulatory clock starts"),

    SUSAR_REGULATORY_FILING(
        false,
        List.of("qualified-investigator"),
        "Regulatory submission of SUSAR report — qualified investigator confirmation required"),

    PATIENT_WITHDRAWAL(
        false,
        List.of("principal-investigator"),
        "Patient withdrawal is irreversible — principal investigator confirmation required"),

    DOSE_MODIFICATION(
        true,
        List.of("principal-investigator"),
        "Dose modification recommendation requires physician approval — reversible"),

    PROTOCOL_DEVIATION_RECORDING(
        false,
        List.of("principal-investigator", "irb-committee"),
        "Protocol deviation recording — PI or IRB committee confirmation required");

    private static final String OVERSIGHT_SCOPE = "casehubio/clinical/oversight";

    private final boolean reversible;
    private final List<String> candidateGroups;
    private final String reason;

    ClinicalActionType(
            final boolean reversible,
            final List<String> candidateGroups,
            final String reason) {
        this.reversible = reversible;
        this.candidateGroups = List.copyOf(candidateGroups);
        this.reason = reason;
    }

    public boolean reversible()            { return reversible; }
    public List<String> candidateGroups()  { return candidateGroups; }
    public String reason()                 { return reason; }
    public String scope()                  { return OVERSIGHT_SCOPE; }
    /** Null — regulatory deadline policy is post-GA deployment config, not compile-time constant. */
    public Duration expiresIn()            { return null; }

    /** Returns the PlannedAction actionType string: {@code SUSAR_CRITERIA_DECISION → "susar.criteria.decision"}. */
    public String actionType() {
        return name().toLowerCase().replace('_', '.');
    }

    /** Parses a {@code PlannedAction.actionType()} string back to the enum constant. Null-safe. */
    public static Optional<ClinicalActionType> fromActionType(final String actionType) {
        if (actionType == null) return Optional.empty();
        return Arrays.stream(values())
                .filter(a -> a.actionType().equals(actionType))
                .findFirst();
    }
}
```

- [ ] **Step 4.4: Run tests and verify they pass**

```bash
mvn install -pl api --batch-mode && mvn test -pl api -Dtest=ClinicalActionTypeTest --batch-mode 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 10, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/model/ClinicalActionType.java \
  api/src/test/java/io/casehub/clinical/api/model/ClinicalActionTypeTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): ClinicalActionType enum — 5 regulatory gate constants — Refs #47"
```

---

## Task 5: SusarEvaluatorFunction interface (api module)

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/SusarEvaluatorFunction.java`

- [ ] **Step 5.1: Create the interface**

```java
// api/src/main/java/io/casehub/clinical/api/spi/SusarEvaluatorFunction.java
package io.casehub.clinical.api.spi;

import io.casehub.api.model.WorkerResult;
import java.util.Map;
import java.util.function.Function;

/**
 * Named CDI interface for the SUSAR criteria evaluator worker function.
 *
 * <p>Extends {@code Function<Map<String,Object>,WorkerResult>} as a named interface to
 * prevent CDI generic-erasure ambiguity and establish a clean displacement contract:
 * a future ML-based evaluator implements this interface as {@code @ApplicationScoped}
 * (without {@code @DefaultBean}) and displaces the default automatically.
 */
public interface SusarEvaluatorFunction extends Function<Map<String, Object>, WorkerResult> {}
```

- [ ] **Step 5.2: Install api module**

```bash
mvn install -pl api --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/spi/SusarEvaluatorFunction.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): SusarEvaluatorFunction named CDI interface for worker displacement contract — Refs #47"
```

---

## Task 6: ClinicalActionRiskClassifier (runtime module)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/routing/ClinicalActionRiskClassifier.java`
- Create: `runtime/src/test/java/io/casehub/clinical/routing/ClinicalActionRiskClassifierTest.java`

Note: the `routing` package is new — create the directory.

- [ ] **Step 6.1: Write the failing test**

```java
// runtime/src/test/java/io/casehub/clinical/routing/ClinicalActionRiskClassifierTest.java
package io.casehub.clinical.routing;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskDecision;
import io.casehub.clinical.api.model.ClinicalActionType;
import java.util.Map;
import org.junit.jupiter.api.Test;

class ClinicalActionRiskClassifierTest {

    private final ClinicalActionRiskClassifier classifier = new ClinicalActionRiskClassifier();

    @Test
    void susar_criteria_decision_returns_gate_required_with_correct_fields() {
        PlannedAction action = PlannedAction.of(
                "SUSAR test", ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType(), Map.of());
        RiskDecision decision = classifier.classify(action);
        assertThat(decision).isInstanceOf(RiskDecision.GateRequired.class);
        RiskDecision.GateRequired gate = (RiskDecision.GateRequired) decision;
        assertThat(gate.candidateGroups()).containsExactly("qualified-investigator");
        assertThat(gate.reversible()).isFalse();
        assertThat(gate.scope()).isEqualTo("casehubio/clinical/oversight");
        assertThat(gate.expiresIn()).isNull();
        assertThat(gate.reason()).isEqualTo(ClinicalActionType.SUSAR_CRITERIA_DECISION.reason());
    }

    @Test
    void susar_regulatory_filing_returns_gate_required() {
        PlannedAction action = PlannedAction.of(
                "filing", ClinicalActionType.SUSAR_REGULATORY_FILING.actionType(), Map.of());
        assertThat(classifier.classify(action)).isInstanceOf(RiskDecision.GateRequired.class);
    }

    @Test
    void patient_withdrawal_returns_gate_required_with_pi_group() {
        PlannedAction action = PlannedAction.of(
                "withdrawal", ClinicalActionType.PATIENT_WITHDRAWAL.actionType(), Map.of());
        RiskDecision.GateRequired gate = (RiskDecision.GateRequired) classifier.classify(action);
        assertThat(gate.candidateGroups()).containsExactly("principal-investigator");
        assertThat(gate.reversible()).isFalse();
    }

    @Test
    void dose_modification_returns_reversible_gate_required() {
        PlannedAction action = PlannedAction.of(
                "dose", ClinicalActionType.DOSE_MODIFICATION.actionType(), Map.of());
        RiskDecision.GateRequired gate = (RiskDecision.GateRequired) classifier.classify(action);
        assertThat(gate.reversible()).isTrue();
    }

    @Test
    void protocol_deviation_recording_returns_gate_with_two_groups() {
        PlannedAction action = PlannedAction.of(
                "dev", ClinicalActionType.PROTOCOL_DEVIATION_RECORDING.actionType(), Map.of());
        RiskDecision.GateRequired gate = (RiskDecision.GateRequired) classifier.classify(action);
        assertThat(gate.candidateGroups()).containsExactly("principal-investigator", "irb-committee");
    }

    @Test
    void unknown_action_type_returns_autonomous() {
        PlannedAction action = PlannedAction.of("test", "some.unknown.action", Map.of());
        assertThat(classifier.classify(action)).isInstanceOf(RiskDecision.Autonomous.class);
    }

    @Test
    void null_planned_action_returns_autonomous() {
        assertThat(classifier.classify(null)).isInstanceOf(RiskDecision.Autonomous.class);
    }

    @Test
    void null_action_type_returns_autonomous() {
        PlannedAction action = PlannedAction.of("test", null, Map.of());
        assertThat(classifier.classify(action)).isInstanceOf(RiskDecision.Autonomous.class);
    }
}
```

- [ ] **Step 6.2: Run test to verify it fails**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=ClinicalActionRiskClassifierTest --batch-mode 2>&1 | grep -E "ERROR|error:" | head -5
```

Expected: compilation error — `ClinicalActionRiskClassifier` does not exist.

- [ ] **Step 6.3: Implement `ClinicalActionRiskClassifier`**

```java
// runtime/src/main/java/io/casehub/clinical/routing/ClinicalActionRiskClassifier.java
package io.casehub.clinical.routing;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.clinical.api.model.ClinicalActionType;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;

/**
 * Clinical-domain {@link ActionRiskClassifier}. Discovered by casehub-engine via
 * {@link RiskClassifier} CDI qualifier and composed automatically with other classifiers
 * via {@code ChainedReactiveActionRiskClassifier.mostRestrictive()}.
 *
 * <p>All five clinical action types are unconditionally gated (ALWAYS policy) — these
 * are regulatory obligations (GCP, ICH E2A, 21 CFR Part 312), not configurable thresholds.
 *
 * <p>Unknown action types return {@link RiskDecision.Autonomous} — this classifier does
 * not gate actions it does not own.
 */
@ApplicationScoped
@RiskClassifier
public class ClinicalActionRiskClassifier implements ActionRiskClassifier {

    @Override
    public RiskDecision classify(final PlannedAction action) {
        Optional<ClinicalActionType> typeOpt = ClinicalActionType.fromActionType(
                action != null ? action.actionType() : null);
        if (typeOpt.isEmpty()) {
            return new RiskDecision.Autonomous();
        }
        ClinicalActionType type = typeOpt.get();
        return new RiskDecision.GateRequired(
                type.reason(), type.reversible(), type.candidateGroups(),
                type.expiresIn(), type.scope());
    }
}
```

- [ ] **Step 6.4: Run tests and verify they pass**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=ClinicalActionRiskClassifierTest --batch-mode 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 8, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/routing/ClinicalActionRiskClassifier.java \
  runtime/src/test/java/io/casehub/clinical/routing/ClinicalActionRiskClassifierTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(routing): ClinicalActionRiskClassifier @RiskClassifier — delegates to ClinicalActionType — Refs #47"
```

---

## Task 7: SusarCriteriaEvaluator (runtime module)

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java`

- [ ] **Step 7.1: Write the failing test**

```java
// runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.WorkerResult;
import io.casehub.clinical.api.model.ClinicalActionType;
import java.util.HashMap;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SusarCriteriaEvaluatorTest {

    private final SusarCriteriaEvaluator evaluator = new SusarCriteriaEvaluator();

    @Test
    void grade4_unexpected_suspected_triggers_gate() {
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_4", "unexpected", true, "suspected", true));
        assertThat(result.plannedAction()).isNotNull();
        assertThat(result.plannedAction().actionType())
                .isEqualTo(ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType());
        assertThat(result.plannedAction().description())
                .isEqualTo(ClinicalActionType.SUSAR_CRITERIA_DECISION.reason());
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", true));
    }

    @Test
    void grade5_unexpected_suspected_triggers_gate() {
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_5", "unexpected", true, "suspected", true));
        assertThat(result.plannedAction()).isNotNull();
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", true));
    }

    @Test
    void absent_suspected_key_defaults_to_true_and_triggers_gate() {
        // ICH E2A §I.A.1: absent suspected → conservative default (IMP-related assumed)
        Map<String, Object> ctx = new HashMap<>();
        ctx.put("grade", "GRADE_5");
        ctx.put("unexpected", true);
        // "suspected" deliberately absent
        WorkerResult result = evaluator.apply(ctx);
        assertThat(result.plannedAction()).isNotNull();
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", true));
    }

    @Test
    void grade4_not_unexpected_returns_no_gate() {
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_4", "unexpected", false, "suspected", true));
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", false));
    }

    @Test
    void grade3_unexpected_returns_no_gate() {
        // Grade 3 is 15-day expedited path (21 CFR 312.32(c)(1)(ii)) — deferred out of scope
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_3", "unexpected", true, "suspected", true));
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", false));
    }

    @Test
    void grade4_unexpected_true_suspected_false_returns_no_gate() {
        // Explicit non-IMP event (e.g. road traffic accident) — excluded by suspected=false
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_4", "unexpected", true, "suspected", false));
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).isEqualTo(Map.of("susarRequired", false));
    }

    @Test
    void all_no_gate_paths_include_susar_required_false() {
        // Output map always contains susarRequired — YAML outputMapping reads .susarRequired
        WorkerResult result = evaluator.apply(
                Map.of("grade", "GRADE_1", "unexpected", true, "suspected", true));
        assertThat(result.output()).containsEntry("susarRequired", false);
        assertThat(result.plannedAction()).isNull();
    }
}
```

- [ ] **Step 7.2: Run test to verify it fails**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=SusarCriteriaEvaluatorTest --batch-mode 2>&1 | grep -E "ERROR|error:" | head -5
```

Expected: compilation error — `SusarCriteriaEvaluator` does not exist.

- [ ] **Step 7.3: Implement `SusarCriteriaEvaluator`**

```java
// runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java
package io.casehub.clinical.service;

import io.casehub.api.model.WorkerResult;
import io.casehub.api.spi.PlannedAction;
import io.casehub.clinical.api.model.ClinicalActionType;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.spi.SusarEvaluatorFunction;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.Set;

/**
 * Default SUSAR criteria evaluator (7-day expedited path: Grade 4/5 only).
 *
 * <p>Gates when: {@code grade ∈ {GRADE_4, GRADE_5}} AND {@code unexpected == true}
 * AND {@code suspected != false} (absent key → conservative default per ICH E2A §I.A.1).
 *
 * <p>Grade 3 unexpected AEs (15-day path, 21 CFR 312.32(c)(1)(ii)) are NOT gated
 * here — tracked in clinical#76 together with gate rejection/expiry handling.
 *
 * <p>Displacement: annotate a replacement with {@code @ApplicationScoped} (without
 * {@code @DefaultBean}) and implement {@link SusarEvaluatorFunction}.
 */
@DefaultBean
@ApplicationScoped
public class SusarCriteriaEvaluator implements SusarEvaluatorFunction {

    private static final Set<String> GATE_GRADES = Set.of(
            CtcaeGrade.GRADE_4.name(), CtcaeGrade.GRADE_5.name());

    @Override
    public WorkerResult apply(final Map<String, Object> context) {
        String grade      = (String) context.get("grade");
        boolean unexpected = Boolean.TRUE.equals(context.get("unexpected"));
        // absent "suspected" key → conservative default (true): ICH E2A §I.A.1
        boolean suspected  = !Boolean.FALSE.equals(context.get("suspected"));

        if (GATE_GRADES.contains(grade) && unexpected && suspected) {
            return WorkerResult.of(
                    Map.of("susarRequired", true),
                    PlannedAction.of(
                            ClinicalActionType.SUSAR_CRITERIA_DECISION.reason(),
                            ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType(),
                            Map.of("grade", grade, "unexpected", true)));
        }
        return WorkerResult.of(Map.of("susarRequired", false));
    }
}
```

- [ ] **Step 7.4: Run tests and verify they pass**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=SusarCriteriaEvaluatorTest --batch-mode 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 7, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): SusarCriteriaEvaluator @DefaultBean — gates Grade 4/5 unexpected suspected AEs — Refs #47"
```

---

## Task 8: ClinicalAdverseEventCaseHub — register worker

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java`

- [ ] **Step 8.1: Override `getDefinition()` to inject and register the evaluator**

Replace the entire file:
```java
// runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java
package io.casehub.clinical.service;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.Worker;
import io.casehub.clinical.api.spi.SusarEvaluatorFunction;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

/**
 * Case definition for Grade 3+ adverse event safety escalation (Layer 5).
 *
 * <p>Augments the YAML definition with the {@link SusarCriteriaEvaluator} worker
 * (Layer 8), registered against the {@code safety-monitoring} capability.
 * The {@link SusarEvaluatorFunction} injection point enables CDI displacement:
 * a future ML agent implements the interface as {@code @ApplicationScoped} (no
 * {@code @DefaultBean}) and is selected automatically by Quarkus ArC.
 */
@ApplicationScoped
public class ClinicalAdverseEventCaseHub extends YamlCaseHub {

    @Inject SusarEvaluatorFunction susarEvaluator;

    private volatile CaseDefinition augmentedDefinition;

    public ClinicalAdverseEventCaseHub() {
        super("clinical/ae-escalation.yaml");
    }

    @Override
    public CaseDefinition getDefinition() {
        if (augmentedDefinition == null) {
            synchronized (this) {
                if (augmentedDefinition == null) {
                    CaseDefinition def = super.getDefinition();
                    def.getWorkers().add(Worker.builder()
                            .name("susar-criteria-evaluator")
                            .capabilities(List.of(Capability.builder()
                                    .name("safety-monitoring")
                                    .inputSchema(".")
                                    .outputSchema(".")
                                    .build()))
                            .function(susarEvaluator)
                            .build());
                    augmentedDefinition = def;
                }
            }
        }
        return augmentedDefinition;
    }
}
```

- [ ] **Step 8.2: Verify compilation**

```bash
mvn install -pl api --batch-mode -q && mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 8.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalAdverseEventCaseHub.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): ClinicalAdverseEventCaseHub registers SusarCriteriaEvaluator worker — Refs #47"
```

---

## Task 9: ae-escalation.yaml — susar-assessment binding

**Files:**
- Modify: `runtime/src/main/resources/clinical/ae-escalation.yaml`

- [ ] **Step 9.1: Add the `susar-assessment` binding**

Append to the `bindings:` list in `ae-escalation.yaml`:
```yaml
    - name: susar-assessment
      on:
        contextChange:
          filter: ".susarAssessmentComplete == null"
      worker:
        capability: safety-monitoring
        input: "{ aeId: .aeId, grade: .grade, enrollmentId: .enrollmentId, unexpected: (.unexpected // false), suspected: (.suspected // true) }"
        output: "{ susarAssessmentComplete: true, susarRequired: .susarRequired }"
```

Note: `(.unexpected // false)` and `(.suspected // true)` are JQ defaults — safe if the key is absent from the context (shouldn't happen given Task 3, but defensive).

Note: `susarRequired` is a hook key — its purpose is to trigger a future `regulatory-submission` binding (tracked clinical#76). It does not self-execute in this layer.

Note: Idempotency is preserved by `PlanningStrategyLoopControl.filterToDispatchable()` — once the plan item is RUNNING, subsequent CONTEXT_CHANGED events do not re-dispatch. Clinical already has `casehub-engine-blackboard` on classpath (required for DSMB, Layer 6).

- [ ] **Step 9.2: Verify TrialCoordinationYamlTest still passes (existing YAML tests)**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=TrialCoordinationYamlTest --batch-mode 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 3, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

- [ ] **Step 9.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/clinical/ae-escalation.yaml
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(yaml): add susar-assessment worker binding to ae-escalation — Refs #47"
```

---

## Task 10: SusarActionGateLifecycleTest — integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java`

The test uses `@InjectMock AdverseEventEscalationPolicy` returning `engineManaged(true, false)` — senior monitor required, no DSMB. This gives a two-workitem case (safety-review humanTask + susar-assessment gate) rather than three, making the test sequence manageable.

The gate WorkItem is detected by its `title` field, which `ActionGateWorkItemHandler` sets to `GateRequired.reason()` = `ClinicalActionType.SUSAR_CRITERIA_DECISION.reason()` = "SUSAR criteria met — clinician sign-off required before regulatory clock starts".

- [ ] **Step 10.1: Write the failing test**

```java
// runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java
package io.casehub.clinical.service;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.ClinicalActionType;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.support.WorkItemCompletionCapture;
import io.casehub.clinical.support.WorkItemQueries;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.workadapter.WorkItemLifecycleAdapter;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SusarActionGateLifecycleTest {

    @Inject AeEscalationCaseService aeEscalationCaseService;
    @Inject WorkItemQueries workItemQueries;
    @Inject WorkItemService workItemService;
    @Inject WorkItemCompletionCapture completionCapture;
    @Inject WorkItemLifecycleAdapter lifecycleAdapter;
    @InjectMock AdverseEventEscalationPolicy mockPolicy;

    private UUID aeId;
    private UUID enrollmentId;
    private UUID siteId;

    @BeforeEach
    @Transactional
    void setup() {
        aeId = UUID.randomUUID();
        enrollmentId = UUID.randomUUID();
        siteId = UUID.randomUUID();
        completionCapture.reset();

        // Override policy: senior monitor required, no DSMB — two workitems only
        when(mockPolicy.evaluate(any()))
                .thenReturn(AdverseEventEscalationRequirements.engineManaged(true, false));

        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = enrollmentId;
        ae.grade = CtcaeGrade.GRADE_4;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.persist();
    }

    @Test
    void grade4_unexpected_suspected_creates_susar_gate_workitem() throws Exception {
        aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_4));

        // Await SUSAR gate WorkItem — created by ActionGateWorkItemHandler on action gate schedule
        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> {
                    List<WorkItem> gateItems = susarGateWorkItems();
                    assertThat(gateItems).as("SUSAR gate WorkItem").isNotEmpty();
                    assertThat(gateItems.get(0).title)
                            .isEqualTo(ClinicalActionType.SUSAR_CRITERIA_DECISION.reason());
                    assertThat(gateItems.get(0).candidateGroups)
                            .isEqualTo("qualified-investigator");
                });
    }

    @Test
    void gate_approval_allows_case_to_complete() throws Exception {
        aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_4));

        // Wait for gate WorkItem
        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(susarGateWorkItems()).isNotEmpty());

        // Wait for safety-review humanTask WorkItem
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(safetyReviewWorkItems()).isNotEmpty());

        // Step 1: approve the SUSAR gate WorkItem
        WorkItem gateWorkItem = susarGateWorkItems().get(0);
        workItemService.completeFromSystem(gateWorkItem.id, "qualified-investigator",
                "{\"decision\":\"APPROVED\",\"reviewedAt\":\"" + Instant.now() + "\"}");

        await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(completionCapture.wasCompleted(gateWorkItem.id)).isTrue());

        // Drive gate approval through engine adapter
        WorkItem completedGate = susarGateWorkItems().get(0);
        lifecycleAdapter.onWorkItemLifecycle(WorkItemLifecycleEvent.of(
                "COMPLETED", completedGate, "qualified-investigator", completedGate.resolution));

        // Step 2: complete the safety-review humanTask
        WorkItem safetyWorkItem = safetyReviewWorkItems().get(0);
        workItemService.completeFromSystem(safetyWorkItem.id, "senior-monitor",
                "{\"outcome\":\"REVIEWED\",\"reviewedAt\":\"" + Instant.now() + "\"}");

        await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(completionCapture.wasCompleted(safetyWorkItem.id)).isTrue());

        WorkItem completedSafety = safetyReviewWorkItems().get(0);
        lifecycleAdapter.onWorkItemLifecycle(WorkItemLifecycleEvent.of(
                "COMPLETED", completedSafety, "senior-monitor", completedSafety.resolution));

        // Case completes — AeEscalationListener sets status COMPLETED
        await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() ->
                        assertThat(findAe(aeId).escalationStatus).isEqualTo(AeEscalationStatus.COMPLETED));
    }

    @Test
    void grade5_not_unexpected_does_not_create_susar_gate() throws Exception {
        // Setup: Grade 5, unexpected=false
        updateAeUnexpected(false);

        aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_5));

        // Wait for safety-review WorkItem (confirms case started)
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(safetyReviewWorkItems()).isNotEmpty());

        // No SUSAR gate WorkItem should have been created
        await().during(2, SECONDS).atMost(3, SECONDS)
                .untilAsserted(() -> assertThat(susarGateWorkItems())
                        .as("No SUSAR gate for non-unexpected AE").isEmpty());
    }

    @Test
    void grade3_unexpected_does_not_create_susar_gate() throws Exception {
        // Grade 3 is 15-day path — not covered by this layer (deferred)
        aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_3));

        // Wait for safety-review WorkItem
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
                .untilAsserted(() -> assertThat(safetyReviewWorkItems()).isNotEmpty());

        await().during(2, SECONDS).atMost(3, SECONDS)
                .untilAsserted(() -> assertThat(susarGateWorkItems())
                        .as("No SUSAR gate for Grade 3").isEmpty());
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private AdverseEventReportedEvent aeEvent(CtcaeGrade grade) {
        return new AdverseEventReportedEvent(aeId, enrollmentId, siteId, grade, Instant.now(), "test-tenant");
    }

    private List<WorkItem> susarGateWorkItems() {
        return workItemQueries.scanAll().stream()
                .filter(wi -> wi.title != null
                        && wi.title.equals(ClinicalActionType.SUSAR_CRITERIA_DECISION.reason()))
                .toList();
    }

    private List<WorkItem> safetyReviewWorkItems() {
        return workItemQueries.scanAll().stream()
                .filter(wi -> wi.title != null && wi.title.contains("Senior safety monitor"))
                .toList();
    }

    @Transactional
    AdverseEvent findAe(UUID id) {
        return AdverseEvent.findById(id);
    }

    @Transactional
    void updateAeUnexpected(boolean unexpected) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae != null) {
            ae.unexpected = unexpected;
            ae.grade = CtcaeGrade.GRADE_5;
        }
    }
}
```

- [ ] **Step 10.2: Run test to verify it fails**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=SusarActionGateLifecycleTest --batch-mode 2>&1 | grep -E "Tests run|FAILURE|ERROR" | head -10
```

Expected: compilation succeeds but tests fail (e.g. assertion error: SUSAR gate WorkItem not found within timeout — classifier not yet wired into the engine CDI context, or gate not firing).

If compilation fails, check that all types are imported correctly and the api module is installed.

- [ ] **Step 10.3: Run all unit tests to confirm existing tests still pass**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime --batch-mode 2>&1 | grep -E "Tests run.*Failures|BUILD"
```

Expected: `BUILD SUCCESS` with the unit tests passing (integration test failures are expected until wiring is complete).

- [ ] **Step 10.4: Run just the new integration test**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest=SusarActionGateLifecycleTest --batch-mode 2>&1 | tail -30
```

Examine output. If the gate WorkItem is not appearing, verify:
1. `ClinicalAdverseEventCaseHub.getDefinition()` is returning the augmented definition (log statement or breakpoint)
2. The `susar-assessment` binding filter `.susarAssessmentComplete == null` is satisfied
3. The `SusarCriteriaEvaluator` is being dispatched and returning a PlannedAction

- [ ] **Step 10.5: Commit (when all 4 integration tests pass)**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(service): SusarActionGateLifecycleTest — gate creation, approval, and negative paths — Refs #47"
```

---

## Task 11: Full test suite verification

- [ ] **Step 11.1: Run the full reactor test**

```bash
mvn test --batch-mode 2>&1 | tail -20
```

Expected: `Tests run: N, Failures: 0, Errors: 0` for all modules, `BUILD SUCCESS`.

Note: `IrbGateLifecycleTest` has pre-existing ConditionTimeout failures on main (tracked as Task #2). These are unrelated to this branch. If they appear, confirm they also fail on main before investigating.

- [ ] **Step 11.2: Invoke code review**

Use `superpowers:requesting-code-review` on all changes.

Any finding Minor or above that is not fixed this session must be captured as a GitHub issue before sign-off. Batch related minor findings into a single issue.

- [ ] **Step 11.3: Invoke doc sync**

Use `implementation-doc-sync` — scoped to what changed this session.

---

## Troubleshooting

**Gate WorkItem not created:**
- Confirm `ClinicalAdverseEventCaseHub.getDefinition()` is being called and returns the augmented definition (add a log or check CDI scope)
- Confirm `casehub-engine-work-adapter` is on the classpath — `ActionGateWorkItemHandler` must be active
- Confirm `SusarCriteriaEvaluator.apply()` receives the context and the grade is GRADE_4 or GRADE_5

**CDI AmbiguousResolutionException on `SusarEvaluatorFunction`:**
- Should not happen — `@DefaultBean @ApplicationScoped SusarCriteriaEvaluator` is the only impl
- If it occurs, check that no other bean implements `SusarEvaluatorFunction`

**`@InjectMock AdverseEventEscalationPolicy` not working:**
- Verify `mockito-core` is on the test classpath (it is — existing tests use it)
- Stub in `@BeforeEach` — `@InjectMock` replaces the CDI bean for the entire test class; all tests see the mock

**ConditionTimeout in `SusarActionGateLifecycleTest.gate_approval_allows_case_to_complete`:**
- The gate approval fires `ActionGateApprovedEvent` on the Vert.x event bus — may need a slightly longer timeout
- Increase `atMost(10, SECONDS)` for the final `escalationStatus == COMPLETED` assertion if needed
- This is an async path through the engine; the `IrbGateLifecycleTest` failures may indicate engine CDI wiring issues in the test environment
