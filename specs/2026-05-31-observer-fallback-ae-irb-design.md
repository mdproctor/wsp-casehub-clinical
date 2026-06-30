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

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(UUID aeId, UUID enrollmentId, CtcaeGrade grade) {
    // actorId    = ClinicalActors.CLINICAL_SERVICE
    // actorType  = ActorType.SYSTEM
    // actorRole  = "AeEscalationCase-observer-failed"
    // entryType  = LedgerEntryType.EVENT
    // ctcaeGrade = grade != null ? grade.name() : null
    // dsmbEscalated       = false
    // safetyReviewOutcome = null
    // completedAt = clock.instant()
}
```

Also fix `writeCompletionEntry`: `entry.ctcaeGrade = grade != null ? grade.name() : null` (removes latent NPE when grade is absent from case context).

#### `IrbApprovalLedgerWriter.writeObserverFailureEntry(IrbApproval approval)`

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void writeObserverFailureEntry(IrbApproval approval) {
    // actorId    = ClinicalActors.CLINICAL_SERVICE  (system failure record, not the IRB committee)
    // actorType  = ActorType.SYSTEM
    // actorRole  = "IrbCommittee-observer-failed"   (extends "IrbCommittee" — convention: success-role + "-observer-failed")
    // entryType  = LedgerEntryType.EVENT
    // irbApprovalId = approval.id
    // deviationId   = approval.deviationId
    // irbDecision   = approval.decision.name()       (always non-null: resolved before try, assignment can't throw)
    // committeeId   = approval.committeeId != null ? approval.committeeId : "unknown"  (defensive)
    // decidedAt     = clock.instant()
}
```

`IrbApprovalLedgerWriter` requires `import io.casehub.clinical.api.ClinicalActors` (currently absent).

The human-actor exemption in protocol PP-20260530-d6775a applies only to `writeDecisionEntry` (records the IRB committee as a named human actor). The fallback entry is a system event.

`approval.id` references an existing `irb_approval` row (approval was loaded from DB before the try block). No FK constraint in `V1009` on `irb_approval_id` → `irb_approval.id`, so no commit-time violation risk.

**actorRole naming convention:** failure role = success role + `"-observer-failed"`. Matches `DeviationLedgerWriter` ("sponsor-notifier" → "sponsor-notifier-observer-failed") and makes FDA audit queries by actorRole consistent across all listener types.

| Writer | Success role | Failure role |
|--------|-------------|-------------|
| `AeEscalationLedgerWriter` | `"AeEscalationCase"` | `"AeEscalationCase-observer-failed"` |
| `IrbApprovalLedgerWriter` | `"IrbCommittee"` | `"IrbCommittee-observer-failed"` |
| `DeviationLedgerWriter` | `"sponsor-notifier"` | `"sponsor-notifier-observer-failed"` |

---

### 3. Listener restructures

#### `AeEscalationListener.onCaseLifecycle`

Pre-aeId phase (filter, instance lookup, aeId parse, `markCompleted`) is unchanged.

After `markCompleted` returns true, context resolution moves **outside** the try block (matching the IRB pattern). This eliminates the irresolvable AUDIT GAP: if context resolution throws, `enrollmentId` is guaranteed non-null in the catch (since the `null` case returns early), and the try block is narrow — covering only operations where a fallback entry is warranted.

```
// Context resolution — outside try; early-return logic unchanged
UUID enrollmentId = resolveUuid(instance.getCaseContext().getPath("enrollmentId"));
if (enrollmentId == null) { LOG.warnf("..."); return; }
UUID siteId            = resolveUuid(instance.getCaseContext().getPath("siteId"));
CtcaeGrade grade       = resolveGrade(instance.getCaseContext().getPath("grade"));
String safetyReviewOutcome = resolveOutcome(instance.getCaseContext().getPath("safetyReview"));
boolean dsmbEscalated  = instance.getCaseContext().getPath("dsmbEscalation") != null;
Instant completedAt    = Instant.now();

// Try block: ledger write + downstream event only
boolean ledgerWritten = false;
try {
    ledgerWriter.writeCompletionEntry(aeId, enrollmentId, grade, safetyReviewOutcome, dsmbEscalated, completedAt);
    ledgerWritten = true;
    completedEvents.fireAsync(new AeEscalationCompletedEvent(
            aeId, grade, siteId, safetyReviewOutcome, dsmbEscalated, completedAt));
} catch (Exception e) {
    if (!ledgerWritten) {
        LOG.errorf(e, "AeEscalationListener: unexpected error for aeId=%s (enrollmentId=%s, grade=%s) — writing failure entry", ...);
        try { ledgerWriter.writeObserverFailureEntry(aeId, enrollmentId, grade); }
        catch (Exception writeEx) {
            LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for aeId=%s", aeId);
        }
    } else {
        LOG.errorf(e, "AeEscalationListener: downstream fireAsync failed for aeId=%s — ledger exists, no fallback needed", aeId);
    }
}
```

`ledgerWritten` prevents a spurious failure entry when `fireAsync` throws after the ledger write succeeds. Without it: exception caught → outer TX commits → both the success entry and the REQUIRES_NEW failure entry are committed (double-recording). The `enrollmentId == null` guard is no longer needed in the catch (context resolution failures propagate to the `@ObservesAsync` dispatcher, which logs them).

#### `IrbDecisionListener.onWorkItemLifecycle`

Approval load stays outside the try block. Post-load body:

```
boolean ledgerDecisionWritten = false;
try {
    approval.decision = decision;
    approval.persist();

    if (event.status() == WorkItemStatus.EXPIRED) { ... caseHub.signal(...); }

    ledgerWriter.writeDecisionEntry(approval);
    ledgerDecisionWritten = true;

    resolvedEvents.fireAsync(new IrbApprovalResolvedEvent(...));

} catch (Exception e) {
    if (!ledgerDecisionWritten) {
        LOG.errorf(e, "IrbDecisionListener: unexpected error for deviationId=%s approvalId=%s — writing failure entry", ...);
        try { ledgerWriter.writeObserverFailureEntry(approval); }
        catch (Exception writeEx) {
            LOG.errorf(writeEx, "AUDIT GAP: could not write observer failure entry for deviationId=%s", deviationId);
        }
    } else {
        LOG.errorf(e, "IrbDecisionListener: downstream fireAsync failed for deviationId=%s — ledger exists, no fallback needed", deviationId);
    }
}
```

`ledgerDecisionWritten` is required for the same reason as `ledgerWritten` in `AeEscalationListener`: if `resolvedEvents.fireAsync()` throws synchronously after `writeDecisionEntry` succeeds, the exception is caught, the outer TX commits (`approval.persist()` + normal ledger entry both commit), and without the flag, a spurious REQUIRES_NEW failure entry would also be committed — three rows recording a single event, two contradicting each other.

The approval entity fields accessed in `writeObserverFailureEntry` (`id`, `deviationId`, `committeeId`, `decision`) are simple columns with no lazy associations, safe to access on the detached entity across the REQUIRES_NEW boundary.

---

### 4. Tests

#### `AeEscalationListenerTest` (Mockito, extend existing)

| Test | Scenario | Assert |
|------|----------|--------|
| `writeCompletionEntry_throws_writes_observer_failure_entry` | `markCompleted` returns true; `writeCompletionEntry` throws | `writeObserverFailureEntry(aeId, enrollmentId, grade)` called; listener doesn't throw |
| `writeCompletionEntry_and_fallback_both_throw_does_not_propagate` | Both throw | Listener swallows (AUDIT GAP logged) |
| `fireAsync_throws_after_ledger_written_no_failure_entry` | `writeCompletionEntry` succeeds (`ledgerWritten = true`); `fireAsync` throws | `writeObserverFailureEntry` NOT called; listener doesn't throw |

#### `AeEscalationLedgerWriterTest` (Mockito, extend existing)

| Test | Assert |
|------|--------|
| `writeCompletionEntry_with_null_grade_stores_null` | Null grade stored without NPE (covers the null-grade fix) |
| `writeObserverFailureEntry_with_null_grade_saves_null_grade` | Failure entry has null ctcaeGrade |
| `writeObserverFailureEntry_with_valid_grade_saves_grade` | Failure entry has grade.name() |

#### `IrbDecisionListenerTest` (QuarkusTest, new file)

Setup: `@BeforeEach @Transactional` creates `IrbApproval` (PENDING) only — `ProtocolDeviation` not needed (listener never loads it; no FK enforced in drop-and-create schema).  
Mocks: `@InjectMock IrbApprovalLedgerWriter ledgerWriter`, `@InjectMock ClinicalDeviationCaseHub caseHub`, `@InjectMock Event<IrbApprovalResolvedEvent> resolvedEvents`.

| Test | Scenario | Assert |
|------|----------|--------|
| `approved_workitem_calls_writeDecisionEntry` | COMPLETED WorkItem with valid resolution | `writeDecisionEntry(approval)` called |
| `expired_workitem_signals_case_and_calls_writeDecisionEntry` | EXPIRED status | `caseHub.signal(...)` + `writeDecisionEntry` called |
| `non_irb_workitem_skipped` | No deviationId in payload | No interactions with ledgerWriter or caseHub |
| `writeDecisionEntry_throws_calls_writeObserverFailureEntry` | `doThrow` on `writeDecisionEntry` | `writeObserverFailureEntry(approval)` called; listener doesn't throw |
| `writeDecisionEntry_and_fallback_both_throw_does_not_propagate` | Both throw | Listener swallows |
| `fireAsync_throws_after_ledger_written_no_failure_entry` | `writeDecisionEntry` succeeds; `resolvedEvents.fireAsync` throws | `writeObserverFailureEntry` NOT called; listener doesn't throw |

#### `IrbApprovalLedgerWriterTest` (Mockito, extend existing)

| Test | Assert |
|------|--------|
| `writeObserverFailureEntry_uses_system_actor_and_irb_committee_observer_failed_role` | `actorId = ClinicalActors.CLINICAL_SERVICE`, `actorType = SYSTEM`, `actorRole = "IrbCommittee-observer-failed"` |

---

## Files changed

| File | Change |
|------|--------|
| `runtime/.../ledger/AeEscalationLedgerEntry.java` | `ctcae_grade` nullable |
| `runtime/.../service/AeEscalationLedgerWriter.java` | Add `writeObserverFailureEntry`; fix null grade in `writeCompletionEntry` |
| `runtime/.../service/AeEscalationListener.java` | Context resolution outside try; narrow try block with `ledgerWritten` flag |
| `runtime/.../service/IrbApprovalLedgerWriter.java` | Add `writeObserverFailureEntry`; add ClinicalActors import |
| `runtime/.../service/IrbDecisionListener.java` | Add `ledgerDecisionWritten` flag; try/catch around post-load body |
| `runtime/.../resources/db/migration/qhorus/V1012__alter_ae_escalation_grade_nullable.sql` | New |
| `runtime/.../service/AeEscalationListenerTest.java` | 3 new tests (replaces original 3: adds fireAsync test, drops AUDIT GAP test) |
| `runtime/.../service/AeEscalationLedgerWriterTest.java` | 3 new tests (null-grade fix + 2 failure writer) |
| `runtime/.../service/IrbDecisionListenerTest.java` | New file — 6 tests |
| `runtime/.../service/IrbApprovalLedgerWriterTest.java` | 1 new test |

---

## Protocols satisfied

- PP-20260530-49856c (`observer-ledger-fallback`): double try/catch + REQUIRES_NEW on both listeners ✅
- PP-20260530-d6775a (`clinical-actor-id-constant`): `ClinicalActors.CLINICAL_SERVICE` used in all new system-actor writes ✅
- PLATFORM.md coherence: application-layer changes only, no platform boundary violations ✅