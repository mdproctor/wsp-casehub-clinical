# SUSAR Fix and GDPR Compliance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the broken SUSAR oversight binding, add a gate decision audit listener, and wire GDPR consent withdrawal with full Merkle audit trail — closing issues #77, #76, and #7.

**Architecture:** Three-phase `SusarOversightCaseService` observes `AdverseEventReportedEvent` and starts a dedicated `ClinicalSusarOversightCaseHub` case when SUSAR criteria are confirmed, with a DB-keyed `SusarGateDecisionListener` writing tamper-evident ledger entries for all gate outcomes. GDPR consent withdrawal pseudonymizes the external patient identifier and erasures ledger actor tokenization.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, casehub-engine worker binding via `getDefinition()` programmatic augmentation, Vert.x event bus `@ConsumeEvent`, casehub-ledger JOINED inheritance.

---

## Pre-reading

Before any task: read `CLAUDE.md` in the project root for the three-phase pattern, XA requirements, test `application.properties` conventions, and `@ConsumeEvent(blocking = true)` rules.

Build commands:
```bash
mvn install -pl api --batch-mode -q                           # when api changes
mvn test -pl runtime --batch-mode                             # full runtime suite
mvn test -pl runtime -Dtest=ClassName --batch-mode            # single class
```

---

## File Map

### Phase 0 — Spec + rescue
- Modify: `docs/specs/2026-06-12-susar-fix-gdpr-design.md`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java`

### Phase 1 — Issue #77
- Create: `api/src/main/java/io/casehub/clinical/api/model/SusarOversightStatus.java`
- Create: `runtime/src/main/resources/clinical/susar-oversight.yaml`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalSusarOversightCaseHub.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/resources/db/migration/default/V119__ae_susar_oversight_fields.sql`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightCaseService.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightStatusUpdater.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarOversightLifecycleTest.java`

### Phase 2 — Issue #76
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/SusarDecisionLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2021__susar_decision_ledger_entry.sql`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarDecisionLedgerWriter.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventLedgerWriter.java` (+ 5 others)
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarGateDecisionListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarGateDecisionListenerTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarOversightApprovedLifecycleTest.java`

### Phase 3 — Issue #7
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/ConsentWithdrawalLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2022__consent_withdrawal_ledger_entry.sql`
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java`
- Create: `runtime/src/main/resources/db/migration/default/V120__patient_enrollment_withdrawn_at.sql`
- Create: `runtime/src/main/java/io/casehub/clinical/service/ConsentWithdrawalService.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/ConsentWithdrawalServiceTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/PatientAuditResourceTest.java`

---

## Phase 0: Spec corrections and issue-47 rescue

### Task 1: Fix the spec document — three corrections

**Files:**
- Modify: `docs/specs/2026-06-12-susar-fix-gdpr-design.md`

The spec has three errors found in review. Fix them directly in the document.

- [ ] **Step 1.1: Fix the YAML structure**

Replace the `susar-oversight.yaml` block. The binding schema (`io.casehub.model.Binding`) has no `worker:` field — it is silently ignored by Jackson. Capabilities are defined in `spec.capabilities` and referenced directly on bindings. `inputSchema` is JQ (not mini-DSL).

```yaml
dsl: "0.1"
version: "1.0.0"
name: susar-oversight
namespace: clinical
title: SUSAR Expedited Safety Report Oversight Gate

spec:

  capabilities:
    - name: safety-monitoring
      inputSchema: "{ aeId: .aeId }"

  goals:
    - name: susar-complete
      kind: success
      condition: ".susarAssessmentComplete != null"

  bindings:
    - name: susar-assessment
      on:
        contextChange:
          filter: ".aeId != null and .susarAssessmentComplete == null"
      capability: safety-monitoring
```

Update the YAML field notes accordingly:
- `inputSchema` is on the top-level `spec.capabilities` entry, not on the binding
- `capability: safety-monitoring` is a direct field on the binding (not nested under `worker:`)
- Worker output is still written directly by `WorkflowExecutionCompletedHandler` via deferred output — unchanged

- [ ] **Step 1.2: Fix `ClinicalSusarOversightCaseHub`**

Replace the bare class with the augmented form. Without `.function(susarEvaluator)`, the engine has no Java function to dispatch to when the `safety-monitoring` capability is scheduled — this causes a null-function dispatch failure, not a startup error.

```java
@ApplicationScoped
public class ClinicalSusarOversightCaseHub extends YamlCaseHub {

    @Inject SusarEvaluatorFunction susarEvaluator;
    private volatile CaseDefinition augmentedDefinition;

    public ClinicalSusarOversightCaseHub() { super("clinical/susar-oversight.yaml"); }

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
                                    .inputSchema("{ aeId: .aeId }")
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

- [ ] **Step 1.3: Add `COMPLETED` to `SusarOversightStatus`**

Update the enum definition in the spec:

```java
public enum SusarOversightStatus {
    /** Default — Grade 1/2 or non-SUSAR AEs. */
    NONE,
    /** SUSAR criteria met; oversight case started. */
    REQUESTED,
    /** Oversight case goal satisfied. */
    COMPLETED,
    /** Case start failed. */
    FAILED
}
```

Add a note under Phase 1 testing: "A `SusarOversightListener` observes `CaseLifecycleEvent` (matching `GoalReached`) and calls `SusarOversightStatusUpdater.markCompleted(aeId)` in `REQUIRES_NEW` — same pattern as `AeStatusUpdater` / `AeEscalationListener`."

- [ ] **Step 1.4: Commit the spec fixes**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add docs/specs/2026-06-12-susar-fix-gdpr-design.md
git -C /Users/mdproctor/claude/casehub/clinical commit -m "docs(spec): fix YAML binding structure, ClinicalSusarOversightCaseHub augmentation, add SusarOversightStatus.COMPLETED — Refs #77"
```

---

### Task 2: Cherry-pick SusarCriteriaEvaluator fixes from issue-47

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java`

Three items from `issue-47-action-risk-classifier` that haven't landed on main.

- [ ] **Step 2.1: Apply the SusarCriteriaEvaluator code review fixes**

Open `runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java`. Apply these four changes:

1. Change `GATE_GRADES` from `Set<String>` to `Set<CtcaeGrade>`:
```java
private static final Set<CtcaeGrade> GATE_GRADES = Set.of(
        CtcaeGrade.GRADE_4, CtcaeGrade.GRADE_5);
```

2. Add UUID parse guard before `findById`:
```java
final UUID aeId;
try {
    aeId = UUID.fromString(aeIdStr);
} catch (IllegalArgumentException e) {
    return noGate();
}
final AdverseEvent ae = AdverseEvent.findById(aeId);
```

3. Add `suspected` to action context:
```java
actionCtx.put("suspected", ae.suspected);
```

4. Extract `noGate()` helper to eliminate the three duplicate returns:
```java
private static WorkerResult noGate() {
    return WorkerResult.of(Map.of("susarRequired", false, "susarAssessmentComplete", true));
}
```

Also add `import java.util.HashMap;` (the `new HashMap<>()` in the actionCtx construction needs it explicitly).

- [ ] **Step 2.2: Update SusarCriteriaEvaluatorTest**

Replace `runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.WorkerResult;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Unit tests for the defensive no-DB paths in SusarCriteriaEvaluator.
 *
 * <p>Note: {@code @Transactional} on {@code apply()} is a CDI interceptor — it has no
 * effect when the class is instantiated directly here (bypasses CDI). These tests
 * exercise only the early-return paths that do NOT call {@code AdverseEvent.findById()}.
 * Gate-positive and DB-loading paths are covered by SusarActionGateLifecycleTest.
 */
class SusarCriteriaEvaluatorTest {

    private final SusarCriteriaEvaluator evaluator = new SusarCriteriaEvaluator();

    @Test
    void null_aeId_returns_no_gate_with_assessment_complete() {
        WorkerResult result = evaluator.apply(Map.of());
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).containsEntry("susarRequired", false);
        assertThat(result.output()).containsEntry("susarAssessmentComplete", true);
    }

    @Test
    void malformed_aeId_returns_no_gate() {
        WorkerResult result = evaluator.apply(Map.of("aeId", "not-a-uuid"));
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).containsEntry("susarRequired", false);
        assertThat(result.output()).containsEntry("susarAssessmentComplete", true);
    }
}
```

- [ ] **Step 2.3: Create SusarActionGateLifecycleTest**

Create `runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.WorkerResult;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.ClinicalActionType;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.AdverseEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Integration test — SusarCriteriaEvaluator entity loading via @Transactional.
 *
 * <p>Verifies the evaluator correctly loads AdverseEvent from DB and evaluates criteria.
 * The full engine gate lifecycle through SusarOversightCaseService is covered
 * by SusarOversightLifecycleTest and SusarOversightApprovedLifecycleTest.
 */
@QuarkusTest
class SusarActionGateLifecycleTest {

    @Inject SusarCriteriaEvaluator evaluator;

    @Test
    @Transactional
    void grade4_unexpected_suspected_returns_planned_action() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, true, true);
        WorkerResult result = evaluator.apply(Map.of("aeId", aeId.toString()));
        assertThat(result.plannedAction()).isNotNull();
        assertThat(result.plannedAction().actionType())
                .isEqualTo(ClinicalActionType.SUSAR_CRITERIA_DECISION.actionType());
        assertThat(result.output()).containsEntry("susarRequired", true);
        assertThat(result.output()).containsEntry("susarAssessmentComplete", true);
    }

    @Test
    @Transactional
    void grade5_unexpected_suspected_returns_planned_action() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_5, true, true);
        WorkerResult result = evaluator.apply(Map.of("aeId", aeId.toString()));
        assertThat(result.plannedAction()).isNotNull();
        assertThat(result.output()).containsEntry("susarRequired", true);
    }

    @Test
    @Transactional
    void grade4_not_unexpected_returns_no_gate() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, false, true);
        WorkerResult result = evaluator.apply(Map.of("aeId", aeId.toString()));
        assertThat(result.plannedAction()).isNull();
        assertThat(result.output()).containsEntry("susarRequired", false);
    }

    @Test
    @Transactional
    void grade3_unexpected_returns_no_gate() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_3, true, true);
        WorkerResult result = evaluator.apply(Map.of("aeId", aeId.toString()));
        assertThat(result.plannedAction()).isNull();
    }

    @Test
    @Transactional
    void grade4_suspected_false_returns_no_gate() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, true, false);
        WorkerResult result = evaluator.apply(Map.of("aeId", aeId.toString()));
        assertThat(result.plannedAction()).isNull();
    }

    @Test
    void missing_aeId_returns_no_gate() {
        WorkerResult result = evaluator.apply(Map.of());
        assertThat(result.plannedAction()).isNull();
    }

    @Test
    void valid_uuid_not_in_db_returns_no_gate() {
        WorkerResult result = evaluator.apply(Map.of("aeId", UUID.randomUUID().toString()));
        assertThat(result.plannedAction()).isNull();
    }

    private UUID persistAe(CtcaeGrade grade, boolean unexpected, boolean suspected) {
        UUID aeId = UUID.randomUUID();
        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.unexpected = unexpected;
        ae.suspected = suspected;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "test-tenant";
        ae.persist();
        return aeId;
    }
}
```

- [ ] **Step 2.4: Compile and run the evaluator tests**

```bash
mvn install -pl api --batch-mode -q && mvn test -pl runtime -Dtest="SusarCriteriaEvaluatorTest,SusarActionGateLifecycleTest" --batch-mode
```

Expected: all tests pass.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarCriteriaEvaluatorTest.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarActionGateLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "fix(service,test): rescue SusarCriteriaEvaluator code review fixes + evaluator tests from issue-47 — Refs #77"
```

---

### Task 3: Close issue-47 branch

- [ ] **Step 3.1: Stamp the branch as closed**

```bash
git -C /Users/mdproctor/claude/casehub/clinical checkout issue-47-action-risk-classifier
git -C /Users/mdproctor/claude/casehub/clinical commit --allow-empty -m "chore: branch closed"
git -C /Users/mdproctor/claude/casehub/clinical checkout issue-76-susar-fix-gdpr
```

- [ ] **Step 3.2: Close GitHub issue #47**

```bash
gh issue close 47 --repo casehubio/clinical --comment "All deliverables rescued to issue-76-susar-fix-gdpr. ClinicalAdverseEventCaseHub augmentation superseded by ClinicalSusarOversightCaseHub. ae-escalation binding superseded by susar-oversight.yaml. Evaluator code review fixes and tests cherry-picked."
```

---

## Phase 1: Issue #77 — SUSAR Oversight Case Hub

### Task 4: SusarOversightStatus enum

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/SusarOversightStatus.java`

Modelled on `AeEscalationStatus` in the same package. The `COMPLETED` state is set by `SusarOversightListener` when `GoalReached` fires. Without it, `REQUESTED` is ambiguous between "running" and "done."

- [ ] **Step 4.1: Create the enum**

```java
package io.casehub.clinical.api.model;

public enum SusarOversightStatus {
    /** Default — Grade 1/2 or criteria not met; no oversight case. */
    NONE,
    /** SUSAR criteria confirmed; oversight case start requested. */
    REQUESTED,
    /** Oversight case goal satisfied (gate approved or rejected). */
    COMPLETED,
    /** Case start failed — engine unavailable or pool timeout. */
    FAILED
}
```

- [ ] **Step 4.2: Build api**

```bash
mvn install -pl api --batch-mode -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add api/src/main/java/io/casehub/clinical/api/model/SusarOversightStatus.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(api): SusarOversightStatus enum — NONE/REQUESTED/COMPLETED/FAILED — Refs #77"
```

---

### Task 5: susar-oversight.yaml and ClinicalSusarOversightCaseHub

**Files:**
- Create: `runtime/src/main/resources/clinical/susar-oversight.yaml`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalSusarOversightCaseHub.java`

The YAML must have `spec.capabilities` so the mapper can populate `capabilityMap`. The binding references the capability by name directly (not nested under `worker:`). The `getDefinition()` override registers the Java function — without this, the engine cannot dispatch to `SusarCriteriaEvaluator`.

- [ ] **Step 5.1: Create susar-oversight.yaml**

```yaml
dsl: "0.1"
version: "1.0.0"
name: susar-oversight
namespace: clinical
title: SUSAR Expedited Safety Report Oversight Gate

spec:

  capabilities:
    - name: safety-monitoring
      inputSchema: "{ aeId: .aeId }"

  goals:
    - name: susar-complete
      kind: success
      condition: ".susarAssessmentComplete != null"

  bindings:
    - name: susar-assessment
      on:
        contextChange:
          filter: ".aeId != null and .susarAssessmentComplete == null"
      capability: safety-monitoring
```

- [ ] **Step 5.2: Replace ClinicalSusarOversightCaseHub**

Full file content:

```java
package io.casehub.clinical.service;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.Worker;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

/**
 * Case definition for SUSAR expedited safety report oversight (Layer 8).
 *
 * <p>Augments the YAML definition with the {@link SusarCriteriaEvaluator} worker,
 * registered against the {@code safety-monitoring} capability. The worker fires once
 * per case on valid {@code aeId} in context, evaluates SUSAR criteria from the
 * persisted entity, and returns a {@link io.casehub.api.spi.PlannedAction} requiring
 * qualified-investigator sign-off.
 *
 * <p>CDI displacement: annotate a replacement evaluator with {@code @ApplicationScoped}
 * (no {@code @DefaultBean}) to displace the rule-based default automatically.
 */
@ApplicationScoped
public class ClinicalSusarOversightCaseHub extends YamlCaseHub {

    @Inject SusarEvaluatorFunction susarEvaluator;
    private volatile CaseDefinition augmentedDefinition;

    public ClinicalSusarOversightCaseHub() { super("clinical/susar-oversight.yaml"); }

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
                                    .inputSchema("{ aeId: .aeId }")
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

- [ ] **Step 5.3: Compile to verify YAML parses and class compiles**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/resources/clinical/susar-oversight.yaml \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalSusarOversightCaseHub.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service,yaml): ClinicalSusarOversightCaseHub + susar-oversight.yaml — Refs #77"
```

---

### Task 6: AdverseEvent entity changes and V119 migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/resources/db/migration/default/V119__ae_susar_oversight_fields.sql`

- [ ] **Step 6.1: Add fields and named query to AdverseEvent**

Add after the existing `suspected` field:

```java
@Enumerated(EnumType.STRING)
@Column(name = "susar_oversight_status", nullable = false)
public SusarOversightStatus susarOversightStatus = SusarOversightStatus.NONE;

@Column(name = "susar_oversight_case_id")
public UUID susarOversightCaseId;
```

Add import: `import io.casehub.clinical.api.model.SusarOversightStatus;`

Add named query method after `findByIdForTenant`:

```java
public static AdverseEvent findBySusarOversightCaseId(UUID caseId) {
    return find("susarOversightCaseId", caseId).firstResult();
}
```

This query needs no tenant scope — engine case IDs are globally unique UUIDs.

- [ ] **Step 6.2: Create V119 migration**

```sql
ALTER TABLE adverse_event ADD COLUMN susar_oversight_status VARCHAR(20) NOT NULL DEFAULT 'NONE';
ALTER TABLE adverse_event ADD COLUMN susar_oversight_case_id UUID;
```

- [ ] **Step 6.3: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java \
  runtime/src/main/resources/db/migration/default/V119__ae_susar_oversight_fields.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(entity,flyway): AdverseEvent susar oversight fields + findBySusarOversightCaseId + V119 — Refs #77"
```

---

### Task 7: SusarOversightCaseService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightCaseService.java`

Three-phase pattern (identical structure to `AeEscalationCaseService`). Phase 1 is `@Transactional`; Phase 2 (engine call) has no transaction; Phase 3 is `@Transactional`. The `susarOversight: true` key in the initial context is the discriminator used by `SusarOversightListener`.

- [ ] **Step 7.1: Create the service**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.api.spi.SusarEvaluatorFunction;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Observes AdverseEventReportedEvent concurrently with AeEscalationCaseService.
 * Starts a SUSAR oversight case when the evaluator confirms criteria are met.
 *
 * <p>Three-phase pattern keeps startCase().join() outside any @Transactional boundary
 * to avoid deadlocking the Agroal pool (same as TrialActivationService).
 */
@ApplicationScoped
public class SusarOversightCaseService {

    private static final Logger LOG = Logger.getLogger(SusarOversightCaseService.class);

    @Inject ClinicalSusarOversightCaseHub susarOversightCaseHub;
    @Inject SusarEvaluatorFunction susarEvaluator;

    public void onAdverseEventReported(@ObservesAsync AdverseEventReportedEvent event) {
        try {
            Map<String, Object> initialContext = prepareAndMark(event);
            if (initialContext == null) return;
            UUID caseId = susarOversightCaseHub.startCase(initialContext).toCompletableFuture().join();
            persistCaseId(event.aeId(), caseId);
        } catch (Exception e) {
            LOG.errorf(e, "SusarOversightCaseService: oversight case failed for aeId=%s", event.aeId());
            try {
                markFailed(event.aeId());
            } catch (Exception ex) {
                LOG.errorf(ex, "SusarOversightCaseService: markFailed also failed for aeId=%s", event.aeId());
            }
        }
    }

    @Transactional
    Map<String, Object> prepareAndMark(AdverseEventReportedEvent event) {
        AdverseEvent ae = AdverseEvent.findById(event.aeId());
        if (ae == null) {
            LOG.warnf("SusarOversightCaseService: AE not found for aeId=%s — skipping", event.aeId());
            return null;
        }
        // Idempotency guard — protects against CDI at-least-once re-delivery
        if (ae.susarOversightStatus != SusarOversightStatus.NONE) {
            LOG.debugf("SusarOversightCaseService: aeId=%s already processed (status=%s) — skipping", event.aeId(), ae.susarOversightStatus);
            return null;
        }
        // Criteria check via SPI — honours CDI displacement for ML-based evaluator
        var result = susarEvaluator.apply(Map.of("aeId", ae.id.toString()));
        boolean susarRequired = Boolean.TRUE.equals(result.output().get("susarRequired"));
        if (!susarRequired) {
            LOG.debugf("SusarOversightCaseService: SUSAR criteria not met for aeId=%s — no oversight case", event.aeId());
            return null;
        }
        ae.susarOversightStatus = SusarOversightStatus.REQUESTED;
        // context map for oversight case
        Map<String, Object> ctx = new HashMap<>();
        ctx.put("aeId", ae.id.toString());
        ctx.put("grade", ae.grade.name());
        ctx.put("unexpected", ae.unexpected);
        ctx.put("suspected", ae.suspected);
        ctx.put("enrollmentId", ae.enrollmentId.toString());
        ctx.put("siteId", event.siteId().toString());
        ctx.put("tenantId", ae.tenantId);
        ctx.put("susarOversight", true); // discriminator for SusarOversightListener
        return ctx;
    }

    @Transactional
    void persistCaseId(UUID aeId, UUID caseId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            LOG.warnf("SusarOversightCaseService: AE not found in Phase 3 for aeId=%s", aeId);
            return;
        }
        ae.susarOversightCaseId = caseId;
    }

    @Transactional
    void markFailed(UUID aeId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) return;
        ae.susarOversightStatus = SusarOversightStatus.FAILED;
    }
}
```

- [ ] **Step 7.2: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarOversightCaseService.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): SusarOversightCaseService — three-phase observer with idempotency guard — Refs #77"
```

---

### Task 8: SusarOversightStatusUpdater and SusarOversightListener

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightStatusUpdater.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarOversightListener.java`

Modelled exactly on `AeStatusUpdater` and `AeEscalationListener`. The listener discriminates by `susarOversight` key in case context — set by `SusarOversightCaseService` in the initial context map.

- [ ] **Step 8.1: Create SusarOversightStatusUpdater**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Writes SusarOversightStatus.COMPLETED to AdverseEvent in REQUIRES_NEW.
 * Separate from SusarOversightListener so the Panache call can be mocked
 * in unit tests, and the status write commits independently.
 */
@ApplicationScoped
public class SusarOversightStatusUpdater {

    private static final Logger LOG = Logger.getLogger(SusarOversightStatusUpdater.class);

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public boolean markCompleted(UUID aeId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            LOG.warnf("SusarOversightStatusUpdater: AE not found for aeId=%s", aeId);
            return false;
        }
        if (ae.susarOversightStatus == SusarOversightStatus.COMPLETED) {
            LOG.debugf("SusarOversightStatusUpdater: aeId=%s already COMPLETED — skipping", aeId);
            return false;
        }
        ae.susarOversightStatus = SusarOversightStatus.COMPLETED;
        return true;
    }
}
```

- [ ] **Step 8.2: Create SusarOversightListener**

```java
package io.casehub.clinical.service;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Observes CaseLifecycleEvent and marks SUSAR oversight cases COMPLETED.
 *
 * <p>Discriminates by {@code susarOversight} key in case context — set by
 * SusarOversightCaseService in the initial context. GoalReached fires reliably
 * in-memory tests (engine#393); CaseCompleted is also accepted as a fallback.
 */
@ApplicationScoped
public class SusarOversightListener {

    private static final Logger LOG = Logger.getLogger(SusarOversightListener.class);
    private static final Duration LOOKUP_TIMEOUT = Duration.ofSeconds(5);

    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject SusarOversightStatusUpdater statusUpdater;

    @Transactional
    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;

        var instance = caseInstanceRepository
                .findByUuid(event.caseId(), event.tenancyId())
                .await().atMost(LOOKUP_TIMEOUT);
        if (instance == null) return;

        // Discriminator — only SUSAR oversight cases carry this key
        if (instance.getCaseContext().getPath("susarOversight") == null) return;

        Object aeIdObj = instance.getCaseContext().getPath("aeId");
        if (aeIdObj == null) {
            LOG.warnf("SusarOversightListener: susarOversight case has no aeId: caseId=%s", event.caseId());
            return;
        }
        UUID aeId;
        try {
            aeId = UUID.fromString(aeIdObj.toString());
        } catch (IllegalArgumentException e) {
            LOG.warnf("SusarOversightListener: invalid aeId in case context: %s", aeIdObj);
            return;
        }

        boolean first = statusUpdater.markCompleted(aeId);
        if (!first) return;
        LOG.infof("SusarOversightListener: susarOversightStatus set to COMPLETED for aeId=%s caseId=%s", aeId, event.caseId());
    }
}
```

- [ ] **Step 8.3: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 8.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarOversightStatusUpdater.java \
  runtime/src/main/java/io/casehub/clinical/service/SusarOversightListener.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): SusarOversightStatusUpdater + SusarOversightListener — COMPLETED lifecycle — Refs #77"
```

---

### Task 9: Tests for Issue #77

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarOversightLifecycleTest.java`

Unit tests via `@InjectMocks` are incompatible with Panache (same reason `AeEscalationCaseServiceTest` is empty). All tests are `@QuarkusTest`.

- [ ] **Step 9.1: Create SusarOversightLifecycleTest**

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Integration test for SusarOversightCaseService three-phase lifecycle.
 * Tests idempotency guard, criteria gate, and successful case start.
 */
@QuarkusTest
class SusarOversightLifecycleTest {

    @Inject SusarOversightCaseService service;

    @Test
    @Transactional
    void grade4_unexpected_suspected_starts_oversight_case() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, true, true);
        AdverseEventReportedEvent event = new AdverseEventReportedEvent(
                aeId, UUID.randomUUID(), UUID.randomUUID(), CtcaeGrade.GRADE_4);

        service.onAdverseEventReported(event);

        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            AdverseEvent ae = AdverseEvent.findById(aeId);
            // status is REQUESTED immediately after Phase 1; COMPLETED after case goal fires
            assertThat(ae.susarOversightStatus).isIn(
                    SusarOversightStatus.REQUESTED, SusarOversightStatus.COMPLETED);
            assertThat(ae.susarOversightCaseId).isNotNull();
        });
    }

    @Test
    @Transactional
    void grade2_does_not_start_oversight_case() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_2, false, true);
        AdverseEventReportedEvent event = new AdverseEventReportedEvent(
                aeId, UUID.randomUUID(), UUID.randomUUID(), CtcaeGrade.GRADE_2);

        service.onAdverseEventReported(event);

        AdverseEvent ae = AdverseEvent.findById(aeId);
        assertThat(ae.susarOversightStatus).isEqualTo(SusarOversightStatus.NONE);
        assertThat(ae.susarOversightCaseId).isNull();
    }

    @Test
    @Transactional
    void idempotency_guard_prevents_double_start() {
        UUID aeId = persistAe(CtcaeGrade.GRADE_4, true, true);
        AdverseEvent ae = AdverseEvent.findById(aeId);
        ae.susarOversightStatus = SusarOversightStatus.REQUESTED; // simulate already processed
        AdverseEventReportedEvent event = new AdverseEventReportedEvent(
                aeId, UUID.randomUUID(), UUID.randomUUID(), CtcaeGrade.GRADE_4);

        service.onAdverseEventReported(event);

        // susarOversightCaseId stays null — no second case was started
        assertThat(AdverseEvent.<AdverseEvent>findById(aeId).susarOversightCaseId).isNull();
    }

    private UUID persistAe(CtcaeGrade grade, boolean unexpected, boolean suspected) {
        UUID aeId = UUID.randomUUID();
        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = grade;
        ae.unexpected = unexpected;
        ae.suspected = suspected;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "default";
        ae.persist();
        return aeId;
    }
}
```

- [ ] **Step 9.2: Run the test**

```bash
mvn test -pl runtime -Dtest=SusarOversightLifecycleTest --batch-mode
```

Expected: all three tests pass.

- [ ] **Step 9.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/SusarOversightLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(service): SusarOversightLifecycleTest — three-phase, idempotency, criteria gate — Refs #77"
```

---

## Phase 2: Issue #76 — SUSAR Gate Decision Listener

### Task 10: SusarDecisionLedgerEntry and V2021

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/SusarDecisionLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2021__susar_decision_ledger_entry.sql`

Must live in `io.casehub.clinical.ledger` (not `entity`) — Panache cannot span two persistence units. `subjectId = enrollmentId` so the entry appears in `LedgerProvExportService.exportSubject(enrollmentId, tenancyId)`.

- [ ] **Step 10.1: Create SusarDecisionLedgerEntry**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record for SUSAR oversight gate decisions (approved, rejected, expired).
 * FDA IND / EU AI Act Art.12 requirement: clinician gate decisions on SUSAR criteria
 * must be independently verifiable. JOINED inheritance on qhorus datasource. V2021.
 *
 * {@code subjectId = enrollmentId} — required for LedgerProvExportService.exportSubject()
 * to include this entry in the patient's PROV-DM audit export.
 */
@Entity
@Table(name = "susar_decision_ledger_entry")
@DiscriminatorValue("SusarDecision")
public class SusarDecisionLedgerEntry extends LedgerEntry {

    @Column(name = "ae_id", nullable = false)
    public UUID aeId;

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Column(name = "ctcae_grade", length = 20)
    public String ctcaeGrade;

    @Column(name = "gate_outcome", nullable = false, length = 20)
    public String gateOutcome;

    @Column(name = "decided_at", nullable = false)
    public Instant decidedAt;

    @Column(name = "decided_by", length = 255)
    public String decidedBy;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                aeId       != null ? aeId.toString()       : "",
                enrollmentId != null ? enrollmentId.toString() : "",
                ctcaeGrade != null ? ctcaeGrade            : "",
                gateOutcome != null ? gateOutcome          : "",
                decidedAt  != null ? decidedAt.toString()  : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 10.2: Create V2021 migration**

```sql
CREATE TABLE susar_decision_ledger_entry (
    id              UUID          NOT NULL,
    ae_id           UUID          NOT NULL,
    enrollment_id   UUID          NOT NULL,
    ctcae_grade     VARCHAR(20),
    gate_outcome    VARCHAR(20)   NOT NULL,
    decided_at      TIMESTAMP     NOT NULL,
    decided_by      VARCHAR(255),
    CONSTRAINT pk_susar_decision_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_susar_decision_ledger_entry_base FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 10.3: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 10.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/SusarDecisionLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V2021__susar_decision_ledger_entry.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(ledger,flyway): SusarDecisionLedgerEntry + V2021 — Refs #76"
```

---

### Task 11: ClinicalComplianceSupplement and SusarDecisionLedgerWriter

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarDecisionLedgerWriter.java`

`ClinicalComplianceSupplement` centralizes supplement construction — no free-form strings in individual writers. It must live in `runtime/` not `api/` — it references GCP/FDA strings, not consumed downstream.

To find the correct `ComplianceSupplement` builder API, look at the class via IntelliJ MCP before implementing: `mcp__intellij-index__ide_find_definition` on `io.casehub.ledger.runtime.model.supplement.ComplianceSupplement`.

- [ ] **Step 11.1: Inspect ComplianceSupplement builder API**

Before writing the factory, check the actual builder methods on `ComplianceSupplement` from the ledger JAR. Use IntelliJ MCP to read the decompiled class. The key fields confirmed in the spec are `planRef`, `algorithmRef`, and `humanOverrideAvailable`.

- [ ] **Step 11.2: Create ClinicalComplianceSupplement**

```java
package io.casehub.clinical.service;

import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;

/**
 * Factory for EU AI Act Art.12 compliance supplements attached to AI agent decision entries.
 * Centralises all GCP/FDA reference strings — no free-form strings in individual writers.
 */
public final class ClinicalComplianceSupplement {

    private ClinicalComplianceSupplement() {}

    public static ComplianceSupplement aeEscalation() {
        return ComplianceSupplement.builder()
                .planRef("ICH E6(R3) §5.17 — serious adverse event reporting")
                .algorithmRef("AdverseEventEscalationPolicy (rule-based CTCAE routing)")
                .humanOverrideAvailable(true)
                .build();
    }

    public static ComplianceSupplement irbDecision() {
        return ComplianceSupplement.builder()
                .planRef("21 CFR Part 312 — IRB review and approval")
                .algorithmRef("IrbCommitteePolicySpi (configurable IRB routing)")
                .humanOverrideAvailable(true)
                .build();
    }

    public static ComplianceSupplement protocolDeviation() {
        return ComplianceSupplement.builder()
                .planRef("ICH E6(R3) §4.5 — protocol deviation recording")
                .algorithmRef("ProtocolDeviationService (rule-based severity classification)")
                .humanOverrideAvailable(true)
                .build();
    }

    public static ComplianceSupplement susarGateDecision() {
        return ComplianceSupplement.builder()
                .planRef("ICH E2A §I.A.1 + 21 CFR 312.32 — SUSAR criteria and expedited reporting")
                .algorithmRef("SusarCriteriaEvaluator (rule-based CTCAE Grade 4/5 unexpected/suspected)")
                .humanOverrideAvailable(true)
                .build();
    }

    public static ComplianceSupplement safetyOfficerNotification() {
        return ComplianceSupplement.builder()
                .planRef("ICH E6(R3) §5.17 — safety officer notification on Grade 3+ AE")
                .algorithmRef("AdverseEventEscalationPolicy (rule-based CTCAE routing)")
                .humanOverrideAvailable(true)
                .build();
    }

    public static ComplianceSupplement sponsorNotification() {
        return ComplianceSupplement.builder()
                .planRef("ICH E6(R3) §5.17 — sponsor notification on serious adverse event")
                .algorithmRef("AdverseEventEscalationPolicy (rule-based CTCAE routing)")
                .humanOverrideAvailable(true)
                .build();
    }
}
```

**Note:** If `ComplianceSupplement.builder()` uses a different API (e.g., constructor with fields or a static factory), adjust to match. The IntelliJ MCP inspection from step 11.1 determines the correct API.

- [ ] **Step 11.3: Create SusarDecisionLedgerWriter**

Look at `AdverseEventLedgerWriter` as the reference pattern for sequence number logic.

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.ledger.SusarDecisionLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.LedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Clock;
import java.time.Instant;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Writes tamper-evident ledger entries for all three SUSAR gate outcomes.
 * Each write is REQUIRES_NEW — commits independently of surrounding transaction state.
 */
@ApplicationScoped
public class SusarDecisionLedgerWriter {

    private static final Logger LOG = Logger.getLogger(SusarDecisionLedgerWriter.class);

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Clock clock;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeEntry(AdverseEvent ae, String gateOutcome, Instant decidedAt, String decidedBy) {
        SusarDecisionLedgerEntry entry = new SusarDecisionLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = ae.enrollmentId;  // patient-scoped for PROV export
        entry.sequenceNumber = nextSequenceNumber(ae.enrollmentId);
        entry.entryType = LedgerEntryType.DECISION;
        entry.actorId = decidedBy != null ? decidedBy : ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = decidedBy != null ? ActorType.HUMAN : ActorType.SYSTEM;
        entry.actorRole = "SusarOversightGate";
        entry.occurredAt = clock.instant();
        entry.aeId = ae.id;
        entry.enrollmentId = ae.enrollmentId;
        entry.ctcaeGrade = ae.grade != null ? ae.grade.name() : null;
        entry.gateOutcome = gateOutcome;
        entry.decidedAt = decidedAt;
        entry.decidedBy = decidedBy;
        entry.addSupplement(ClinicalComplianceSupplement.susarGateDecision());
        ledgerEntryRepository.save(entry, "default");
        LOG.infof("SusarDecisionLedgerWriter: wrote %s entry for enrollmentId=%s aeId=%s",
                gateOutcome, ae.enrollmentId, ae.id);
    }

    private int nextSequenceNumber(UUID enrollmentId) {
        return ledgerEntryRepository.findLatestBySubjectId(enrollmentId, "default")
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

**Note:** `entry.addSupplement(...)` — verify the exact method name for attaching a `ComplianceSupplement` to a `LedgerEntry` by reading `LedgerEntry` from the ledger JAR via IntelliJ MCP before committing.

- [ ] **Step 11.4: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 11.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java \
  runtime/src/main/java/io/casehub/clinical/service/SusarDecisionLedgerWriter.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): ClinicalComplianceSupplement + SusarDecisionLedgerWriter — Refs #76"
```

---

### Task 12: Update existing six LedgerWriter beans

**Files (all in `runtime/.../service/`):**
- Modify: `AdverseEventLedgerWriter.java`
- Modify: `AeEscalationLedgerWriter.java`
- Modify: `DeviationLedgerWriter.java`
- Modify: `IrbLedgerWriter.java`
- Modify: `SafetyOfficerNotificationLedgerWriter.java`
- Modify: `SponsorNotificationLedgerWriter.java`

Find the exact writer class names first:

```bash
ls /Users/mdproctor/claude/casehub/clinical/runtime/src/main/java/io/casehub/clinical/service/*LedgerWriter*
```

For each writer, find the primary decision entry write method and add `.addSupplement(ClinicalComplianceSupplement.xxx())` using the appropriate factory method. Use IntelliJ MCP `ide_find_references` on each `LedgerWriter` class to confirm which factory method matches.

- [ ] **Step 12.1: Read all six writers**

Use IntelliJ MCP to read each file. Identify the method that creates the primary `LedgerEntry` for that writer (the decision entry, not a failure or completion entry).

- [ ] **Step 12.2: Add supplement to each primary entry**

For each writer, after `entry.entryType = LedgerEntryType.DECISION;` (or the analogous primary entry construction), add:

```java
entry.addSupplement(ClinicalComplianceSupplement.<method>());
```

where `<method>` is the matching factory method:
- `AdverseEventLedgerWriter` → `aeEscalation()`
- `AeEscalationLedgerWriter` → `aeEscalation()`
- `DeviationLedgerWriter` → `protocolDeviation()`
- `IrbLedgerWriter` → `irbDecision()`
- `SafetyOfficerNotificationLedgerWriter` → `safetyOfficerNotification()`
- `SponsorNotificationLedgerWriter` → `sponsorNotification()`

- [ ] **Step 12.3: Compile and run affected tests**

```bash
mvn test -pl runtime -Dtest="AdverseEventLedgerWriterTest,IrbLedgerWriterTest,SafetyOfficerNotificationIntegrationTest" --batch-mode
```

Expected: all pass — supplement is attached, existing assertions are unaffected.

- [ ] **Step 12.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add runtime/src/main/java/io/casehub/clinical/service/
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): attach ClinicalComplianceSupplement to all six LedgerWriter primary entries — Refs #76"
```

---

### Task 13: SusarGateDecisionListener

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/SusarGateDecisionListener.java`

All three `@ConsumeEvent` methods are `blocking = true` — they do DB work and must not block the Vert.x event loop. The DB discriminator `AdverseEvent.findBySusarOversightCaseId(event.caseId())` is race-free (persisted in Phase 3 before the gate WorkItem is created).

For the rejected/expired paths, `signal()` is called twice. The second call (`susarRequired`) may arrive at an already-completed case after the first (`susarAssessmentComplete`). The engine discards signals to terminal cases with a WARN log — this is benign.

- [ ] **Step 13.1: Create SusarGateDecisionListener**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ClinicalActors;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.quarkus.vertx.ConsumeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import org.jboss.logging.Logger;

/**
 * Writes ledger entries for all SUSAR oversight gate outcomes.
 *
 * <p>Gate discrimination uses DB query (AdverseEvent.findBySusarOversightCaseId) — not
 * CaseInstanceCache, which is racy: the engine's gate handlers clear pendingActionGate
 * before the clinical listener sees the event. The DB approach is race-free and survives
 * JVM restart (engine#433).
 *
 * <p>For rejected/expired: signals both susarAssessmentComplete and susarRequired to the
 * oversight case. The second signal may arrive at an already-completed case; the engine
 * discards it with a WARN log — this is benign.
 */
@ApplicationScoped
public class SusarGateDecisionListener {

    private static final Logger LOG = Logger.getLogger(SusarGateDecisionListener.class);

    @Inject ClinicalSusarOversightCaseHub susarOversightCaseHub;
    @Inject SusarDecisionLedgerWriter ledgerWriter;

    @ConsumeEvent(value = "casehub.action.gate.approved", blocking = true)
    public void onApproved(ActionGateApprovedEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return; // not a SUSAR oversight gate
        LOG.infof("SusarGateDecisionListener: gate APPROVED caseId=%s aeId=%s", event.caseId(), ae.id);
        // No case signalling — engine's ActionGateApprovedHandler calls refireCompletion(),
        // which writes deferred worker output (susarAssessmentComplete: true) via
        // WorkflowExecutionCompletedHandler, satisfying the susar-complete goal.
        ledgerWriter.writeEntry(ae, "APPROVED", Instant.now(), event.approvedBy());
    }

    @ConsumeEvent(value = "casehub.action.gate.rejected", blocking = true)
    public void onRejected(ActionGateRejectedEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return;
        LOG.infof("SusarGateDecisionListener: gate REJECTED caseId=%s aeId=%s", event.caseId(), ae.id);
        susarOversightCaseHub.signal(event.caseId(), "susarAssessmentComplete", true);
        susarOversightCaseHub.signal(event.caseId(), "susarRequired", false);
        ledgerWriter.writeEntry(ae, "REJECTED", Instant.now(), event.rejectedBy());
    }

    @ConsumeEvent(value = "casehub.action.gate.expired", blocking = true)
    public void onExpired(ActionGateExpiredEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return;
        LOG.infof("SusarGateDecisionListener: gate EXPIRED caseId=%s aeId=%s", event.caseId(), ae.id);
        susarOversightCaseHub.signal(event.caseId(), "susarAssessmentComplete", true);
        susarOversightCaseHub.signal(event.caseId(), "susarRequired", false);
        ledgerWriter.writeEntry(ae, "EXPIRED", Instant.now(), ClinicalActors.CLINICAL_SERVICE);
    }
}
```

- [ ] **Step 13.2: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 13.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/SusarGateDecisionListener.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): SusarGateDecisionListener — DB discriminator, three outcomes — Refs #76"
```

---

### Task 14: Tests for Issue #76

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarGateDecisionListenerTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/SusarOversightApprovedLifecycleTest.java`

Pattern: `SafetyOfficerNotificationIntegrationTest` — call listener methods directly; no `Event.fireAsync()`.

- [ ] **Step 14.1: Create SusarGateDecisionListenerTest**

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.reset;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

/**
 * Tests SusarGateDecisionListener using direct method invocation.
 * Pattern: SafetyOfficerNotificationIntegrationTest.
 */
@QuarkusTest
class SusarGateDecisionListenerTest {

    @Inject SusarGateDecisionListener listener;
    @InjectMock SusarDecisionLedgerWriter ledgerWriter;
    @InjectMock ClinicalSusarOversightCaseHub caseHub;

    @BeforeEach
    void reset() {
        reset(ledgerWriter, caseHub);
    }

    @Test
    @Transactional
    void approved_writes_ledger_entry_and_does_not_signal() {
        UUID caseId = UUID.randomUUID();
        AdverseEvent ae = persistAe(caseId);

        listener.onApproved(new ActionGateApprovedEvent(caseId, 1L, null, "dr-smith"));

        verify(ledgerWriter).writeEntry(Mockito.argThat(a -> a.id.equals(ae.id)),
                Mockito.eq("APPROVED"), Mockito.any(Instant.class), Mockito.eq("dr-smith"));
        verifyNoInteractions(caseHub);
    }

    @Test
    @Transactional
    void rejected_signals_case_and_writes_ledger_entry() {
        UUID caseId = UUID.randomUUID();
        AdverseEvent ae = persistAe(caseId);

        listener.onRejected(new ActionGateRejectedEvent(caseId, 1L, null, "dr-jones"));

        verify(caseHub).signal(caseId, "susarAssessmentComplete", true);
        verify(caseHub).signal(caseId, "susarRequired", false);
        verify(ledgerWriter).writeEntry(Mockito.argThat(a -> a.id.equals(ae.id)),
                Mockito.eq("REJECTED"), Mockito.any(Instant.class), Mockito.eq("dr-jones"));
    }

    @Test
    @Transactional
    void expired_signals_case_and_writes_ledger_entry_with_system_actor() {
        UUID caseId = UUID.randomUUID();
        AdverseEvent ae = persistAe(caseId);

        listener.onExpired(new ActionGateExpiredEvent(caseId, 1L));

        verify(caseHub).signal(caseId, "susarAssessmentComplete", true);
        verify(caseHub).signal(caseId, "susarRequired", false);
        verify(ledgerWriter).writeEntry(Mockito.argThat(a -> a.id.equals(ae.id)),
                Mockito.eq("EXPIRED"), Mockito.any(Instant.class),
                Mockito.eq("clinical-service"));
    }

    @Test
    void non_susar_gate_is_silently_ignored() {
        UUID unknownCaseId = UUID.randomUUID(); // no AE has this as susarOversightCaseId

        listener.onRejected(new ActionGateRejectedEvent(unknownCaseId, 1L, null, "dr-jones"));

        verifyNoInteractions(ledgerWriter, caseHub);
    }

    @Transactional
    AdverseEvent persistAe(UUID susarOversightCaseId) {
        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_4;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "default";
        ae.susarOversightStatus = SusarOversightStatus.REQUESTED;
        ae.susarOversightCaseId = susarOversightCaseId;
        ae.persist();
        return ae;
    }
}
```

- [ ] **Step 14.2: Create SusarOversightApprovedLifecycleTest**

This is the engine lifecycle test for the approved path. The engine's `ActionGateApprovedHandler` calls `refireCompletion()`, which writes deferred worker output and satisfies the goal. Pattern: `IrbGateLifecycleTest` approved path.

Check `IrbGateLifecycleTest` before writing this test — the setup pattern for triggering the full approved-path engine cycle is already established there. Adapt it for the SUSAR oversight case hub and the `susar-complete` goal.

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.api.model.SusarOversightStatus;
import io.casehub.clinical.entity.AdverseEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Approved-path lifecycle test: oversight case starts → gate created →
 * WorkItem approved → engine re-fires WorkflowExecutionCompleted →
 * susarAssessmentComplete written → susar-complete goal satisfied → case terminal.
 *
 * Pattern: IrbGateLifecycleTest approved path. Check that test before editing this one.
 */
@QuarkusTest
class SusarOversightApprovedLifecycleTest {

    @Inject SusarOversightCaseService service;

    @Test
    @Transactional
    void approved_gate_completes_oversight_case() {
        // Arrange — this test drives the engine directly; check IrbGateLifecycleTest
        // for the exact WorkItem approval mechanism used in this project.
        UUID aeId = persistAe();
        // TODO: complete this test following the IrbGateLifecycleTest approved-path pattern.
        // The assertion is: after WorkItem completion, susarOversightStatus = COMPLETED.
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            AdverseEvent ae = AdverseEvent.findById(aeId);
            assertThat(ae.susarOversightStatus).isEqualTo(SusarOversightStatus.COMPLETED);
        });
    }

    @Transactional
    UUID persistAe() {
        UUID aeId = UUID.randomUUID();
        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = UUID.randomUUID();
        ae.grade = CtcaeGrade.GRADE_4;
        ae.unexpected = true;
        ae.suspected = true;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.tenantId = "default";
        ae.persist();
        return aeId;
    }
}
```

**Important:** Before writing the full test body, read `IrbGateLifecycleTest` completely. The WorkItem approval mechanism (how `WorkItem.complete()` is triggered in test context) is already established there — copy that mechanism rather than inventing a new one.

- [ ] **Step 14.3: Run the listener test**

```bash
mvn test -pl runtime -Dtest=SusarGateDecisionListenerTest --batch-mode
```

Expected: all four tests pass.

- [ ] **Step 14.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/SusarGateDecisionListenerTest.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarOversightApprovedLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(service): SusarGateDecisionListenerTest + SusarOversightApprovedLifecycleTest — Refs #76"
```

---

## Phase 3: Issue #7 — GDPR and Regulatory Compliance

### Task 15: ConsentWithdrawalLedgerEntry and V2022

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/ledger/ConsentWithdrawalLedgerEntry.java`
- Create: `runtime/src/main/resources/db/migration/qhorus/V2022__consent_withdrawal_ledger_entry.sql`

- [ ] **Step 15.1: Create ConsentWithdrawalLedgerEntry**

```java
package io.casehub.clinical.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

/**
 * Tamper-evident record for GDPR Art.17 consent withdrawal.
 *
 * {@code actorId = enrollmentId.toString()} at write time; LedgerErasureService
 * pseudonymizes this field post-erasure. UUID-only domainContentBytes ensures
 * the Merkle chain survives erasure. JOINED inheritance on qhorus datasource. V2022.
 */
@Entity
@Table(name = "consent_withdrawal_ledger_entry")
@DiscriminatorValue("ConsentWithdrawal")
public class ConsentWithdrawalLedgerEntry extends LedgerEntry {

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Column(name = "withdrawn_at", nullable = false)
    public Instant withdrawnAt;

    @Column(name = "ledger_entries_affected", nullable = false)
    public long ledgerEntriesAffected = 0L;

    @Column(name = "memories_erased", nullable = false)
    public boolean memoriesErased = false;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                enrollmentId != null ? enrollmentId.toString() : "",
                withdrawnAt  != null ? withdrawnAt.toString()  : "")
                .getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 15.2: Create V2022 migration**

```sql
CREATE TABLE consent_withdrawal_ledger_entry (
    id                      UUID        NOT NULL,
    enrollment_id           UUID        NOT NULL,
    withdrawn_at            TIMESTAMP   NOT NULL,
    ledger_entries_affected BIGINT      NOT NULL DEFAULT 0,
    memories_erased         BOOLEAN     NOT NULL DEFAULT FALSE,
    CONSTRAINT pk_consent_withdrawal_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_consent_withdrawal_ledger_entry_base FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 15.3: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 15.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/ConsentWithdrawalLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V2022__consent_withdrawal_ledger_entry.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(ledger,flyway): ConsentWithdrawalLedgerEntry + V2022 — Refs #7"
```

---

### Task 16: PatientEnrollment.withdrawnAt and V120

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java`
- Create: `runtime/src/main/resources/db/migration/default/V120__patient_enrollment_withdrawn_at.sql`

`EnrollmentStatus.WITHDRAWN` already exists in the enum — no enum change needed.

- [ ] **Step 16.1: Add withdrawnAt to PatientEnrollment**

Add after `enrolledAt`:

```java
@Column(name = "withdrawn_at")
public Instant withdrawnAt;
```

- [ ] **Step 16.2: Create V120**

```sql
ALTER TABLE patient_enrollment ADD COLUMN withdrawn_at TIMESTAMP;
```

- [ ] **Step 16.3: Compile and commit**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java \
  runtime/src/main/resources/db/migration/default/V120__patient_enrollment_withdrawn_at.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(entity,flyway): PatientEnrollment.withdrawnAt + V120 — Refs #7"
```

---

### Task 17: ConsentWithdrawalService

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ConsentWithdrawalService.java`

**XA REQUIRED:** this service writes to both datasources in one `@Transactional`. `quarkus.datasource.jdbc.transactions=xa` and `quarkus.datasource.qhorus.jdbc.transactions=xa` must be set. Check production `application.properties` — they are already set for `ProtocolDeviationService`. Without XA, Agroal throws "Failed to enlist" on every call.

`LedgerErasureService.erase()` requires `casehub.ledger.identity.tokenisation.enabled=true` in both `application.properties` and test `application.properties` — without it `erase()` always returns `affectedEntryCount=0`.

Step order for the entry: write entry with `ledgerEntriesAffected=0` first (creating the ActorIdentity mapping via tokenization), then call `erase()` to get the count, then update the entry. This avoids a separate "count before write" that would undercount by 1.

- [ ] **Step 17.1: Create ConsentWithdrawalService**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.api.model.LedgerEntryType;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.ledger.ConsentWithdrawalLedgerEntry;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.service.LedgerEntryRepository;
import io.casehub.platform.api.memory.MemoryStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.core.Response;
import java.time.Clock;
import java.time.Instant;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * GDPR Art.17 consent withdrawal — pseudonymizes patientId, tokenizes actorId in ledger,
 * erases memories keyed to the enrollment.
 *
 * XA required: writes PatientEnrollment (default datasource) + ConsentWithdrawalLedgerEntry
 * (qhorus datasource) in one @Transactional. quarkus.datasource.jdbc.transactions=xa must
 * be set on both datasources. casehub.ledger.identity.tokenisation.enabled=true is also
 * required for LedgerErasureService.erase() to pseudonymize actorId entries.
 */
@ApplicationScoped
public class ConsentWithdrawalService {

    private static final Logger LOG = Logger.getLogger(ConsentWithdrawalService.class);

    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject LedgerErasureService ledgerErasureService;
    @Inject MemoryStore memoryStore;
    @Inject Clock clock;

    @Transactional
    public Response withdraw(UUID enrollmentId, String tenantId) {
        PatientEnrollment enrollment = PatientEnrollment.find(
                "id = ?1 AND tenantId = ?2", enrollmentId, tenantId).firstResult();
        if (enrollment == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        if (enrollment.consentStatus == ConsentStatus.WITHDRAWN) {
            return Response.status(Response.Status.CONFLICT)
                    .entity("Consent already withdrawn").build();
        }

        Instant withdrawnAt = clock.instant();

        // Pseudonymize external patient identifier
        enrollment.consentStatus = ConsentStatus.WITHDRAWN;
        enrollment.enrollmentStatus = EnrollmentStatus.WITHDRAWN;
        enrollment.withdrawnAt = withdrawnAt;
        enrollment.patientId = "erased-" + UUID.randomUUID();
        enrollment.persist();

        // Write ledger entry — creates ActorIdentity mapping for enrollmentId.toString()
        ConsentWithdrawalLedgerEntry entry = buildEntry(enrollmentId, tenantId, withdrawnAt);
        ledgerEntryRepository.save(entry, "default");

        // Pseudonymize actorId in all ledger entries where actorId = token(enrollmentId)
        var erasureResult = ledgerErasureService.erase(enrollmentId.toString());
        LOG.infof("ConsentWithdrawalService: erased enrollmentId=%s mappingFound=%s affectedEntries=%d",
                enrollmentId, erasureResult.mappingFound(), erasureResult.affectedEntryCount());

        // Update the entry with the erasure count — re-fetch by id (same TX, same connection)
        ConsentWithdrawalLedgerEntry persisted = ledgerEntryRepository
                .findEntryById(entry.id, "default")
                .map(e -> (ConsentWithdrawalLedgerEntry) e)
                .orElse(entry);
        persisted.ledgerEntriesAffected = erasureResult.affectedEntryCount();

        // Erase patient-specific memories (GDPR Art.5(2))
        boolean memoriesErased = memoryStore.eraseEntity(enrollmentId, tenantId);
        persisted.memoriesErased = memoriesErased;

        return Response.noContent().build();
    }

    private ConsentWithdrawalLedgerEntry buildEntry(UUID enrollmentId, String tenantId, Instant withdrawnAt) {
        ConsentWithdrawalLedgerEntry entry = new ConsentWithdrawalLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = enrollmentId;
        entry.sequenceNumber = nextSequenceNumber(enrollmentId);
        entry.entryType = io.casehub.ledger.api.model.LedgerEntryType.EVENT;
        entry.actorId = enrollmentId.toString(); // tokenized at save; erased by LedgerErasureService
        entry.actorType = ActorType.HUMAN;
        entry.actorRole = "PatientWithdrawal";
        entry.occurredAt = withdrawnAt;
        entry.enrollmentId = enrollmentId;
        entry.withdrawnAt = withdrawnAt;
        return entry;
    }

    private int nextSequenceNumber(UUID enrollmentId) {
        return ledgerEntryRepository.findLatestBySubjectId(enrollmentId, "default")
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

**Note:** `memoryStore.eraseEntity(UUID, String)` — verify this exact method exists on `MemoryStore` by reading the interface via IntelliJ MCP before committing. The method signature may differ.

- [ ] **Step 17.2: Verify test application.properties has tokenisation enabled**

Check `runtime/src/test/resources/application.properties`. If `casehub.ledger.identity.tokenisation.enabled` is not present, add:
```properties
casehub.ledger.identity.tokenisation.enabled=true
```

- [ ] **Step 17.3: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 17.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/ConsentWithdrawalService.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(service): ConsentWithdrawalService — GDPR Art.17 with ledger erasure and memory wipe — Refs #7"
```

---

### Task 18: PatientResource — three new endpoints

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`

Three new endpoints. `LedgerProvExportService.exportSubject()` and `LedgerVerificationService.inclusionProof()` both throw `IllegalArgumentException` for not-found — no global ExceptionMapper exists, must catch explicitly.

- [ ] **Step 18.1: Add injections and endpoints to PatientResource**

Add to the class fields:

```java
@Inject ConsentWithdrawalService consentWithdrawalService;
@Inject LedgerProvExportService ledgerProvExportService;
@Inject LedgerVerificationService ledgerVerificationService;
```

Add three endpoint methods:

```java
@POST
@Path("/{enrollmentId}/withdraw-consent")
public Response withdrawConsent(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    TrialSite site = TrialSite.findByIdForTenant(siteId, principal);
    if (site == null || !site.trialId.equals(trialId))
        return Response.status(Response.Status.NOT_FOUND).build();
    return consentWithdrawalService.withdraw(enrollmentId, principal.tenancyId());
}

@GET
@Path("/{enrollmentId}/audit/prov")
@Produces("application/ld+json")
public Response getAuditProv(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId) {
    // Verify tenant scope before delegating
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    try {
        String jsonLd = ledgerProvExportService.exportSubject(enrollmentId, principal.tenancyId());
        return Response.ok(jsonLd).build();
    } catch (IllegalArgumentException e) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
}

@GET
@Path("/{enrollmentId}/audit/entries/{entryId}/proof")
public Response getMerkleProof(
        @PathParam("trialId") UUID trialId,
        @PathParam("siteId") UUID siteId,
        @PathParam("enrollmentId") UUID enrollmentId,
        @PathParam("entryId") UUID entryId) {
    PatientEnrollment enrollment = PatientEnrollment.findByIdForTenant(enrollmentId, principal);
    if (enrollment == null || !enrollment.siteId.equals(siteId))
        return Response.status(Response.Status.NOT_FOUND).build();
    try {
        var proof = ledgerVerificationService.inclusionProof(entryId, principal.tenancyId());
        return Response.ok(proof).build();
    } catch (IllegalArgumentException e) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
}
```

Add required imports for `LedgerProvExportService`, `LedgerVerificationService`, and `ConsentWithdrawalService`.

- [ ] **Step 18.2: Compile**

```bash
mvn compile -pl runtime --batch-mode 2>&1 | grep -E "BUILD|ERROR"
```

- [ ] **Step 18.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(resource): withdraw-consent, audit/prov, audit/entries/proof endpoints — Refs #7"
```

---

### Task 19: GDPR tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/ConsentWithdrawalServiceTest.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/PatientAuditResourceTest.java`

- [ ] **Step 19.1: Create ConsentWithdrawalServiceTest**

```java
package io.casehub.clinical.service;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.entity.ClinicalTrial;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class ConsentWithdrawalServiceTest {

    @Inject ConsentWithdrawalService service;

    @Test
    @Transactional
    void withdraw_sets_both_statuses_pseudonymizes_patientId_and_sets_withdrawnAt() {
        UUID enrollmentId = UUID.randomUUID();
        String originalPatientId = "patient-mrn-12345";
        persistEnrollment(enrollmentId, originalPatientId);

        var response = service.withdraw(enrollmentId, "default");

        assertThat(response.getStatus()).isEqualTo(204);
        PatientEnrollment updated = PatientEnrollment.findById(enrollmentId);
        assertThat(updated.consentStatus).isEqualTo(ConsentStatus.WITHDRAWN);
        assertThat(updated.enrollmentStatus).isEqualTo(EnrollmentStatus.WITHDRAWN);
        assertThat(updated.patientId).startsWith("erased-");
        assertThat(updated.patientId).doesNotContain(originalPatientId);
        assertThat(updated.withdrawnAt).isNotNull();
    }

    @Test
    @Transactional
    void withdraw_returns_409_if_already_withdrawn() {
        UUID enrollmentId = UUID.randomUUID();
        persistEnrollment(enrollmentId, "patient-xyz");
        PatientEnrollment enrollment = PatientEnrollment.findById(enrollmentId);
        enrollment.consentStatus = ConsentStatus.WITHDRAWN;

        var response = service.withdraw(enrollmentId, "default");

        assertThat(response.getStatus()).isEqualTo(409);
    }

    @Test
    void withdraw_returns_404_for_unknown_enrollment() {
        var response = service.withdraw(UUID.randomUUID(), "default");
        assertThat(response.getStatus()).isEqualTo(404);
    }

    @Transactional
    void persistEnrollment(UUID enrollmentId, String patientId) {
        PatientEnrollment e = new PatientEnrollment();
        e.id = enrollmentId;
        e.siteId = UUID.randomUUID();
        e.patientId = patientId;
        e.tenantId = "default";
        e.consentStatus = ConsentStatus.PENDING;
        e.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        e.persist();
    }
}
```

- [ ] **Step 19.2: Create PatientAuditResourceTest**

```java
package io.casehub.clinical.resource;

import static io.restassured.RestAssured.given;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.entity.PatientEnrollment;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class PatientAuditResourceTest {

    @Test
    void prov_endpoint_returns_404_for_enrollment_with_no_ledger_entries() {
        UUID trialId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        UUID enrollmentId = persistEnrollment(siteId);

        given()
            .header("X-Tenant-Id", "default")
        .when()
            .get("/trials/{t}/sites/{s}/patients/{e}/audit/prov", trialId, siteId, enrollmentId)
        .then()
            .statusCode(404);
    }

    @Test
    void merkle_proof_endpoint_returns_404_for_unknown_entry() {
        UUID trialId = UUID.randomUUID();
        UUID siteId = UUID.randomUUID();
        UUID enrollmentId = persistEnrollment(siteId);
        UUID unknownEntryId = UUID.randomUUID();

        given()
            .header("X-Tenant-Id", "default")
        .when()
            .get("/trials/{t}/sites/{s}/patients/{e}/audit/entries/{id}/proof",
                    trialId, siteId, enrollmentId, unknownEntryId)
        .then()
            .statusCode(404);
    }

    @Transactional
    UUID persistEnrollment(UUID siteId) {
        PatientEnrollment e = new PatientEnrollment();
        e.id = UUID.randomUUID();
        e.siteId = siteId;
        e.patientId = "test-patient";
        e.tenantId = "default";
        e.consentStatus = ConsentStatus.PENDING;
        e.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        e.persist();
        return e.id;
    }
}
```

- [ ] **Step 19.3: Run GDPR tests**

```bash
mvn test -pl runtime -Dtest="ConsentWithdrawalServiceTest,PatientAuditResourceTest" --batch-mode
```

Expected: all pass.

- [ ] **Step 19.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/ConsentWithdrawalServiceTest.java \
  runtime/src/test/java/io/casehub/clinical/resource/PatientAuditResourceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "test(service,resource): ConsentWithdrawalServiceTest + PatientAuditResourceTest — Refs #7"
```

---

## Final: Full suite and branch close

### Task 20: Full test suite

- [ ] **Step 20.1: Run the complete test suite**

```bash
mvn test --batch-mode 2>&1 | tail -30
```

Expected: `BUILD SUCCESS`. If any test fails, diagnose before proceeding — do not commit a broken suite.

- [ ] **Step 20.2: Verify issue-47 branch is stamped closed**

```bash
git -C /Users/mdproctor/claude/casehub/clinical log -1 --format="%s" issue-47-action-risk-classifier
```

Expected: `chore: branch closed`

### Task 21: Doc sync and work-end

- [ ] **Step 21.1: Run implementation-doc-sync**

Invoke the `implementation-doc-sync` skill to sweep CLAUDE.md and any affected docs for drift introduced by this session (new CDI wiring rules, new migration version conventions, new test patterns).

- [ ] **Step 21.2: Run work-end**

Invoke the `work-end` skill to close `issue-76-susar-fix-gdpr`, promote artifacts, and file the GitHub issues as closed.

---

## Self-review against spec

| Spec section | Task |
|---|---|
| YAML binding structure fix | Task 1.1, Task 5.1 |
| ClinicalSusarOversightCaseHub augmented | Task 1.2, Task 5.2 |
| SusarOversightStatus COMPLETED | Task 1.3, Task 4 |
| SusarCriteriaEvaluator code fixes | Task 2.1 |
| SusarActionGateLifecycleTest | Task 2.3 |
| issue-47 closed | Task 3 |
| SusarOversightCaseService three-phase + idempotency | Task 7 |
| SusarOversightListener + statusUpdater | Task 8 |
| AdverseEvent.findBySusarOversightCaseId | Task 6.1 |
| V119 migration | Task 6.2 |
| SusarDecisionLedgerEntry subjectId=enrollmentId + FK | Task 10 |
| ClinicalComplianceSupplement 6 methods | Task 11.2 |
| SusarDecisionLedgerWriter REQUIRES_NEW | Task 11.3 |
| 6 existing writers updated | Task 12 |
| SusarGateDecisionListener DB discriminator | Task 13 |
| Gate listener tests (approved/rejected/expired/non-SUSAR) | Task 14.1 |
| Approved-path lifecycle test | Task 14.2 |
| ConsentWithdrawalLedgerEntry FK + subjectId=enrollmentId | Task 15 |
| PatientEnrollment.withdrawnAt + V120 | Task 16 |
| ConsentWithdrawalService XA + ErasureResult capture | Task 17 |
| withdraw-consent endpoint | Task 18 |
| PROV audit endpoint + 404 handling | Task 18 |
| Merkle proof endpoint + 404 handling | Task 18 |
| GDPR tests | Task 19 |
| casehub.ledger.identity.tokenisation.enabled in test props | Task 17.2 |
