# Engine SPI Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix CaseLifecycleEvent internal import (#28), add AeEscalationStatus + engine_case_id fields (#27/#26), introduce IrbCommitteeAssignmentPolicy SPI (#29), and add LAYER-LOG build note (#39) — sharing a three-phase service refactor across issues #27/#26.

**Architecture:** The three-phase pattern (same as `TrialActivationService`) splits `@Transactional` boundaries so `startCase().join()` never holds a DB connection. Phase 1 marks REQUESTED + builds context, Phase 2 starts the engine case, Phase 3 persists the case ID. A catch block around Phase 2–3 calls Phase 4 (`markFailed`). All new test assertions fold into existing `@QuarkusTest` lifecycle tests.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache Active Record, JUnit 5, Mockito, AssertJ, Awaitility

**Spec:** `docs/specs/2026-05-28-engine-spi-cleanup-design.md`

---

## File Map

**Create:**
- `api/src/main/java/io/casehub/clinical/api/model/AeEscalationStatus.java`
- `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeContext.java`
- `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignment.java`
- `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignmentPolicy.java`
- `runtime/src/main/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicy.java`
- `runtime/src/main/resources/db/migration/default/V111__ae_escalation_status.sql`
- `runtime/src/main/resources/db/migration/default/V112__ae_engine_case_id.sql`
- `runtime/src/main/resources/db/migration/default/V113__protocol_deviation_engine_case_id.sql`
- `runtime/src/test/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicyTest.java`

**Modify:**
- `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` — import + COMPLETED write-back
- `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — three-phase refactor
- `runtime/src/main/java/io/casehub/clinical/service/IrbDeviationCaseService.java` — three-phase refactor + SPI
- `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java` — two new fields
- `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java` — one new field
- `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java` — import fix + arity
- `runtime/src/test/java/io/casehub/clinical/service/AeEscalationCaseServiceTest.java` — delete Panache-incompatible tests
- `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java` — @BeforeEach entities + new assertions
- `runtime/src/test/java/io/casehub/clinical/service/IrbDeviationCaseServiceTest.java` — add `@Mock committeePolicy`
- `runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java` — @BeforeEach entities + new assertions
- `LAYER-LOG.md` — vertical slice build note

---

## Task 1: Fix CaseLifecycleEvent imports and constructor arity (#28)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java`

- [ ] **Step 1: Update AeEscalationListener import**

In `AeEscalationListener.java`, replace line:
```java
import io.casehub.engine.internal.event.CaseLifecycleEvent;
```
with:
```java
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
```

- [ ] **Step 2: Update AeEscalationListenerTest imports and constructor call**

In `AeEscalationListenerTest.java`, replace:
```java
import io.casehub.engine.internal.event.CaseLifecycleEvent;
import io.casehub.engine.internal.model.CaseInstance;
```
with:
```java
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.engine.common.internal.model.CaseInstance;
```

Then find the constructor call (it will be 6-arg) and add `null` as the 7th argument (`traceId`). The new record has 7 components: `(UUID caseId, String commandType, String eventType, String caseStatus, String actorId, String actorRole, String traceId)`. In the test:
```java
listener.onCaseLifecycle(new CaseLifecycleEvent(
        caseId, "CompleteCase", "CaseCompleted", "COMPLETED", "system", "system", null));
```

- [ ] **Step 3: Run clean compile (mandatory per GE-20260526-43a51d)**

```bash
mvn clean test-compile -pl api,runtime --batch-mode
```
Expected: `BUILD SUCCESS`. If it fails on `CaseLifecycleEvent(...)` with "wrong number of arguments", the arity fix in Step 2 is incomplete — verify all constructor call sites.

- [ ] **Step 4: Run AeEscalationListenerTest**

```bash
mvn test -pl runtime -Dtest=AeEscalationListenerTest --batch-mode
```
Expected: `Tests run: N, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "fix(#28): update CaseLifecycleEvent to public SPI package; fix 7-arg arity

CaseLifecycleEvent moved from engine.internal.event to engine.common.spi.event
in engine#378. CaseInstance moved to engine.common.internal.model.
CaseLifecycleEvent gained traceId (7th component) — add null in test constructor.

Closes #28
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: LAYER-LOG vertical slice build note (#39)

**Files:**
- Modify: `LAYER-LOG.md`

- [ ] **Step 1: Add build approach paragraph**

In `LAYER-LOG.md`, find the line:
```
Entries are ordered for learning, not chronology. Each entry is complete when the layer closes — no placeholders.
```
Insert this paragraph immediately after it (before the blank line that follows):
```
**Build approach:** Layer ordering here is for teaching, not building. The recommended
pattern is vertical slice first — the thinnest working path through all layers — then
deepen each layer to production completeness. See `../parent/docs/AGENTIC-HARNESS-GUIDE.md`
§Build Order. Any layers built before this guidance existed are accurate history; future
layer work follows vertical slice first.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add LAYER-LOG.md
git -C /Users/mdproctor/claude/casehub/clinical commit -m "docs(#39): add vertical slice build approach note to LAYER-LOG

Closes #39
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: AeEscalationStatus enum (#27)

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/AeEscalationStatus.java`

- [ ] **Step 1: Create the enum**

```java
package io.casehub.clinical.api.model;

public enum AeEscalationStatus {
    /** Default — Grade 1/2 AEs; no escalation initiated. */
    NONE,
    /** Grade 3+; escalation case started. */
    REQUESTED,
    /** Engine case reached CaseCompleted with safety review. */
    COMPLETED,
    /** Case start failed (engine unavailable, pool timeout). */
    FAILED
}
```

- [ ] **Step 2: Build api module**

```bash
mvn install -pl api --batch-mode
```
Expected: `BUILD SUCCESS`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/model/AeEscalationStatus.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#27): add AeEscalationStatus enum (NONE/REQUESTED/COMPLETED/FAILED)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: AdverseEvent entity fields + Flyway migrations V111/V112 (#27/#26)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/resources/db/migration/default/V111__ae_escalation_status.sql`
- Create: `runtime/src/main/resources/db/migration/default/V112__ae_engine_case_id.sql`

- [ ] **Step 1: Add fields to AdverseEvent**

In `AdverseEvent.java`, add this import at the top with the other imports:
```java
import io.casehub.clinical.api.model.AeEscalationStatus;
```

Then append these two fields after the existing `workItemId` field:
```java
@Enumerated(EnumType.STRING)
@Column(name = "escalation_status", nullable = false)
public AeEscalationStatus escalationStatus = AeEscalationStatus.NONE;

@Column(name = "engine_case_id")
public UUID engineCaseId;
```

- [ ] **Step 2: Create V111 migration**

```sql
-- V111__ae_escalation_status.sql
ALTER TABLE adverse_event ADD COLUMN escalation_status VARCHAR(50) NOT NULL DEFAULT 'NONE';
```

- [ ] **Step 3: Create V112 migration**

```sql
-- V112__ae_engine_case_id.sql
ALTER TABLE adverse_event ADD COLUMN engine_case_id UUID;
```

- [ ] **Step 4: Compile check**

```bash
mvn compile -pl api,runtime --batch-mode
```
Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java \
  runtime/src/main/resources/db/migration/default/V111__ae_escalation_status.sql \
  runtime/src/main/resources/db/migration/default/V112__ae_engine_case_id.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#27,#26): add escalationStatus and engineCaseId to AdverseEvent

V111: escalation_status VARCHAR(50) NOT NULL DEFAULT 'NONE'
V112: engine_case_id UUID (nullable)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: ProtocolDeviation.engineCaseId + V113 migration (#26)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`
- Create: `runtime/src/main/resources/db/migration/default/V113__protocol_deviation_engine_case_id.sql`

- [ ] **Step 1: Add field to ProtocolDeviation**

In `ProtocolDeviation.java`, append after the last existing field:
```java
/** Links this CRITICAL deviation to its IRB review engine case. Null until IrbDeviationCaseService starts the case. */
@Column(name = "engine_case_id")
public UUID engineCaseId;
```

(`UUID` is already imported.)

- [ ] **Step 2: Create V113 migration**

```sql
-- V113__protocol_deviation_engine_case_id.sql
ALTER TABLE protocol_deviation ADD COLUMN engine_case_id UUID;
```

- [ ] **Step 3: Compile check**

```bash
mvn compile -pl api,runtime --batch-mode
```
Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java \
  runtime/src/main/resources/db/migration/default/V113__protocol_deviation_engine_case_id.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#26): add engineCaseId to ProtocolDeviation (V113)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: IrbCommitteeAssignmentPolicy SPI types (#29)

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeContext.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignment.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignmentPolicy.java`

- [ ] **Step 1: Create IrbCommitteeContext**

```java
package io.casehub.clinical.api.spi;

import io.casehub.clinical.api.model.DeviationSeverity;
import java.util.UUID;

/**
 * Context passed to {@link IrbCommitteeAssignmentPolicy#evaluate}.
 * {@code trialId} may be null if the site has no active trial case.
 */
public record IrbCommitteeContext(
        UUID deviationId,
        UUID siteId,
        UUID trialId,
        DeviationSeverity severity) {}
```

- [ ] **Step 2: Create IrbCommitteeAssignment**

```java
package io.casehub.clinical.api.spi;

import java.util.List;

/** Assignment returned by {@link IrbCommitteeAssignmentPolicy#evaluate}. */
public record IrbCommitteeAssignment(String committeeId, List<String> candidateGroups) {}
```

- [ ] **Step 3: Create IrbCommitteeAssignmentPolicy**

```java
package io.casehub.clinical.api.spi;

/**
 * Maps deviation context to an IRB committee assignment.
 * Mirrors {@link DeviationResponsePolicy} — implement as
 * {@code @ApplicationScoped @Alternative @Priority(1)} to override the default.
 */
@FunctionalInterface
public interface IrbCommitteeAssignmentPolicy {
    IrbCommitteeAssignment evaluate(IrbCommitteeContext context);
}
```

- [ ] **Step 4: Build api module**

```bash
mvn install -pl api --batch-mode
```
Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeContext.java \
  api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignment.java \
  api/src/main/java/io/casehub/clinical/api/spi/IrbCommitteeAssignmentPolicy.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#29): IrbCommitteeAssignmentPolicy SPI — context, assignment, interface

Mirrors DeviationResponsePolicy pattern. evaluate() aligns with all other SPIs.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: DefaultIrbCommitteeAssignmentPolicy (TDD) (#29)

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicyTest.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicy.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.spi.IrbCommitteeAssignment;
import io.casehub.clinical.api.spi.IrbCommitteeContext;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultIrbCommitteeAssignmentPolicyTest {

    private final DefaultIrbCommitteeAssignmentPolicy policy = new DefaultIrbCommitteeAssignmentPolicy();

    @Test
    void evaluate_returns_non_null_assignment_for_any_context() {
        IrbCommitteeContext ctx = new IrbCommitteeContext(
                UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), DeviationSeverity.CRITICAL);

        IrbCommitteeAssignment result = policy.evaluate(ctx);

        assertThat(result).isNotNull();
        assertThat(result.committeeId()).isNotBlank();
        assertThat(result.candidateGroups()).isNotEmpty();
    }

    @Test
    void evaluate_returns_irb_committee_default_regardless_of_input() {
        IrbCommitteeContext ctx = new IrbCommitteeContext(
                UUID.randomUUID(), UUID.randomUUID(), null, DeviationSeverity.MINOR);

        IrbCommitteeAssignment result = policy.evaluate(ctx);

        assertThat(result.committeeId()).isEqualTo("irb-committee");
        assertThat(result.candidateGroups()).containsExactly("irb-committee");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
mvn test -pl runtime -Dtest=DefaultIrbCommitteeAssignmentPolicyTest --batch-mode
```
Expected: `COMPILATION ERROR` or `ClassNotFoundException` — class not created yet.

- [ ] **Step 3: Implement DefaultIrbCommitteeAssignmentPolicy**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.spi.IrbCommitteeAssignment;
import io.casehub.clinical.api.spi.IrbCommitteeAssignmentPolicy;
import io.casehub.clinical.api.spi.IrbCommitteeContext;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;

@ApplicationScoped
@DefaultBean
public class DefaultIrbCommitteeAssignmentPolicy implements IrbCommitteeAssignmentPolicy {

    @Override
    public IrbCommitteeAssignment evaluate(IrbCommitteeContext context) {
        return new IrbCommitteeAssignment("irb-committee", List.of("irb-committee"));
    }
}
```

- [ ] **Step 4: Run to verify tests pass**

```bash
mvn test -pl runtime -Dtest=DefaultIrbCommitteeAssignmentPolicyTest --batch-mode
```
Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/test/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicyTest.java \
  runtime/src/main/java/io/casehub/clinical/service/DefaultIrbCommitteeAssignmentPolicy.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#29): DefaultIrbCommitteeAssignmentPolicy @DefaultBean returns irb-committee

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: AeEscalationLifecycleTest — @BeforeEach entity setup

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java`

This must be done BEFORE Task 9 — the entities must exist before the three-phase refactor runs.

- [ ] **Step 1: Add @Transactional to setup() and persist a minimal AdverseEvent**

In `AeEscalationLifecycleTest.java`:

Add these imports (if not already present):
```java
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.transaction.Transactional;
import java.time.Instant;
```

Replace the existing `@BeforeEach void setup()` with:
```java
@BeforeEach
@Transactional
void setup() {
    aeId = UUID.randomUUID();
    enrollmentId = UUID.randomUUID();
    siteId = UUID.randomUUID();
    completionCapture.reset();

    AdverseEvent ae = new AdverseEvent();
    ae.id = aeId;
    ae.enrollmentId = enrollmentId;
    ae.grade = CtcaeGrade.GRADE_3;
    ae.actuality = EventActuality.ACTUAL;
    ae.outcome = AeOutcome.ONGOING;
    ae.occurredAt = Instant.now();
    ae.reportedAt = Instant.now();
    ae.persist();
}
```

Add a `@Transactional` helper method for reading the entity in assertions:
```java
@Transactional
AdverseEvent findAe(UUID id) {
    return AdverseEvent.findById(id);
}
```

- [ ] **Step 2: Run existing tests to confirm they still pass**

```bash
mvn test -pl runtime -Dtest=AeEscalationLifecycleTest --batch-mode
```
Expected: All existing tests pass (entity setup doesn't break current tests since the services haven't been refactored yet).

---

## Task 9: AeEscalationCaseService three-phase refactor (TDD) (#27/#26)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationCaseServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java`

- [ ] **Step 1: Delete Panache-incompatible tests from AeEscalationCaseServiceTest**

After the three-phase refactor, every test in `AeEscalationCaseServiceTest` that calls `service.onAdverseEventReported(event)` will fail because Phase 1 calls `AdverseEvent.findById()` — a Panache static method that requires a Quarkus container. `@InjectMocks` creates a bare instance with no CDI context.

Replace the entire file content with:
```java
package io.casehub.clinical.service;

// Happy-path coverage moved to AeEscalationLifecycleTest (@QuarkusTest) after three-phase
// refactor introduced Panache calls in Phase 1 — incompatible with @InjectMocks.
//
// Signaling coverage gap (grade4 → runtime.signal) tracked as casehubio/clinical#NNN.
// Guard/filter tests are not needed here — AeEscalationCaseService has no early-return guards.
class AeEscalationCaseServiceTest {
}
```

(Replace `#NNN` with the issue number filed in Step 2.)

- [ ] **Step 2: File signaling coverage gap issue**

```bash
gh issue create --repo casehubio/clinical \
  --title "test: AeEscalationCaseService grade4/5 signaling coverage gap" \
  --label "enhancement" \
  --body "After the three-phase refactor (branch issue-28-engine-spi-cleanup), AeEscalationCaseServiceTest was cleared because @InjectMocks can't mock Panache static calls (Phase 1 calls AdverseEvent.findById()). The grade4/grade5 trial case signaling tests (runtime.signal assertions) were lost.

Coverage needed:
- Grade 4 → signalTrialGrade4Active() → runtime.signal() called
- Grade 5 → same
- Grade 3 → no signal

Options:
1. Add quarkus-panache-mock and write @ExtendWith(QuarkusMockAnnotations) tests
2. Add a lifecycle test with a TrialSite + ClinicalTrial + active engine case and verify signal via blackboard state

Refs #28 (three-phase refactor)"
```
Note the issue number from the output.

- [ ] **Step 3: Add REQUESTED and engineCaseId assertions to AeEscalationLifecycleTest**

In `AeEscalationLifecycleTest.java`, add these assertions at the start of `grade3_opens_one_senior_monitor_gate()`, immediately after the `aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_3))` call:

```java
// Phase 1 sets REQUESTED synchronously (direct call, not via CDI event bus)
assertThat(findAe(aeId).escalationStatus).isEqualTo(AeEscalationStatus.REQUESTED);
```

And after the final `await` that verifies `CaseStatus.COMPLETED`, add:
```java
// Phase 3 persists the case ID — verify it's non-null
assertThat(findAe(aeId).engineCaseId).isNotNull();
```

Also add the import:
```java
import io.casehub.clinical.api.model.AeEscalationStatus;
```

- [ ] **Step 4: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=AeEscalationLifecycleTest --batch-mode
```
Expected: `AssertionError` — `escalationStatus` is `NONE` (Phase 1 not yet refactored).

- [ ] **Step 5: Implement AeEscalationCaseService three-phase refactor**

Replace the entire `AeEscalationCaseService.java` with:
```java
package io.casehub.clinical.service;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.clinical.api.AdverseEventReportedEvent;
import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.spi.AdverseEventContext;
import io.casehub.clinical.api.spi.AdverseEventEscalationPolicy;
import io.casehub.clinical.api.spi.AdverseEventEscalationRequirements;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * Observes AdverseEventReportedEvent (Grade 3+ AEs) and starts an AE escalation engine case.
 *
 * <p>Three-phase pattern (same as TrialActivationService) keeps startCase().join() outside any
 * @Transactional boundary to avoid deadlocking the Agroal connection pool. Phase 4 markFailed()
 * fires on any exception from Phase 2–3 to avoid leaving escalationStatus stuck at REQUESTED.
 */
@ApplicationScoped
public class AeEscalationCaseService {

    private static final Logger LOG = Logger.getLogger(AeEscalationCaseService.class);
    private static final Set<CtcaeGrade> SEVERE_GRADES = Set.of(CtcaeGrade.GRADE_4, CtcaeGrade.GRADE_5);

    @Inject ClinicalAdverseEventCaseHub caseHub;
    @Inject AdverseEventEscalationPolicy policy;
    @Inject CaseHubRuntime runtime;
    @Inject TrialCaseLookup trialCaseLookup;

    public void onAdverseEventReported(@ObservesAsync AdverseEventReportedEvent event) {
        try {
            Map<String, Object> initialContext = prepareAndMarkRequested(event);
            if (initialContext == null) return;
            UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
            persistCaseId(event.aeId(), caseId);
            if (SEVERE_GRADES.contains(event.grade())) {
                signalTrialGrade4Active(event.siteId(), true);
            }
        } catch (Exception e) {
            LOG.errorf(e, "AeEscalationCaseService: escalation failed for aeId=%s — marking FAILED", event.aeId());
            markFailed(event.aeId());
        }
    }

    @Transactional
    Map<String, Object> prepareAndMarkRequested(AdverseEventReportedEvent event) {
        AdverseEvent ae = AdverseEvent.findById(event.aeId());
        if (ae == null) {
            LOG.warnf("AeEscalationCaseService: AdverseEvent not found for aeId=%s — skipping escalation", event.aeId());
            return null;
        }
        ae.escalationStatus = AeEscalationStatus.REQUESTED;

        AdverseEventEscalationRequirements requirements = policy.evaluate(
                new AdverseEventContext(event.aeId(), event.enrollmentId(), event.siteId(), event.grade()));

        Map<String, Object> ctx = new HashMap<>();
        ctx.put("aeId", event.aeId().toString());
        ctx.put("enrollmentId", event.enrollmentId().toString());
        ctx.put("siteId", event.siteId().toString());
        ctx.put("grade", event.grade().name());
        ctx.put("requiresSeniorMonitor", requirements.requiresSeniorMonitor());
        ctx.put("requiresDsmbEscalation", requirements.requiresDsmbEscalation());
        return ctx;
    }

    @Transactional
    void persistCaseId(UUID aeId, UUID caseId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            LOG.warnf("AeEscalationCaseService: AdverseEvent not found in Phase 3 for aeId=%s", aeId);
            return;
        }
        ae.engineCaseId = caseId;
    }

    @Transactional
    void markFailed(UUID aeId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            LOG.warnf("AeEscalationCaseService: AdverseEvent not found in markFailed for aeId=%s", aeId);
            return;
        }
        ae.escalationStatus = AeEscalationStatus.FAILED;
    }

    private void signalTrialGrade4Active(UUID siteId, boolean active) {
        UUID trialCaseId = trialCaseLookup.findTrialEngineCase(siteId);
        if (trialCaseId != null) {
            runtime.signal(trialCaseId, "grade4Active." + siteId, active);
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=AeEscalationLifecycleTest --batch-mode
```
Expected: All tests pass including the new REQUESTED and engineCaseId assertions.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationCaseServiceTest.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#27,#26): AeEscalationCaseService three-phase refactor

Phase 1: AdverseEvent.escalationStatus = REQUESTED
Phase 2: startCase().join() outside TX
Phase 3: ae.engineCaseId = caseId
Phase 4: markFailed() on exception

Lifecycle test: assert REQUESTED after start, engineCaseId non-null.

Refs #27, Refs #26
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 10: AeEscalationListener COMPLETED write-back (TDD) (#27)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`

- [ ] **Step 1: Add COMPLETED assertion to grade3 lifecycle test**

In `AeEscalationLifecycleTest.java`, at the very end of `grade3_opens_one_senior_monitor_gate()` (after the existing `CaseStatus.COMPLETED` await), add:

```java
// AeEscalationListener fires @ObservesAsync on CaseLifecycleEvent — small lag after case completes
await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
        .untilAsserted(() ->
                assertThat(findAe(aeId).escalationStatus).isEqualTo(AeEscalationStatus.COMPLETED));
```

- [ ] **Step 2: Run to verify it fails**

```bash
mvn test -pl runtime -Dtest=AeEscalationLifecycleTest#grade3_opens_one_senior_monitor_gate --batch-mode
```
Expected: `ConditionTimeoutException` — COMPLETED not set (listener doesn't write it yet).

- [ ] **Step 3: Implement COMPLETED write-back in AeEscalationListener**

In `AeEscalationListener.java`:

Add these imports:
```java
import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.entity.AdverseEvent;
```

In `onCaseLifecycle()`, after the `aeId = UUID.fromString(...)` block (and its `catch` return), but BEFORE the `enrollmentId` resolution and null-check, insert:

```java
// Write COMPLETED before enrollmentId check — status reflects case completion
// regardless of whether context is complete enough for ledger write.
AdverseEvent aeForStatus = AdverseEvent.findById(aeId);
if (aeForStatus != null) aeForStatus.escalationStatus = AeEscalationStatus.COMPLETED;
```

The method is already `@Transactional`, so no boundary change is needed.

- [ ] **Step 4: Run to verify tests pass**

```bash
mvn test -pl runtime -Dtest=AeEscalationLifecycleTest --batch-mode
```
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#27): AeEscalationListener sets escalationStatus=COMPLETED on CaseCompleted

Status update fires before enrollmentId check — reflects case completion
regardless of context completeness.

Closes #27
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 11: IrbGateLifecycleTest — @BeforeEach entity setup + IrbDeviationCaseServiceTest guard fix

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/IrbDeviationCaseServiceTest.java`

- [ ] **Step 1: Update IrbGateLifecycleTest setup with entity creation**

In `IrbGateLifecycleTest.java`:

Add a `trialId` field:
```java
private UUID trialId;
```

Add these imports:
```java
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.clinical.entity.TrialSite;
```

Replace `@BeforeEach void setup()` with:
```java
@BeforeEach
@Transactional
void setup() {
    deviationId = UUID.randomUUID();
    siteId = UUID.randomUUID();
    trialId = UUID.randomUUID();
    completionCapture.reset();

    TrialSite site = new TrialSite();
    site.id = siteId;
    site.trialId = trialId;
    site.investigatorId = "test-pi";
    site.persist();

    ProtocolDeviation deviation = new ProtocolDeviation();
    deviation.id = deviationId;
    deviation.siteId = siteId;
    deviation.deviationType = "CONSENT_DEVIATION";
    deviation.severity = DeviationSeverity.CRITICAL;
    deviation.piApprovalStatus = PiApprovalStatus.APPROVED;
    deviation.persist();
}
```

Add a `@Transactional` helper method for reading ProtocolDeviation in assertions:
```java
@Transactional
ProtocolDeviation findDeviation(UUID id) {
    return ProtocolDeviation.findById(id);
}
```

Update the stale checkpoint 1 comment in `irb_approved_full_lifecycle()`:
```java
// Checkpoint 1: start IRB case — observer delegates to internal @Transactional phase methods
irbDeviationCaseService.onDeviationResolved(criticalDeviationApproved());
```

- [ ] **Step 2: Add @Mock committeePolicy to IrbDeviationCaseServiceTest**

In `IrbDeviationCaseServiceTest.java`, add:
```java
import io.casehub.clinical.api.spi.IrbCommitteeAssignmentPolicy;
```

And add this mock field (before `@InjectMocks`):
```java
@Mock IrbCommitteeAssignmentPolicy committeePolicy;
```

The existing guard tests don't call `committeePolicy.evaluate()` — they return before Phase 1. Mockito will inject the mock but never call it. No behavior setup needed.

- [ ] **Step 3: Run existing tests to verify they still pass**

```bash
mvn test -pl runtime -Dtest=IrbGateLifecycleTest,IrbDeviationCaseServiceTest --batch-mode
```
Expected: All existing tests pass.

---

## Task 12: IrbDeviationCaseService three-phase refactor + SPI (#26/#29)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/IrbDeviationCaseService.java`

- [ ] **Step 1: Add engineCaseId and committeeId assertions to IrbGateLifecycleTest**

In `irb_approved_full_lifecycle()`, immediately after the `Checkpoint 1` line (after `irbDeviationCaseService.onDeviationResolved(criticalDeviationApproved())`), add:

```java
// Phase 3 persists caseId on ProtocolDeviation — give it a moment to complete
await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS)
        .untilAsserted(() ->
                assertThat(findDeviation(deviationId).engineCaseId).isNotNull());

// DefaultIrbCommitteeAssignmentPolicy sets committeeId = "irb-committee"
assertThat(approvalDecision()).isNotNull(); // IrbApproval persisted
```

Also add a dedicated test for the SPI committeeId:
```java
@Test
@Transactional
void irb_approval_committeeId_matches_policy_default() {
    irbDeviationCaseService.onDeviationResolved(criticalDeviationApproved());

    IrbApproval approval = IrbApproval.find("deviationId = ?1", deviationId).firstResult();
    assertThat(approval).isNotNull();
    assertThat(approval.committeeId).isEqualTo("irb-committee");
}
```

Add these imports:
```java
import io.casehub.clinical.entity.ProtocolDeviation;
```
(if not already present from Task 11)

- [ ] **Step 2: Run to verify tests fail**

```bash
mvn test -pl runtime -Dtest=IrbGateLifecycleTest --batch-mode
```
Expected: Failures on engineCaseId and/or the new committeeId test — service not yet refactored.

- [ ] **Step 3: Implement IrbDeviationCaseService three-phase refactor**

Replace the entire `IrbDeviationCaseService.java` with:
```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.ProtocolDeviationResolvedEvent;
import io.casehub.clinical.api.model.EscalationRequirement;
import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.casehub.clinical.api.spi.IrbCommitteeAssignment;
import io.casehub.clinical.api.spi.IrbCommitteeAssignmentPolicy;
import io.casehub.clinical.api.spi.IrbCommitteeContext;
import io.casehub.clinical.entity.IrbApproval;
import io.casehub.clinical.entity.ProtocolDeviation;
import io.casehub.clinical.entity.TrialSite;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * Observes ProtocolDeviationResolvedEvent and starts an IRB deviation review engine case when
 * the deviation requires IRB review and the PI has approved it.
 *
 * <p>Three-phase pattern: Phase 1 creates IrbApproval + evaluates IrbCommitteeAssignmentPolicy,
 * Phase 2 starts the engine case (outside TX), Phase 3 writes deviation.engineCaseId.
 */
@ApplicationScoped
public class IrbDeviationCaseService {

    private static final Logger LOG = Logger.getLogger(IrbDeviationCaseService.class);

    @Inject ClinicalDeviationCaseHub caseHub;
    @Inject IrbCommitteeAssignmentPolicy committeePolicy;

    public void onDeviationResolved(@ObservesAsync ProtocolDeviationResolvedEvent event) {
        if (event.escalationRequirement() != EscalationRequirement.IRB_REVIEW) return;
        if (event.terminalStatus() != PiApprovalStatus.APPROVED) return;

        try {
            Map<String, Object> initialContext = prepareAndCreateApproval(event);
            UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
            persistDeviationCaseId(event.deviationId(), caseId);
        } catch (Exception e) {
            LOG.errorf(e, "IrbDeviationCaseService: IRB case start failed for deviationId=%s", event.deviationId());
        }
    }

    @Transactional
    Map<String, Object> prepareAndCreateApproval(ProtocolDeviationResolvedEvent event) {
        TrialSite site = TrialSite.findById(event.siteId());
        UUID trialId = site != null ? site.trialId : null;

        IrbCommitteeContext committeeCtx = new IrbCommitteeContext(
                event.deviationId(), event.siteId(), trialId, event.severity());
        IrbCommitteeAssignment assignment = committeePolicy.evaluate(committeeCtx);

        IrbApproval approval = new IrbApproval();
        approval.id = UUID.randomUUID();
        approval.siteId = event.siteId();
        approval.deviationId = event.deviationId();
        approval.reviewType = "PROTOCOL_DEVIATION";
        approval.committeeId = assignment.committeeId();
        approval.decisionDeadline = Instant.now().plus(Duration.ofHours(72));
        approval.decision = IrbDecision.PENDING;
        approval.persist();

        Map<String, Object> ctx = new HashMap<>();
        ctx.put("deviationId", event.deviationId().toString());
        ctx.put("siteId", event.siteId().toString());
        ctx.put("severity", event.severity().name());
        ctx.put("escalationRequirement", event.escalationRequirement().name());
        ctx.put("irbConsultationRequired", true);
        ctx.put("committeeId", assignment.committeeId());
        ctx.put("candidateGroups", assignment.candidateGroups());
        return ctx;
    }

    @Transactional
    void persistDeviationCaseId(UUID deviationId, UUID caseId) {
        ProtocolDeviation deviation = ProtocolDeviation.findById(deviationId);
        if (deviation == null) {
            LOG.warnf("IrbDeviationCaseService: ProtocolDeviation not found for deviationId=%s", deviationId);
            return;
        }
        deviation.engineCaseId = caseId;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=IrbGateLifecycleTest,IrbDeviationCaseServiceTest --batch-mode
```
Expected: All tests pass including new engineCaseId and committeeId assertions.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/IrbDeviationCaseService.java \
  runtime/src/test/java/io/casehub/clinical/service/IrbGateLifecycleTest.java \
  runtime/src/test/java/io/casehub/clinical/service/IrbDeviationCaseServiceTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(#26,#29): IrbDeviationCaseService three-phase refactor + IrbCommitteeAssignmentPolicy SPI

Phase 1: evaluate SPI, create IrbApproval with assignment.committeeId()
Phase 2: startCase().join() outside TX
Phase 3: deviation.engineCaseId = caseId

DefaultIrbCommitteeAssignmentPolicy wired via @DefaultBean.
IrbGateLifecycleTest: assert deviation.engineCaseId non-null, approval.committeeId = 'irb-committee'.

Closes #26, Closes #29
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 13: Full test suite + code review + doc sync

- [ ] **Step 1: Run full test suite**

```bash
mvn test --batch-mode
```
Expected: `Tests run: ≥136, Failures: 0, Errors: 0` (likely more — new tests added)

- [ ] **Step 2: Invoke code review**

Invoke `superpowers:requesting-code-review` — review all staged changes against the spec and platform patterns before any final commit.

- [ ] **Step 3: Invoke implementation-doc-sync**

Invoke `implementation-doc-sync` — sync CLAUDE.md, DESIGN.md, and any affected docs with what was actually built.

---

## Self-Review

**Spec coverage check:**
- ✅ #28: AeEscalationListener + test import fix, arity fix (Task 1)
- ✅ #39: LAYER-LOG build note (Task 2)
- ✅ #27: AeEscalationStatus enum (Task 3), AdverseEvent fields (Task 4), AeEscalationCaseService Phase 1 REQUESTED + Phase 4 FAILED (Task 9), AeEscalationListener COMPLETED (Task 10)
- ✅ #26: AdverseEvent.engineCaseId (Task 4), ProtocolDeviation.engineCaseId (Task 5), Phase 3 in AeEscalationCaseService (Task 9), Phase 3 in IrbDeviationCaseService (Task 12)
- ✅ #29: SPI types (Task 6), DefaultIrbCommitteeAssignmentPolicy (Task 7), IrbDeviationCaseService wiring (Task 12)
- ✅ Integration test @BeforeEach entity creation (Tasks 8 and 11)
- ✅ IrbGateLifecycleTest stale comment fixed (Task 11)
- ✅ AeEscalationCaseServiceTest Mockito incompatibility resolved (Task 9)
- ✅ IrbDeviationCaseServiceTest @Mock committeePolicy added (Task 11)
- ✅ Signaling coverage gap issue filed (Task 9, Step 2)
