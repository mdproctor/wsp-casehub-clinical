# Escalation Re-evaluation on Grade Upgrade — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #147 — feat: re-evaluate escalation on upgrade when engineCaseId already exists
**Issue group:** #147

**Goal:** When an AE is regraded upward and an engine case already exists, start a fresh escalation case at the new grade instead of silently returning null.

**Architecture:** Remove the `engineCaseId != null` guard in `prepareAndMarkForRegrade()`. Add case-ID discrimination to completion handling so superseded cases write ledger entries but don't corrupt authoritative state. Add a grade threshold gate to `AeGradeChangeEscalationListener` to prevent low-grade upgrades from entering the engine case path.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mockito, AssertJ, Awaitility

## Global Constraints

- No Flyway migrations — all changes are code-only
- `AeStatusUpdater.markCompleted()` callers: `AeEscalationListener` (production), `AeEscalationListenerTest` (7 mock sites), `AeEscalationListenerMemoryTest` (2 mock sites)
- `CompletionResult` enum is a nested type inside `AeStatusUpdater` — single-file scope
- Existing tests use `when(statusUpdater.markCompleted(aeId)).thenReturn(true/false)` — all must be updated to the new signature
- Test pattern: `AeEscalationListenerTest` uses `@Mock/@InjectMocks` (unit); `AeEscalationListenerMemoryTest` uses `@QuarkusTest/@InjectMock`

---

## Batch 1: Case-ID guarded completion

### Task 1: CompletionResult enum + AeStatusUpdater signature change

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeStatusUpdater.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AeStatusUpdaterTest.java`

**Interfaces:**
- Produces: `AeStatusUpdater.CompletionResult` enum (COMPLETED, ALREADY_COMPLETED, SUPERSEDED, NOT_FOUND); `markCompleted(UUID aeId, UUID expectedCaseId)` returning `CompletionResult`

- [ ] **Step 1: Write the failing tests**

Create `AeStatusUpdaterTest.java`:

```java
package io.casehub.clinical.service;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class AeStatusUpdaterTest {

    @Inject AeStatusUpdater statusUpdater;
    @Inject FixedCurrentPrincipal principal;

    private UUID aeId;
    private UUID caseId;

    @BeforeEach
    @Transactional
    void setup() {
        aeId = UUID.randomUUID();
        caseId = UUID.randomUUID();

        AdverseEvent ae = new AdverseEvent();
        ae.id = aeId;
        ae.enrollmentId = UUID.randomUUID();
        ae.tenantId = principal.tenancyId();
        ae.grade = CtcaeGrade.GRADE_3;
        ae.actuality = EventActuality.ACTUAL;
        ae.outcome = AeOutcome.ONGOING;
        ae.occurredAt = Instant.now();
        ae.reportedAt = Instant.now();
        ae.escalationStatus = AeEscalationStatus.REQUESTED;
        ae.engineCaseId = caseId;
        ae.persist();
    }

    @Test
    void matching_caseId_returns_COMPLETED() {
        var result = statusUpdater.markCompleted(aeId, caseId);
        assertThat(result).isEqualTo(AeStatusUpdater.CompletionResult.COMPLETED);
    }

    @Test
    void mismatched_caseId_returns_SUPERSEDED() {
        var result = statusUpdater.markCompleted(aeId, UUID.randomUUID());
        assertThat(result).isEqualTo(AeStatusUpdater.CompletionResult.SUPERSEDED);
    }

    @Test
    void null_engineCaseId_with_non_null_expected_returns_SUPERSEDED() {
        setEngineCaseId(null);
        var result = statusUpdater.markCompleted(aeId, UUID.randomUUID());
        assertThat(result).isEqualTo(AeStatusUpdater.CompletionResult.SUPERSEDED);
    }

    @Test
    void already_completed_returns_ALREADY_COMPLETED() {
        setEscalationStatus(AeEscalationStatus.COMPLETED);
        var result = statusUpdater.markCompleted(aeId, caseId);
        assertThat(result).isEqualTo(AeStatusUpdater.CompletionResult.ALREADY_COMPLETED);
    }

    @Test
    void nonexistent_ae_returns_NOT_FOUND() {
        var result = statusUpdater.markCompleted(UUID.randomUUID(), caseId);
        assertThat(result).isEqualTo(AeStatusUpdater.CompletionResult.NOT_FOUND);
    }

    @Transactional
    void setEscalationStatus(AeEscalationStatus status) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        ae.escalationStatus = status;
    }

    @Transactional
    void setEngineCaseId(UUID id) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        ae.engineCaseId = id;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=AeStatusUpdaterTest --batch-mode`
Expected: Compilation failure — `CompletionResult` does not exist, `markCompleted` has wrong signature. 5 tests expected.

- [ ] **Step 3: Implement CompletionResult enum and update markCompleted**

Replace the full body of `AeStatusUpdater.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.model.AeEscalationStatus;
import io.casehub.clinical.entity.AdverseEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class AeStatusUpdater {

    private static final Logger LOG = Logger.getLogger(AeStatusUpdater.class);

    public enum CompletionResult {
        COMPLETED,
        ALREADY_COMPLETED,
        SUPERSEDED,
        NOT_FOUND
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public CompletionResult markCompleted(UUID aeId, UUID expectedCaseId) {
        AdverseEvent ae = AdverseEvent.findById(aeId);
        if (ae == null) {
            LOG.warnf("AeStatusUpdater: AdverseEvent not found for aeId=%s — status not updated", aeId);
            return CompletionResult.NOT_FOUND;
        }
        if (ae.escalationStatus == AeEscalationStatus.COMPLETED) {
            LOG.debugf("AeStatusUpdater: aeId=%s already COMPLETED — skipping", aeId);
            return CompletionResult.ALREADY_COMPLETED;
        }
        if (!expectedCaseId.equals(ae.engineCaseId)) {
            LOG.infof("Superseded escalation case %s completed for aeId=%s — current case is %s",
                expectedCaseId, aeId, ae.engineCaseId);
            return CompletionResult.SUPERSEDED;
        }
        ae.escalationStatus = AeEscalationStatus.COMPLETED;
        LOG.infof("AeStatusUpdater: escalationStatus set to COMPLETED for aeId=%s", aeId);
        return CompletionResult.COMPLETED;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=AeStatusUpdaterTest --batch-mode`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/AeStatusUpdater.java
git add runtime/src/test/java/io/casehub/clinical/service/AeStatusUpdaterTest.java
git commit -m "feat(#147): CompletionResult enum + case-ID guarded markCompleted Refs #147"
```

---

### Task 2: Update AeEscalationListener for differentiated completion handling

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java` (add `writeSupersededCompletionEntry`)
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerMemoryTest.java`

**Interfaces:**
- Consumes: `AeStatusUpdater.CompletionResult` enum, `markCompleted(UUID, UUID)` from Task 1

- [ ] **Step 1: Write the failing test for superseded completion**

Add to `AeEscalationListenerTest.java`:

```java
@Test
void superseded_case_writes_ledger_but_skips_memory_and_event() {
    UUID caseId = UUID.randomUUID();
    UUID aeId = UUID.randomUUID();
    UUID enrollmentId = UUID.randomUUID();
    UUID siteId = UUID.randomUUID();

    ObjectNode snapshot = buildSnapshot(aeId, enrollmentId, siteId, "GRADE_3",
            "REVIEWED", null, "test-tenant", null);

    when(statusUpdater.markCompleted(aeId, caseId))
        .thenReturn(AeStatusUpdater.CompletionResult.SUPERSEDED);

    listener.onCaseLifecycle(goalReachedEvent(caseId, snapshot));

    verify(ledgerWriter).writeSupersededCompletionEntry(eq(aeId), eq(enrollmentId),
        eq(CtcaeGrade.GRADE_3), eq("REVIEWED"), eq(false), any());
    verifyNoInteractions(memoryService);
    verifyNoInteractions(completedEvents);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=AeEscalationListenerTest#superseded_case_writes_ledger_but_skips_memory_and_event --batch-mode`
Expected: Compilation failure — `markCompleted` signature mismatch, `writeSupersededCompletionEntry` does not exist

- [ ] **Step 3: Add writeSupersededCompletionEntry to AeEscalationLedgerWriter**

Read `AeEscalationLedgerWriter.java` and add a `writeSupersededCompletionEntry` method that mirrors `writeCompletionEntry` but uses `actorRole = "AeEscalationCase-superseded"` instead of the default. The method signature is identical to `writeCompletionEntry`:

```java
public void writeSupersededCompletionEntry(UUID aeId, UUID enrollmentId,
        CtcaeGrade grade, String safetyReviewOutcome,
        boolean dsmbEscalated, Instant completedAt) {
    // Same as writeCompletionEntry but with actorRole = "AeEscalationCase-superseded"
}
```

The implementation follows the existing `writeCompletionEntry` pattern — read it first, then duplicate with the different `actorRole`. No schema change needed — `actorRole` is already a String field on `JpaLedgerEntry`.

- [ ] **Step 5: Update AeEscalationListener.onCaseLifecycle()**

Replace the body of `onCaseLifecycle()` in `AeEscalationListener.java`. The key changes:
- `markCompleted(aeId)` → `markCompleted(aeId, event.caseId())`
- `boolean firstCompletion` → `AeStatusUpdater.CompletionResult result`
- After ledger write: `if (result == CompletionResult.SUPERSEDED) return;`

New method body:

```java
public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
    LOG.debugf("AeEscalationListener: received eventType=%s caseStatus=%s caseId=%s", event.eventType(), event.caseStatus(), event.caseId());
    if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;

    JsonNode snapshot = event.contextSnapshot();
    if (snapshot == null) return;

    String aeIdStr = snapshot.path("aeId").asText(null);
    if (aeIdStr == null) return;

    UUID aeId;
    try {
        aeId = UUID.fromString(aeIdStr);
    } catch (IllegalArgumentException e) {
        LOG.warnf("AeEscalationListener: invalid aeId in case context: %s", aeIdStr);
        return;
    }

    AeStatusUpdater.CompletionResult result = statusUpdater.markCompleted(aeId, event.caseId());
    if (result == AeStatusUpdater.CompletionResult.NOT_FOUND
            || result == AeStatusUpdater.CompletionResult.ALREADY_COMPLETED) return;

    UUID enrollmentId = resolveUuid(snapshot.path("enrollmentId").asText(null));
    if (enrollmentId == null) {
        LOG.warnf("AeEscalationListener: enrollmentId missing from case context for aeId=%s — ledger write skipped", aeId);
        return;
    }
    UUID siteId = resolveUuid(snapshot.path("siteId").asText(null));
    CtcaeGrade grade = resolveGrade(snapshot.path("grade").asText(null));
    String safetyReviewOutcome = snapshot.path("safetyReview").path(OUTCOME_KEY).asText(null);
    boolean dsmbEscalated = !snapshot.path("dsmbEscalation").isMissingNode()
            && !snapshot.path("dsmbEscalation").isNull();
    boolean unexpected = snapshot.path("unexpected").asBoolean(false);
    Instant completedAt = Instant.now();

    boolean ledgerWritten = false;
    try {
        if (result == AeStatusUpdater.CompletionResult.SUPERSEDED) {
            ledgerWriter.writeSupersededCompletionEntry(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, completedAt);
            return;
        }
        ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, completedAt);
        ledgerWritten = true;

        try { aeTrajectoryAlertService.evaluate(aeId, event.tenancyId()); } catch (Exception te) { LOG.warnf(te, "Trajectory alert evaluation failed for aeId=%s", aeId); }
        String tenantId = snapshot.path("tenantId").asText(null);
        if (tenantId != null) {
            memoryService.storeAeOutcome(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, tenantId);
        }
        completedEvents.fireAsync(new AeEscalationCompletedEvent(
                aeId, grade, siteId, safetyReviewOutcome, dsmbEscalated, completedAt, unexpected));
    } catch (Exception e) {
        if (!ledgerWritten) {
            LOG.errorf(e, "AeEscalationListener: unexpected error for aeId=%s (enrollmentId=%s, grade=%s) — writing failure entry", aeId, enrollmentId, grade);
            try {
                ledgerWriter.writeObserverFailureEntry(aeId, enrollmentId, grade);
            } catch (Exception writeEx) {
                LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for aeId=%s", aeId);
            }
        } else {
            LOG.errorf(e, "AeEscalationListener: downstream fireAsync failed for aeId=%s — ledger entry exists, no fallback needed", aeId);
        }
    }
}
```

- [ ] **Step 6: Update existing tests in AeEscalationListenerTest**

All `when(statusUpdater.markCompleted(aeId)).thenReturn(true/false)` calls must change to `when(statusUpdater.markCompleted(eq(aeId), any())).thenReturn(CompletionResult.COMPLETED/ALREADY_COMPLETED)`.

Tests returning `true` → `CompletionResult.COMPLETED` (7 sites):
- `completed_event_carries_siteId_from_case_context` (line 48)
- `completed_event_carries_unexpected_from_case_context` (line 75)
- `markCompleted_true_but_enrollmentId_null_skips_ledger_write` (line 107)
- `writeCompletionEntry_throws_writes_observer_failure_entry` (line 140)
- `writeCompletionEntry_and_fallback_both_throw_does_not_propagate` (line 158)
- `fireAsync_throws_after_ledger_written_no_failure_entry` (line 176)

Test returning `false` → `CompletionResult.ALREADY_COMPLETED` (1 site):
- `idempotency_guard_skips_ledger_write_on_duplicate_goal_reached` (line 124)

Add import: `import static org.mockito.ArgumentMatchers.any;` (if not present) and `import io.casehub.clinical.service.AeStatusUpdater.CompletionResult;`

Each mock site changes from:
```java
when(statusUpdater.markCompleted(aeId)).thenReturn(true);
```
to:
```java
when(statusUpdater.markCompleted(eq(aeId), any())).thenReturn(CompletionResult.COMPLETED);
```

And the false case:
```java
when(statusUpdater.markCompleted(aeId)).thenReturn(false);
```
to:
```java
when(statusUpdater.markCompleted(eq(aeId), any())).thenReturn(CompletionResult.ALREADY_COMPLETED);
```

- [ ] **Step 7: Update existing tests in AeEscalationListenerMemoryTest**

Same pattern — 2 mock sites:
- `storeAeOutcome_called_with_correct_args_on_completion` (line 57)
- `storeAeOutcome_skipped_when_tenantId_absent` (line 83)

Both change from:
```java
when(statusUpdater.markCompleted(aeId)).thenReturn(true);
```
to:
```java
when(statusUpdater.markCompleted(eq(aeId), any())).thenReturn(AeStatusUpdater.CompletionResult.COMPLETED);
```

Add import: `import static org.mockito.ArgumentMatchers.eq;`

- [ ] **Step 8: Run all affected tests**

Run: `mvn test -pl runtime -Dtest="AeEscalationListenerTest,AeEscalationListenerMemoryTest,AeStatusUpdaterTest" --batch-mode`
Expected: All tests PASS (including the new superseded test)

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java
git add runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java
git add runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java
git add runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerMemoryTest.java
git commit -m "feat(#147): differentiated completion handling — superseded cases write ledger only Refs #147"
```

---

## Batch 2: Regrade escalation path

### Task 3: Grade threshold gate in AeGradeChangeEscalationListener

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeEscalationListener.java`
- Create: `runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeEscalationListenerTest.java`

**Interfaces:**
- Consumes: `AeEscalationCaseService.startEscalationForRegrade(UUID, UUID, UUID, CtcaeGrade, String)` (unchanged)

- [ ] **Step 1: Write the failing tests**

Create `AeGradeChangeEscalationListenerTest.java`:

```java
package io.casehub.clinical.service;

import static org.mockito.Mockito.*;

import io.casehub.clinical.api.AeGradeChangedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class AeGradeChangeEscalationListenerTest {

    @Mock AeEscalationCaseService escalationService;
    @InjectMocks AeGradeChangeEscalationListener listener;

    @Test
    void grade1_to_grade2_upgrade_does_not_call_escalation() {
        listener.onGradeChanged(gradeChanged(CtcaeGrade.GRADE_1, CtcaeGrade.GRADE_2));
        verifyNoInteractions(escalationService);
    }

    @Test
    void grade2_to_grade3_upgrade_calls_escalation() {
        var event = gradeChanged(CtcaeGrade.GRADE_2, CtcaeGrade.GRADE_3);
        listener.onGradeChanged(event);
        verify(escalationService).startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }

    @Test
    void grade3_to_grade4_upgrade_calls_escalation() {
        var event = gradeChanged(CtcaeGrade.GRADE_3, CtcaeGrade.GRADE_4);
        listener.onGradeChanged(event);
        verify(escalationService).startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }

    @Test
    void grade4_to_grade5_upgrade_calls_escalation() {
        var event = gradeChanged(CtcaeGrade.GRADE_4, CtcaeGrade.GRADE_5);
        listener.onGradeChanged(event);
        verify(escalationService).startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }

    @Test
    void downgrade_does_not_call_escalation() {
        listener.onGradeChanged(gradeChanged(CtcaeGrade.GRADE_4, CtcaeGrade.GRADE_2));
        verifyNoInteractions(escalationService);
    }

    private AeGradeChangedEvent gradeChanged(CtcaeGrade from, CtcaeGrade to) {
        return new AeGradeChangedEvent(
            UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
            from, to, Instant.now(), "test-user", "test-tenant");
    }
}
```

- [ ] **Step 2: Run tests to verify the Grade 1→2 test fails**

Run: `mvn test -pl runtime -Dtest=AeGradeChangeEscalationListenerTest --batch-mode`
Expected: `grade1_to_grade2_upgrade_does_not_call_escalation` FAILS (listener currently calls escalation for any upgrade)

- [ ] **Step 3: Add grade threshold gate**

Replace `AeGradeChangeEscalationListener.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.AeGradeChangedEvent;
import io.casehub.clinical.api.model.CtcaeGrade;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

@ApplicationScoped
public class AeGradeChangeEscalationListener {

    @Inject AeEscalationCaseService escalationService;

    public void onGradeChanged(@ObservesAsync AeGradeChangedEvent event) {
        if (!event.isUpgrade()) return;
        if (event.newGrade().ordinal() < CtcaeGrade.GRADE_3.ordinal()) return;
        escalationService.startEscalationForRegrade(
            event.aeId(), event.enrollmentId(), event.siteId(), event.newGrade(), event.tenantId());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=AeGradeChangeEscalationListenerTest --batch-mode`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/AeGradeChangeEscalationListener.java
git add runtime/src/test/java/io/casehub/clinical/service/AeGradeChangeEscalationListenerTest.java
git commit -m "feat(#147): add Grade 3+ threshold gate to AeGradeChangeEscalationListener Refs #147"
```

---

### Task 4: Guard removal + engineCaseId nullification + integration test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java`

**Interfaces:**
- Consumes: `AeStatusUpdater.CompletionResult` (from Task 1), differentiated listener (from Task 2), threshold gate (from Task 3)

- [ ] **Step 1: Write the failing integration test**

Add to `AeEscalationLifecycleTest.java`:

```java
@Test
void grade3_to_grade4_regrade_starts_fresh_case_with_dsmb() throws Exception {
    // Phase 1: Report Grade 3 — starts escalation case (senior monitor only)
    aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_3));

    await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
            .untilAsserted(() -> {
                List<WorkItem> items = aeWorkItems();
                assertThat(items.stream().anyMatch(wi -> wi.title().contains("Senior safety monitor")))
                        .as("safety-review WorkItem for Grade 3").isTrue();
            });

    // Complete Grade 3 safety review
    WorkItem safetyWorkItem = aeWorkItems().stream()
            .filter(wi -> wi.title().contains("Senior safety monitor"))
            .findFirst().orElseThrow();
    String resolution = "{\"outcome\":\"REVIEWED\",\"reviewedAt\":\"2026-09-06T13:00:00Z\"}";
    workItemService.completeFromSystem(safetyWorkItem.id(), "senior-monitor", resolution);

    WorkItem completed = aeWorkItems().stream()
            .filter(wi -> wi.title().contains("Senior safety monitor"))
            .findFirst().orElseThrow();
    lifecycleAdapter.onWorkItemLifecycle(
            WorkItemLifecycleEvent.of("COMPLETED", completed, "senior-monitor", completed.resolution()));

    await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS)
            .untilAsserted(() ->
                    assertThat(findAe(aeId).escalationStatus).isEqualTo(AeEscalationStatus.COMPLETED));

    UUID originalCaseId = findAe(aeId).engineCaseId;
    assertThat(originalCaseId).isNotNull();

    // Phase 2: Regrade to Grade 4 — should start a fresh case
    aeEscalationCaseService.startEscalationForRegrade(
            aeId, enrollmentId, siteId, CtcaeGrade.GRADE_4, principal.tenancyId());

    // Fresh case should be started — new engineCaseId, escalationStatus back to REQUESTED or COMPLETED
    AdverseEvent regraded = findAe(aeId);
    assertThat(regraded.engineCaseId).isNotNull();
    assertThat(regraded.engineCaseId).isNotEqualTo(originalCaseId);

    // Grade 4 should have DSMB WorkItem in addition to safety-review
    await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
            .untilAsserted(() -> {
                List<WorkItem> items = aeWorkItems();
                assertThat(items.stream().anyMatch(wi -> wi.title().contains("DSMB")))
                        .as("DSMB WorkItem for Grade 4 regrade").isTrue();
            });

    verify(trialSafetySignalService).signalGrade4Active(siteId);
}

@Test
void grade3_to_grade4_regrade_with_active_case_supersedes_old_case() throws Exception {
    // Phase 1: Report Grade 3 — starts escalation case (senior monitor only)
    aeEscalationCaseService.onAdverseEventReported(aeEvent(CtcaeGrade.GRADE_3));

    await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
            .untilAsserted(() -> {
                List<WorkItem> items = aeWorkItems();
                assertThat(items.stream().anyMatch(wi -> wi.title().contains("Senior safety monitor")))
                        .as("safety-review WorkItem for Grade 3").isTrue();
            });

    UUID originalCaseId = findAe(aeId).engineCaseId;
    assertThat(originalCaseId).isNotNull();
    assertThat(findAe(aeId).escalationStatus)
            .isIn(AeEscalationStatus.REQUESTED, AeEscalationStatus.COMPLETED);

    // Phase 2: Regrade to Grade 4 WITHOUT completing Grade 3 — active case superseded
    aeEscalationCaseService.startEscalationForRegrade(
            aeId, enrollmentId, siteId, CtcaeGrade.GRADE_4, principal.tenancyId());

    // Fresh case should be started — new engineCaseId
    AdverseEvent regraded = findAe(aeId);
    assertThat(regraded.engineCaseId).isNotNull();
    assertThat(regraded.engineCaseId).isNotEqualTo(originalCaseId);

    // Grade 4 case creates DSMB WorkItem
    await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS)
            .untilAsserted(() -> {
                List<WorkItem> items = aeWorkItems();
                assertThat(items.stream().anyMatch(wi -> wi.title().contains("DSMB")))
                        .as("DSMB WorkItem for Grade 4 regrade").isTrue();
            });

    // Phase 3: Complete old Grade 3 safety review — should be SUPERSEDED
    WorkItem oldSafetyWorkItem = aeWorkItems().stream()
            .filter(wi -> wi.title().contains("Senior safety monitor"))
            .findFirst().orElseThrow();
    String resolution = "{\"outcome\":\"REVIEWED\",\"reviewedAt\":\"2026-09-06T13:00:00Z\"}";
    workItemService.completeFromSystem(oldSafetyWorkItem.id(), "senior-monitor", resolution);

    WorkItem completedOld = aeWorkItems().stream()
            .filter(wi -> wi.id().equals(oldSafetyWorkItem.id()))
            .findFirst().orElseThrow();
    lifecycleAdapter.onWorkItemLifecycle(
            WorkItemLifecycleEvent.of("COMPLETED", completedOld, "senior-monitor", completedOld.resolution()));

    // Old case completing should NOT set escalationStatus to COMPLETED
    // (engineCaseId points to new case — case-ID mismatch → SUPERSEDED)
    assertThat(findAe(aeId).escalationStatus).isNotEqualTo(AeEscalationStatus.COMPLETED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=AeEscalationLifecycleTest#grade3_to_grade4_regrade_starts_fresh_case_with_dsmb --batch-mode`
Expected: FAIL — `engineCaseId` is unchanged (guard returns null, no new case started)

- [ ] **Step 3: Remove guard and null engineCaseId in prepareAndMarkForRegrade**

In `AeEscalationCaseService.java`, replace lines 85-129 (`prepareAndMarkForRegrade` method):

```java
@Transactional
Map<String, Object> prepareAndMarkForRegrade(UUID aeId, UUID enrollmentId, UUID siteId,
                                             io.casehub.clinical.api.model.CtcaeGrade grade, String tenantId) {
    AdverseEvent ae = AdverseEvent.findById(aeId);
    if (ae == null) {
        LOG.warnf("AE not found for regrade escalation aeId=%s", aeId);
        return null;
    }
    ae.engineCaseId = null;
    if (ae.workItemId != null) {
        LOG.infof("Cancelling Grade 1/2 WorkItem %s for aeId=%s — engine case taking over", ae.workItemId, aeId);
        ae.workItemId = null;
    }
    ae.escalationStatus = AeEscalationStatus.REQUESTED;

    AdverseEventEscalationRequirements requirements = policy.evaluate(
            new AdverseEventContext(aeId, enrollmentId, siteId, grade));

    Map<String, Object> ctx = new HashMap<>();
    ctx.put("aeId", aeId.toString());
    ctx.put("enrollmentId", enrollmentId.toString());
    ctx.put("siteId", siteId != null ? siteId.toString() : "");
    ctx.put("grade", grade.name());
    ctx.put("requiresSeniorMonitor", requirements.requiresSeniorMonitor());
    ctx.put("requiresDsmbEscalation", requirements.requiresDsmbEscalation());
    ctx.put("tenantId", tenantId);
    var patientCtx = memoryService.queryPatientContext(enrollmentId, tenantId);
    ctx.put("patientContext", patientCtx.toContextMap());
    if (siteId != null) {
        var siteCtx = memoryService.querySiteContext(siteId, tenantId);
        ctx.put("siteContext", siteCtx.toContextMap());
    }
    ctx.put("unexpected", ae.unexpected);
    ctx.put("suspected", ae.suspected);

    var plan = planRetriever.retrieve(ae);
    if (plan.hasRecommendation()) {
        ctx.put("escalationPlanRecommendation", plan.toContextMap());
    }
    return ctx;
}
```

The only change vs the original: removed the `if (ae.engineCaseId != null)` guard (lines 92-97) and added `ae.engineCaseId = null` before setting REQUESTED.

- [ ] **Step 4: Run the integration test**

Run: `mvn test -pl runtime -Dtest=AeEscalationLifecycleTest#grade3_to_grade4_regrade_starts_fresh_case_with_dsmb --batch-mode`
Expected: PASS — new case started with DSMB, new engineCaseId persisted

- [ ] **Step 5: Run the full test suite**

Run: `mvn test -pl runtime --batch-mode`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java
git add runtime/src/test/java/io/casehub/clinical/service/AeEscalationLifecycleTest.java
git commit -m "feat(#147): remove engineCaseId guard — regrade always starts fresh escalation case Refs #147"
```

---

## References

- [2026-09-06-escalation-regrade-design.md] — design spec this plan implements
- [AeEscalationCaseService.java:85-129] — prepareAndMarkForRegrade method
- [AeStatusUpdater.java:28-42] — markCompleted gaining CompletionResult
- [AeEscalationListener.java:35-92] — lifecycle listener
- [AeGradeChangeEscalationListener.java:9-18] — listener gaining threshold gate
- [AeEscalationLifecycleTest.java:35-177] — existing integration test pattern
- [AeEscalationListenerTest.java:28-211] — existing unit test pattern
- [GitHub #147] — focal issue
