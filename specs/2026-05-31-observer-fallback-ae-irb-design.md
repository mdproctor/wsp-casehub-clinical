# Observer Fallback — AeEscalationListener + IrbDecisionListener

**Issue:** casehubio/clinical#48  
**Branch:** issue-48-ae-irb-observer-fallback  
**Date:** 2026-05-31  
**Refs:** #45 (SafetyOfficerNotificationListener + SponsorNotificationListener — established pattern)

---

## Problem

Two `@ObservesAsync @Transactional` listeners lack the double try/catch fallback required by protocol PP-20260530-49856c:

- **`AeEscalationListener`**: observes `CaseLifecycleEvent`. After `AeStatusUpdater.markCompleted(aeId)` commits (REQUIRES_NEW), any subsequent exception leaves escalation status = COMPLETED with no corresponding ledger entry — FDA gap.
- **`IrbDecisionListener`**: observes `WorkItemLifecycleEvent`. If `approval.persist()` or `ledgerWriter.writeDecisionEntry()` throws, the IRB decision is unrecorded in the audit trail — FDA gap.

The `@ObservesAsync` dispatcher silently swallows unhandled exceptions; without an explicit catch, the absence of a ledger entry is indistinguishable from "the event was never triggered."

---

## Design

### 1. Schema change

**`AeEscalationLedgerEntry.ctcaeGrade`**: change `@Column(name = "ctcae_grade", nullable = false)` → `@Column(name = "ctcae_grade")`.

Rationale: `resolveGrade()` returns null when grade is absent from case context or unparseable. The observer failure entry records what is known; null is semantically correct for "grade was not determinable at time of failure." The NOT NULL invariant for normal completion entries is enforced by application logic (grade is always present in a correctly-formed AE escalation case), not the schema.

**Migration** `V1012__alter_ae_escalation_grade_nullable.sql` (qhorus datasource):
```sql
ALTER TABLE ae_escalation_ledger_entry ALTER COLUMN ctcae_grade DROP NOT NULL;
```

Tests use `drop-and-create` from Hibernate annotations — the entity annotation change is sufficient for the test schema. No default-datasource migration needed.

---

### 2. New writer methods

#### `AeEscalationLedgerWriter.writeObserverFailureEntry(UUID aeId, UUID enrollmentId, CtcaeGrade grade)`

```
@Transactional(REQUIRES_NEW)
actorId    = ClinicalActors.CLINICAL_SERVICE
actorType  = SYSTEM
actorRole  = "AeEscalationCase-observer-failed"
entryType  = EVENT
ctcaeGrade = grade != null ? grade.name() : null
dsmbEscalated  = false
safetyReviewOutcome = null
completedAt = clock.instant()
```

Also fix `writeCompletionEntry`: `entry.ctcaeGrade = grade != null ? grade.name() : null` (removes latent NPE when grade is absent from case context).

#### `IrbApprovalLedgerWriter.writeObserverFailureEntry(IrbApproval approval)`

```
@Transactional(REQUIRES_NEW)
actorId    = ClinicalActors.CLINICAL_SERVICE   (not "irb-committee" — this is a system failure record)
actorType  = SYSTEM
actorRole  = "irb-decision-observer-failed"
entryType  = EVENT
irbApprovalId = approval.id
deviationId   = approval.deviationId
irbDecision   = approval.decision != null ? approval.decision.name() : "UNKNOWN"
committeeId   = approval.committeeId != null ? approval.committeeId : "unknown"
decidedAt     = clock.instant()
```

`IrbApprovalLedgerWriter` requires `import io.casehub.clinical.api.ClinicalActors` (currently absent).

The human-actor exemption in protocol PP-20260530-d6775a applies only to `writeDecisionEntry` (records the IRB committee as a named human actor). The fallback entry is a system event.

`approval.id` references an existing `irb_approval` row (approval was loaded from DB before the try block). No FK constraint in `V1009` on `irb_approval_id` → `irb_approval.id`, so no commit-time violation risk.

---

### 3. Listener restructures

#### `AeEscalationListener.onCaseLifecycle`

Pre-aeId phase (filter, instance lookup, aeId parse, `markCompleted`) is unchanged. After `markCompleted` returns true, the method body becomes:

```
UUID enrollmentId = null;
CtcaeGrade grade = null;
boolean ledgerWritten = false;
try {
    enrollmentId = resolveUuid(instance.getCaseContext().getPath("enrollmentId"));
    if (enrollmentId == null) { LOG.warnf("..."); return; }

    UUID siteId = resolveUuid(instance.getCaseContext().getPath("siteId"));
    grade       = resolveGrade(instance.getCaseContext().getPath("grade"));
    String safetyReviewOutcome = resolveOutcome(instance.getCaseContext().getPath("safetyReview"));
    boolean dsmbEscalated = instance.getCaseContext().getPath("dsmbEscalation") != null;
    Instant completedAt = Instant.now();

    ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade,
                                      safetyReviewOutcome, dsmbEscalated, completedAt);
    ledgerWritten = true;

    completedEvents.fireAsync(new AeEscalationCompletedEvent(
            aeId, grade, siteId, safetyReviewOutcome, dsmbEscalated, completedAt));

} catch (Exception e) {
    if (!ledgerWritten) {
        LOG.errorf(e, "AeEscalationListener: unexpected error for aeId=%s (enrollmentId=%s, grade=%s) ...", ...);
        if (enrollmentId == null) {
            LOG.errorf("AUDIT GAP: aeId=%s — enrollmentId not resolved, fallback cannot be written", aeId);
        } else {
            try { ledgerWriter.writeObserverFailureEntry(aeId, enrollmentId, grade); }
            catch (Exception writeEx) {
                LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for aeId=%s", aeId);
            }
        }
    } else {
        LOG.errorf(e, "AeEscalationListener: downstream fireAsync failed for aeId=%s — ledger exists, no fallback needed", aeId);
    }
}
```

`ledgerWritten` flag prevents a spurious failure entry when `fireAsync` throws after a successful ledger write. `enrollmentId == null` guard prevents writing a failure entry when the exception occurred before enrollmentId was resolved (AUDIT GAP log instead).

#### `IrbDecisionListener.onWorkItemLifecycle`

Approval load stays outside the try block. Post-load body becomes:

```
try {
    approval.decision = decision;
    approval.persist();

    if (event.status() == WorkItemStatus.EXPIRED) { ... caseHub.signal(...); }

    ledgerWriter.writeDecisionEntry(approval);

    resolvedEvents.fireAsync(new IrbApprovalResolvedEvent(...));

} catch (Exception e) {
    LOG.errorf(e, "IrbDecisionListener: unexpected error for deviationId=%s approvalId=%s ...", ...);
    try { ledgerWriter.writeObserverFailureEntry(approval); }
    catch (Exception writeEx) {
        LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for deviationId=%s", deviationId);
    }
}
```

No `ledgerWritten` flag needed. `resolvedEvents.fireAsync()` is CDI downstream signalling — a synchronous throw there is acceptable to include in the same fallback path. The approval entity fields are accessed by value (simple columns, no lazy associations) so the detached entity is safe to pass across the REQUIRES_NEW boundary.

---

### 4. Tests

#### `AeEscalationListenerTest` (Mockito, extend existing)

| Test | Scenario | Assert |
|------|----------|--------|
| `writeCompletionEntry_throws_writes_observer_failure_entry` | `markCompleted` returns true; `writeCompletionEntry` throws | `writeObserverFailureEntry(aeId, enrollmentId, grade)` called; listener doesn't throw |
| `writeCompletionEntry_and_fallback_both_throw_does_not_propagate` | Both throws | Listener swallows (AUDIT GAP logged) |
| `context_throws_before_enrollmentId_resolved_logs_audit_gap_only` | `instance.getCaseContext()` throws before enrollmentId set | `writeObserverFailureEntry` NOT called; listener doesn't throw |

#### `AeEscalationLedgerWriterTest` (Mockito, extend existing)

| Test | Assert |
|------|--------|
| `writeObserverFailureEntry_with_null_grade_saves_null_grade` | Saved entry has null ctcaeGrade |
| `writeObserverFailureEntry_with_valid_grade_saves_grade` | Saved entry has grade.name() |

#### `IrbDecisionListenerTest` (QuarkusTest, new file)

Setup: `@BeforeEach @Transactional` creates `ProtocolDeviation` + `IrbApproval` (PENDING) in DB.  
Mocks: `@InjectMock IrbApprovalLedgerWriter ledgerWriter`, `@InjectMock ClinicalDeviationCaseHub caseHub`, `@InjectMock Event<IrbApprovalResolvedEvent> resolvedEvents`.

| Test | Scenario | Assert |
|------|----------|--------|
| `approved_workitem_calls_writeDecisionEntry` | COMPLETED WorkItem with valid resolution | `writeDecisionEntry(approval)` called |
| `expired_workitem_signals_case_and_calls_writeDecisionEntry` | EXPIRED status | `caseHub.signal(...)` + `writeDecisionEntry` called |
| `non_irb_workitem_skipped` | No deviationId in payload | No interactions with ledgerWriter or caseHub |
| `writeDecisionEntry_throws_calls_writeObserverFailureEntry` | `doThrow` on `writeDecisionEntry` | `writeObserverFailureEntry(approval)` called; listener doesn't throw |
| `writeDecisionEntry_and_fallback_both_throw_does_not_propagate` | Both throw | Listener swallows |

#### `IrbApprovalLedgerWriterTest` (Mockito, extend existing)

| Test | Assert |
|------|--------|
| `writeObserverFailureEntry_uses_system_actor_and_observer_failed_role` | `actorId = ClinicalActors.CLINICAL_SERVICE`, `actorType = SYSTEM`, `actorRole = "irb-decision-observer-failed"` |

---

## Files changed

| File | Change |
|------|--------|
| `runtime/.../ledger/AeEscalationLedgerEntry.java` | `ctcae_grade` nullable |
| `runtime/.../service/AeEscalationLedgerWriter.java` | Add `writeObserverFailureEntry`; fix null grade in `writeCompletionEntry` |
| `runtime/.../service/AeEscalationListener.java` | Add try/catch with `ledgerWritten` flag |
| `runtime/.../service/IrbApprovalLedgerWriter.java` | Add `writeObserverFailureEntry`; add ClinicalActors import |
| `runtime/.../service/IrbDecisionListener.java` | Add try/catch around post-load body |
| `runtime/.../resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql` | New |
| `runtime/.../service/AeEscalationListenerTest.java` | 3 new tests |
| `runtime/.../service/AeEscalationLedgerWriterTest.java` | 2 new tests |
| `runtime/.../service/IrbDecisionListenerTest.java` | New file — 5 tests |
| `runtime/.../service/IrbApprovalLedgerWriterTest.java` | 1 new test |

---

## Protocols satisfied

- PP-20260530-49856c (`observer-ledger-fallback`): double try/catch + REQUIRES_NEW on both listeners ✅
- PP-20260530-d6775a (`clinical-actor-id-constant`): `ClinicalActors.CLINICAL_SERVICE` used in all new system-actor writes ✅
- PLATFORM.md coherence: application-layer changes only, no platform boundary violations ✅
