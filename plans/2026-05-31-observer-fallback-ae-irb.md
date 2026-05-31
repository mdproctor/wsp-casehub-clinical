# Observer Fallback — AeEscalationListener + IrbDecisionListener Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add double try/catch + REQUIRES_NEW fallback to `AeEscalationListener` and `IrbDecisionListener` so that any unhandled exception after a REQUIRES_NEW commit still produces a ledger entry, satisfying GCP/FDA audit requirements.

**Architecture:** Context resolution moves outside the try block in `AeEscalationListener` (matching the IRB pattern), making the try block narrow (ledger write + fireAsync only). A `ledgerWritten`/`ledgerDecisionWritten` boolean flag gates the fallback: if the ledger write succeeded and only `fireAsync` threw, no fallback entry is written (prevents double-recording). `IrbDecisionListener` gets the same flag around `writeDecisionEntry`. Both writers add `writeObserverFailureEntry(@Transactional REQUIRES_NEW)` methods. `AeEscalationLedgerEntry.ctcaeGrade` is made nullable to support failure entries where grade was not resolved.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mockito (unit), QuarkusTest + H2 (integration), Panache Active Record, Narayana JTA.

---

## File map

| File | Action | Purpose |
|------|--------|---------|
| `runtime/src/main/java/io/casehub/clinical/ledger/AeEscalationLedgerEntry.java` | Modify | Make `ctcaeGrade` nullable |
| `runtime/src/main/resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql` | Create | Production schema: DROP NOT NULL on ctcae_grade |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java` | Modify | Fix null grade in `writeCompletionEntry`; add `writeObserverFailureEntry` |
| `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java` | Modify | 3 new tests for null-grade fix and failure writer |
| `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java` | Modify | Context outside try; narrow try with `ledgerWritten` flag |
| `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java` | Modify | 3 new tests for fallback and fireAsync edge case |
| `runtime/src/main/java/io/casehub/clinical/service/IrbApprovalLedgerWriter.java` | Modify | Add `writeObserverFailureEntry`; add ClinicalActors import |
| `runtime/src/test/java/io/casehub/clinical/service/IrbApprovalLedgerWriterTest.java` | Modify | 1 new test for failure writer |
| `runtime/src/main/java/io/casehub/clinical/service/IrbDecisionListener.java` | Modify | Add `ledgerDecisionWritten` flag; try/catch around post-load body |
| `runtime/src/test/java/io/casehub/clinical/service/IrbDecisionListenerTest.java` | Create | 6 QuarkusTest integration tests |

All paths are under `/Users/mdproctor/claude/casehub/clinical/`.

---

## Task 1: Schema change — AeEscalationLedgerEntry nullable grade

**Files:**
- Modify: `runtime/src/main/java/io/casehub/clinical/ledger/AeEscalationLedgerEntry.java:19`
- Create: `runtime/src/main/resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql`

- [ ] **Step 1: Change nullable=false to nullable on ctcaeGrade**

In `AeEscalationLedgerEntry.java`, change line 19:
```java
// Before
@Column(name = "ctcae_grade", nullable = false)
public String ctcaeGrade;
```
```java
// After
@Column(name = "ctcae_grade")
public String ctcaeGrade;
```

- [ ] **Step 2: Create production migration**

Create `runtime/src/main/resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql`:
```sql
-- V1012: allow null ctcae_grade in ae_escalation_ledger_entry for observer failure entries.
-- When AeEscalationListener throws before resolving grade from case context,
-- null is semantically correct (grade indeterminate at failure time).
ALTER TABLE ae_escalation_ledger_entry ALTER COLUMN ctcae_grade DROP NOT NULL;
```

- [ ] **Step 3: Compile to verify**

```bash
mvn compile -pl api,runtime --batch-mode
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/ledger/AeEscalationLedgerEntry.java \
  runtime/src/main/resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-48): make ae_escalation_ledger_entry.ctcae_grade nullable for observer failure entries

Refs #48"
```

---

## Task 2: AeEscalationLedgerWriter — fix null grade + add writeObserverFailureEntry (TDD)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java`

- [ ] **Step 1: Write three failing tests**

Add to `AeEscalationLedgerWriterTest.java` (append after existing test):

```java
    @Test
    void writeCompletionEntry_with_null_grade_stores_null() {
        Instant now = Instant.parse("2026-05-31T10:00:00Z");
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        writer.writeCompletionEntry(aeId, enrollmentId, null, null, false, now);

        ArgumentCaptor<AeEscalationLedgerEntry> captor =
                ArgumentCaptor.forClass(AeEscalationLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());
        assertThat(captor.getValue().ctcaeGrade).isNull();
    }

    @Test
    void writeObserverFailureEntry_with_null_grade_saves_null_grade() {
        Instant now = Instant.parse("2026-05-31T10:00:00Z");
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        writer.writeObserverFailureEntry(aeId, enrollmentId, null);

        ArgumentCaptor<AeEscalationLedgerEntry> captor =
                ArgumentCaptor.forClass(AeEscalationLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());
        AeEscalationLedgerEntry entry = captor.getValue();
        assertThat(entry.ctcaeGrade).isNull();
        assertThat(entry.actorRole).isEqualTo("AeEscalationCase-observer-failed");
        assertThat(entry.aeId).isEqualTo(aeId);
        assertThat(entry.enrollmentId).isEqualTo(enrollmentId);
        assertThat(entry.dsmbEscalated).isFalse();
        assertThat(entry.actorId).isEqualTo("clinical-service");
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void writeObserverFailureEntry_with_valid_grade_saves_grade() {
        Instant now = Instant.parse("2026-05-31T10:00:00Z");
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        writer.writeObserverFailureEntry(aeId, enrollmentId, CtcaeGrade.GRADE_3);

        ArgumentCaptor<AeEscalationLedgerEntry> captor =
                ArgumentCaptor.forClass(AeEscalationLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());
        assertThat(captor.getValue().ctcaeGrade).isEqualTo("GRADE_3");
    }
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode
```
Expected: `writeCompletionEntry_with_null_grade_stores_null` fails with NullPointerException (grade.name()); other two fail with NoSuchMethodError (method not yet defined).

- [ ] **Step 3: Fix writeCompletionEntry and add writeObserverFailureEntry**

In `AeEscalationLedgerWriter.java`:

Replace line 40 (`entry.ctcaeGrade = grade.name();`):
```java
        entry.ctcaeGrade = grade != null ? grade.name() : null;
```

Add new method after `writeCompletionEntry` (before `nextSequenceNumber`):
```java
    /**
     * Called from AeEscalationListener observer fallback path.
     * Commits in its own REQUIRES_NEW transaction so it persists even if the
     * outer transaction is in rollback-only state.
     * grade may be null if the exception occurred before grade was resolved from case context.
     */
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeObserverFailureEntry(UUID aeId, UUID enrollmentId, CtcaeGrade grade) {
        var entry = new AeEscalationLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = aeId;
        entry.sequenceNumber = nextSequenceNumber(aeId);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "AeEscalationCase-observer-failed";
        entry.occurredAt = clock.instant();
        entry.aeId = aeId;
        entry.enrollmentId = enrollmentId;
        entry.ctcaeGrade = grade != null ? grade.name() : null;
        entry.dsmbEscalated = false;
        entry.completedAt = clock.instant();
        ledgerEntryRepository.save(entry);
    }
```

Also add `import jakarta.transaction.Transactional;` if not already present (check current imports — it may already be there from other Transactional usage in the file; if absent, add it).

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=AeEscalationLedgerWriterTest --batch-mode
```
Expected: BUILD SUCCESS, 4 tests passed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-48): add AeEscalationLedgerWriter.writeObserverFailureEntry + fix null grade

Refs #48"
```

---

## Task 3: AeEscalationListener — narrow try block + ledgerWritten flag (TDD)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java`

- [ ] **Step 1: Write three failing tests**

Add to `AeEscalationListenerTest.java` (append after existing tests). Add these imports at the top if not already present:
```java
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.anyBoolean;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.never;
```

Add helper method:
```java
    private CaseLifecycleEvent goalReachedEvent(UUID caseId) {
        return new CaseLifecycleEvent(
                caseId, "CompleteCase", "GoalReached", "RUNNING", "system", "system", null);
    }

    private CaseInstance mockInstanceWith(UUID caseId, UUID aeId, UUID enrollmentId, String grade) {
        CaseContext ctx = mock(CaseContext.class);
        when(ctx.getPath("aeId")).thenReturn(aeId.toString());
        when(ctx.getPath("enrollmentId")).thenReturn(enrollmentId.toString());
        when(ctx.getPath("grade")).thenReturn(grade);
        when(ctx.getPath("siteId")).thenReturn(null);
        when(ctx.getPath("safetyReview")).thenReturn(null);
        when(ctx.getPath("dsmbEscalation")).thenReturn(null);
        CaseInstance instance = mock(CaseInstance.class);
        when(instance.getCaseContext()).thenReturn(ctx);
        when(caseInstanceRepository.findByUuid(caseId)).thenReturn(Uni.createFrom().item(instance));
        return instance;
    }
```

Add the three new tests:
```java
    @Test
    void writeCompletionEntry_throws_writes_observer_failure_entry() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        mockInstanceWith(caseId, aeId, enrollmentId, "GRADE_3");
        when(statusUpdater.markCompleted(aeId)).thenReturn(true);
        doThrow(new RuntimeException("ledger write failed"))
            .when(ledgerWriter).writeCompletionEntry(any(), any(), any(), any(), anyBoolean(), any());

        assertThatCode(() -> listener.onCaseLifecycle(goalReachedEvent(caseId)))
            .doesNotThrowAnyException();

        verify(ledgerWriter).writeObserverFailureEntry(eq(aeId), eq(enrollmentId), eq(CtcaeGrade.GRADE_3));
    }

    @Test
    void writeCompletionEntry_and_fallback_both_throw_does_not_propagate() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        mockInstanceWith(caseId, aeId, enrollmentId, "GRADE_4");
        when(statusUpdater.markCompleted(aeId)).thenReturn(true);
        doThrow(new RuntimeException("ledger write failed"))
            .when(ledgerWriter).writeCompletionEntry(any(), any(), any(), any(), anyBoolean(), any());
        doThrow(new RuntimeException("fallback write failed"))
            .when(ledgerWriter).writeObserverFailureEntry(any(), any(), any());

        assertThatCode(() -> listener.onCaseLifecycle(goalReachedEvent(caseId)))
            .doesNotThrowAnyException();
    }

    @Test
    void fireAsync_throws_after_ledger_written_no_failure_entry() {
        UUID caseId = UUID.randomUUID();
        UUID aeId = UUID.randomUUID();
        UUID enrollmentId = UUID.randomUUID();
        mockInstanceWith(caseId, aeId, enrollmentId, "GRADE_3");
        when(statusUpdater.markCompleted(aeId)).thenReturn(true);
        doThrow(new RuntimeException("fireAsync failed"))
            .when(completedEvents).fireAsync(any());

        assertThatCode(() -> listener.onCaseLifecycle(goalReachedEvent(caseId)))
            .doesNotThrowAnyException();

        verify(ledgerWriter, never()).writeObserverFailureEntry(any(), any(), any());
    }
```

- [ ] **Step 2: Run tests to verify new tests fail**

```bash
mvn test -pl runtime -Dtest=AeEscalationListenerTest --batch-mode
```
Expected: 3 existing tests pass; 3 new tests fail (`writeObserverFailureEntry` not called, `fireAsync` throw propagates, etc.).

- [ ] **Step 3: Implement the restructured listener**

Replace the entire `onCaseLifecycle` method body in `AeEscalationListener.java` with:

```java
    @Transactional
    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        LOG.debugf("AeEscalationListener: received eventType=%s caseStatus=%s caseId=%s", event.eventType(), event.caseStatus(), event.caseId());
        if (!"GoalReached".equals(event.eventType()) && !"CaseCompleted".equals(event.eventType())) return;

        var instance = caseInstanceRepository
                .findByUuid(event.caseId())
                .await().atMost(LOOKUP_TIMEOUT);
        if (instance == null) return;

        Object aeIdObj = instance.getCaseContext().getPath("aeId");
        if (aeIdObj == null) return; // not an AE escalation case

        UUID aeId;
        try {
            aeId = UUID.fromString(aeIdObj.toString());
        } catch (IllegalArgumentException e) {
            LOG.warnf("AeEscalationListener: invalid aeId in case context: %s", aeIdObj);
            return;
        }

        // REQUIRES_NEW: commits independently of the outer transaction.
        // Returns false if already COMPLETED — GoalReached fires multiple times per case (idempotency guard).
        boolean firstCompletion = statusUpdater.markCompleted(aeId);
        if (!firstCompletion) return;

        // Context resolution outside try block — if these throw, no REQUIRES_NEW has committed,
        // so there is no FDA gap. Exceptions propagate to the @ObservesAsync dispatcher, which logs them.
        UUID enrollmentId = resolveUuid(instance.getCaseContext().getPath("enrollmentId"));
        if (enrollmentId == null) {
            LOG.warnf("AeEscalationListener: enrollmentId missing from case context for aeId=%s — ledger write skipped", aeId);
            return;
        }
        UUID siteId = resolveUuid(instance.getCaseContext().getPath("siteId"));
        CtcaeGrade grade = resolveGrade(instance.getCaseContext().getPath("grade"));
        String safetyReviewOutcome = resolveOutcome(instance.getCaseContext().getPath("safetyReview"));
        boolean dsmbEscalated = instance.getCaseContext().getPath("dsmbEscalation") != null;
        Instant completedAt = Instant.now();

        // Narrow try/catch: markCompleted committed (REQUIRES_NEW). Any exception here is an FDA gap.
        // ledgerWritten guards against a spurious failure entry when only fireAsync throws after success.
        boolean ledgerWritten = false;
        try {
            ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, completedAt);
            ledgerWritten = true;
            completedEvents.fireAsync(new AeEscalationCompletedEvent(
                    aeId, grade, siteId, safetyReviewOutcome, dsmbEscalated, completedAt));
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

- [ ] **Step 4: Run all listener tests**

```bash
mvn test -pl runtime -Dtest=AeEscalationListenerTest --batch-mode
```
Expected: BUILD SUCCESS, 6 tests passed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/AeEscalationListener.java \
  runtime/src/test/java/io/casehub/clinical/service/AeEscalationListenerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-48): AeEscalationListener — narrow try block with ledgerWritten flag and observer fallback

Refs #48"
```

---

## Task 4: IrbApprovalLedgerWriter — add writeObserverFailureEntry (TDD)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/clinical/service/IrbApprovalLedgerWriterTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/IrbApprovalLedgerWriter.java`

- [ ] **Step 1: Write one failing test**

Add to `IrbApprovalLedgerWriterTest.java` (after existing test). Add imports if not present:
```java
import io.casehub.clinical.api.ClinicalActors;
import io.casehub.platform.api.identity.ActorType;
```

```java
    @Test
    void writeObserverFailureEntry_uses_system_actor_and_irb_committee_observer_failed_role() {
        Instant now = Instant.parse("2026-05-31T10:00:00Z");
        when(clock.instant()).thenReturn(now);
        when(ledgerEntryRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());

        IrbApproval approval = new IrbApproval();
        approval.id = UUID.randomUUID();
        approval.deviationId = UUID.randomUUID();
        approval.siteId = UUID.randomUUID();
        approval.committeeId = "irb-oncology";
        approval.decision = IrbDecision.APPROVED;

        writer.writeObserverFailureEntry(approval);

        ArgumentCaptor<IrbApprovalLedgerEntry> captor =
                ArgumentCaptor.forClass(IrbApprovalLedgerEntry.class);
        verify(ledgerEntryRepository).save(captor.capture());
        IrbApprovalLedgerEntry entry = captor.getValue();
        assertThat(entry.actorId).isEqualTo(ClinicalActors.CLINICAL_SERVICE);
        assertThat(entry.actorType).isEqualTo(ActorType.SYSTEM);
        assertThat(entry.actorRole).isEqualTo("IrbCommittee-observer-failed");
        assertThat(entry.irbApprovalId).isEqualTo(approval.id);
        assertThat(entry.deviationId).isEqualTo(approval.deviationId);
        assertThat(entry.irbDecision).isEqualTo("APPROVED");
        assertThat(entry.committeeId).isEqualTo("irb-oncology");
        assertThat(entry.decidedAt).isEqualTo(now);
        assertThat(entry.subjectId).isEqualTo(approval.id);
    }
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl runtime -Dtest=IrbApprovalLedgerWriterTest --batch-mode
```
Expected: 1 existing test passes; new test fails with NoSuchMethodError.

- [ ] **Step 3: Implement writeObserverFailureEntry**

Add `import io.casehub.clinical.api.ClinicalActors;` to `IrbApprovalLedgerWriter.java` (after existing imports).
Add `import jakarta.transaction.Transactional;` if not already present.

Add after `writeDecisionEntry` (before `nextSequenceNumber`):

```java
    /**
     * Called from IrbDecisionListener observer fallback path.
     * Commits in its own REQUIRES_NEW transaction so it persists even if the
     * outer transaction is in rollback-only state.
     * Uses ClinicalActors.CLINICAL_SERVICE — this records a system-level failure,
     * not the IRB committee's decision (which writeDecisionEntry records).
     * approval.decision is always non-null at call time (set as first statement in try block).
     * committeeId defensive null-guard handles corrupt data only.
     */
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void writeObserverFailureEntry(IrbApproval approval) {
        var entry = new IrbApprovalLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = approval.id;
        entry.sequenceNumber = nextSequenceNumber(approval.id);
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ClinicalActors.CLINICAL_SERVICE;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "IrbCommittee-observer-failed";
        var now = clock.instant();
        entry.occurredAt = now;
        entry.irbApprovalId = approval.id;
        entry.deviationId = approval.deviationId;
        entry.irbDecision = approval.decision != null ? approval.decision.name() : "UNKNOWN";
        entry.committeeId = approval.committeeId != null ? approval.committeeId : "unknown";
        entry.decidedAt = now;
        ledgerEntryRepository.save(entry);
    }
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -pl runtime -Dtest=IrbApprovalLedgerWriterTest --batch-mode
```
Expected: BUILD SUCCESS, 2 tests passed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/IrbApprovalLedgerWriter.java \
  runtime/src/test/java/io/casehub/clinical/service/IrbApprovalLedgerWriterTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-48): add IrbApprovalLedgerWriter.writeObserverFailureEntry with system actor

Refs #48"
```

---

## Task 5: IrbDecisionListener — ledgerDecisionWritten flag + try/catch (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/service/IrbDecisionListenerTest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/IrbDecisionListener.java`

- [ ] **Step 1: Create the test file**

Create `runtime/src/test/java/io/casehub/clinical/service/IrbDecisionListenerTest.java`:

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.IrbApprovalResolvedEvent;
import io.casehub.clinical.api.model.IrbDecision;
import io.casehub.clinical.entity.IrbApproval;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.workadapter.CallerRef;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

@QuarkusTest
class IrbDecisionListenerTest {

    @Inject IrbDecisionListener listener;
    @InjectMock IrbApprovalLedgerWriter ledgerWriter;
    @InjectMock ClinicalDeviationCaseHub caseHub;
    @InjectMock Event<IrbApprovalResolvedEvent> resolvedEvents;

    private UUID approvalId;
    private UUID deviationId;

    @BeforeEach
    @Transactional
    void setUp() {
        approvalId = UUID.randomUUID();
        deviationId = UUID.randomUUID();

        IrbApproval approval = new IrbApproval();
        approval.id = approvalId;
        approval.siteId = UUID.randomUUID();
        approval.deviationId = deviationId;
        approval.reviewType = "FULL";
        approval.committeeId = "irb-oncology";
        approval.decisionDeadline = Instant.now().plusSeconds(86400);
        // decision defaults to IrbDecision.PENDING from field initializer
        approval.persist();
    }

    // --- Helper factories ---

    private WorkItemLifecycleEvent completedEvent(String decisionJson) {
        WorkItem workItem = new WorkItem();
        workItem.id = UUID.randomUUID();
        workItem.status = WorkItemStatus.COMPLETED;
        workItem.payload = "{\"deviationId\":\"" + deviationId + "\"}";
        workItem.resolution = "{\"decision\":\"" + decisionJson + "\"}";
        return WorkItemLifecycleEvent.of("COMPLETED", workItem, "irb-test", null);
    }

    private WorkItemLifecycleEvent expiredEvent() {
        UUID caseId = UUID.randomUUID();
        WorkItem workItem = new WorkItem();
        workItem.id = UUID.randomUUID();
        workItem.status = WorkItemStatus.EXPIRED;
        workItem.payload = "{\"deviationId\":\"" + deviationId + "\"}";
        workItem.callerRef = CallerRef.encode(caseId, "irb-consultation");
        return WorkItemLifecycleEvent.of("EXPIRED", workItem, "irb-test", null);
    }

    private WorkItemLifecycleEvent nonIrbEvent() {
        WorkItem workItem = new WorkItem();
        workItem.id = UUID.randomUUID();
        workItem.status = WorkItemStatus.COMPLETED;
        workItem.payload = "{}"; // no deviationId
        return WorkItemLifecycleEvent.of("COMPLETED", workItem, "irb-test", null);
    }

    // --- Tests ---

    @Test
    void approved_workitem_calls_writeDecisionEntry() {
        listener.onWorkItemLifecycle(completedEvent("APPROVED"));

        ArgumentCaptor<IrbApproval> captor = ArgumentCaptor.forClass(IrbApproval.class);
        verify(ledgerWriter).writeDecisionEntry(captor.capture());
        assertThat(captor.getValue().id).isEqualTo(approvalId);
        assertThat(captor.getValue().decision).isEqualTo(IrbDecision.APPROVED);
    }

    @Test
    void expired_workitem_signals_case_and_calls_writeDecisionEntry() {
        listener.onWorkItemLifecycle(expiredEvent());

        verify(caseHub).signal(any(UUID.class), eq("irbConsultation"), any());
        verify(ledgerWriter).writeDecisionEntry(any(IrbApproval.class));
    }

    @Test
    void non_irb_workitem_skipped() {
        listener.onWorkItemLifecycle(nonIrbEvent());

        verifyNoInteractions(ledgerWriter);
        verifyNoInteractions(caseHub);
    }

    @Test
    void writeDecisionEntry_throws_calls_writeObserverFailureEntry() {
        doThrow(new RuntimeException("ledger write failed"))
            .when(ledgerWriter).writeDecisionEntry(any());

        assertThatCode(() -> listener.onWorkItemLifecycle(completedEvent("APPROVED")))
            .doesNotThrowAnyException();

        verify(ledgerWriter).writeObserverFailureEntry(any(IrbApproval.class));
    }

    @Test
    void writeDecisionEntry_and_fallback_both_throw_does_not_propagate() {
        doThrow(new RuntimeException("ledger write failed"))
            .when(ledgerWriter).writeDecisionEntry(any());
        doThrow(new RuntimeException("fallback write failed"))
            .when(ledgerWriter).writeObserverFailureEntry(any());

        assertThatCode(() -> listener.onWorkItemLifecycle(completedEvent("APPROVED")))
            .doesNotThrowAnyException();
    }

    @Test
    void fireAsync_throws_after_ledger_written_no_failure_entry() {
        doThrow(new RuntimeException("fireAsync failed"))
            .when(resolvedEvents).fireAsync(any());

        assertThatCode(() -> listener.onWorkItemLifecycle(completedEvent("APPROVED")))
            .doesNotThrowAnyException();

        verify(ledgerWriter, never()).writeObserverFailureEntry(any(IrbApproval.class));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail (method not yet changed)**

```bash
mvn test -pl runtime -Dtest=IrbDecisionListenerTest --batch-mode
```
Expected: Tests that verify `writeObserverFailureEntry` is called will fail (the current listener has no try/catch). Tests that verify no exception is propagated may also fail. `fireAsync_throws_after_ledger_written_no_failure_entry` may fail because `fireAsync` throw escapes the listener.

- [ ] **Step 3: Implement the restructured listener**

Replace the entire `onWorkItemLifecycle` method body in `IrbDecisionListener.java` with:

```java
    @Transactional
    public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
        if (!(event.source() instanceof WorkItem workItem)) return;

        UUID deviationId = extractDeviationId(workItem);
        if (deviationId == null) {
            LOG.tracef("WorkItem %s has no deviationId — not an IRB item, skipping", workItem.id);
            return;
        }

        IrbDecision decision = resolveDecision(event.status(), workItem);
        if (decision == null) {
            LOG.tracef("WorkItem %s status=%s is non-terminal for IRB processing — skipping", workItem.id, event.status());
            return;
        }

        IrbApproval approval = IrbApproval
                .find("deviationId = ?1 and decision = 'PENDING'", deviationId)
                .firstResult();
        if (approval == null) {
            LOG.warnf("No PENDING IrbApproval for deviationId=%s — already resolved?", deviationId);
            return;
        }

        // ledgerDecisionWritten guards against a spurious failure entry when only
        // fireAsync throws after writeDecisionEntry succeeds. Without it: exception caught
        // → outer TX commits → success ledger entry + REQUIRES_NEW failure entry both committed.
        boolean ledgerDecisionWritten = false;
        try {
            approval.decision = decision;
            approval.persist();

            if (event.status() == WorkItemStatus.EXPIRED) {
                CallerRef ref = CallerRef.parse(workItem.callerRef);
                if (ref != null) {
                    caseHub.signal(ref.caseId(), "irbConsultation", Map.of(
                            "decision", "EXPIRED",
                            "committeeId", approval.committeeId,
                            "decidedAt", Instant.now().toString()));
                }
            }

            ledgerWriter.writeDecisionEntry(approval);
            ledgerDecisionWritten = true;

            resolvedEvents.fireAsync(new IrbApprovalResolvedEvent(
                    approval.id, deviationId, approval.siteId, decision, Instant.now()));
        } catch (Exception e) {
            if (!ledgerDecisionWritten) {
                LOG.errorf(e, "IrbDecisionListener: unexpected error for deviationId=%s approvalId=%s — writing failure entry", deviationId, approval.id);
                try {
                    ledgerWriter.writeObserverFailureEntry(approval);
                } catch (Exception writeEx) {
                    LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for deviationId=%s", deviationId);
                }
            } else {
                LOG.errorf(e, "IrbDecisionListener: downstream fireAsync failed for deviationId=%s — ledger entry exists, no fallback needed", deviationId);
            }
        }
    }
```

- [ ] **Step 4: Run all IrbDecisionListener tests**

```bash
mvn test -pl runtime -Dtest=IrbDecisionListenerTest --batch-mode
```
Expected: BUILD SUCCESS, 6 tests passed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/clinical add \
  runtime/src/main/java/io/casehub/clinical/service/IrbDecisionListener.java \
  runtime/src/test/java/io/casehub/clinical/service/IrbDecisionListenerTest.java
git -C /Users/mdproctor/claude/casehub/clinical commit -m "feat(issue-48): IrbDecisionListener — ledgerDecisionWritten flag and observer fallback

Refs #48"
```

---

## Task 6: Full test suite + final verification

- [ ] **Step 1: Run full reactor test**

```bash
mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode
```
Expected: BUILD SUCCESS. All existing tests continue to pass alongside the 13 new tests.

- [ ] **Step 2: If failures — check for regressions**

If tests fail, run the specific failing class:
```bash
mvn test -pl runtime -Dtest=<FailingClass> --batch-mode
```
Fix the issue, then return to Step 1.

- [ ] **Step 3: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/clinical push -u origin issue-48-ae-irb-observer-fallback
```

---

## Self-review against spec

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| `ctcae_grade` nullable — entity annotation | Task 1 |
| V1012 migration | Task 1 |
| `AeEscalationLedgerWriter.writeObserverFailureEntry` + null-grade fix | Task 2 |
| `AeEscalationListener` context outside try + narrow try + `ledgerWritten` | Task 3 |
| `IrbApprovalLedgerWriter.writeObserverFailureEntry` + ClinicalActors import | Task 4 |
| `IrbDecisionListener` `ledgerDecisionWritten` + try/catch | Task 5 |
| AeEscalation writer tests: null-grade fix + 2 failure writer tests | Task 2 |
| AeEscalation listener tests: ledger throws, both throw, fireAsync-after-success | Task 3 |
| IRB writer test: system actor + IrbCommittee-observer-failed role | Task 4 |
| IRB listener tests: 6 QuarkusTest (approved, expired, skipped, throw, both-throw, fireAsync-after-success) | Task 5 |
| `actorRole = "IrbCommittee-observer-failed"` | Task 4 |
| `@Transactional(REQUIRES_NEW)` on both failure writers | Tasks 2, 4 |
| No ProtocolDeviation in IrbDecisionListenerTest setup | Task 5 |

All spec requirements covered. No gaps found.

**Placeholder scan:** No TBD, TODO, or vague steps present. All code blocks are complete.

**Type consistency check:**
- `ledgerWriter.writeObserverFailureEntry(aeId, enrollmentId, grade)` — matches Task 2 method signature ✅
- `ledgerWriter.writeObserverFailureEntry(approval)` — matches Task 4 method signature ✅
- `ledgerWriter.writeDecisionEntry(approval)` — matches existing method (unchanged) ✅
- `"AeEscalationCase-observer-failed"` — consistent across Task 2 impl and Task 2 test ✅
- `"IrbCommittee-observer-failed"` — consistent across Task 4 impl and Task 4 test ✅
